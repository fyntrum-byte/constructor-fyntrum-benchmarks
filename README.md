# Constructor de Benchmarks SCVS — Fyntrum

Herramienta interna que convierte los datos abiertos de la Superintendencia de Compañías, Valores y Seguros del Ecuador en rangos sectoriales reales para el **Corporate Financial Analyzer (CFA)**.

Reemplaza los benchmarks estimados del CFA (`const BM`, escritos a mano) por percentiles calculados sobre las empresas ecuatorianas que efectivamente presentaron balances.

**El resultado en una frase:** el informe deja de decir *"tu margen está en el rango del sector"* y pasa a decir *"tu ROE de 12,4% te ubica en el percentil 61 de las 812 compañías de tu actividad y tu tamaño en Ecuador"*.

---

## Cómo se usa

Se corre **una vez al año**, en mayo, cuando la SCVS cierra la recepción de balances del ejercicio anterior (plazo legal: 30 de abril, art. 20 de la Ley de Compañías).

1. Descargar de `appscvsmovil.supercias.gob.ec/ranking/reporte.html`:
   - `bi_ranking.csv` (~373 MB)
   - `bi_ciiu.csv`
2. Abrir `constructor.html` en el navegador. No requiere servidor ni instalación.
3. Arrastrar los dos archivos.
4. Ajustar las reglas si hace falta (los valores por defecto son los recomendados).
5. **Construir benchmarks** → descargar `bm_real.js`.
6. Pegar el contenido en `index.html` del CFA.

Tiempo real de trabajo manual: unos 20 minutos.

> **No abrir los CSV con Excel.** Excel borra los ceros a la izquierda de los códigos CIIU, convierte números largos a notación científica y trunca en 1.048.576 filas — `bi_ranking.csv` tiene ~1,67 millones. Si se guarda desde Excel, el archivo queda destruido.

`bi_compania.csv` y `bi_segmento.csv` no hacen falta: los segmentos están fijos en el código y la agregación por actividad × tamaño no usa el primero. Guardar `bi_compania.csv` para cuando se agregue la dimensión provincia.

---

## Metodología

### Los ratios se recalculan; no se usan los de la fuente

La SCVS publica ratios ya calculados. **Varios están mal.** Verificado contra las cifras absolutas del mismo archivo, ejercicio 2024:

| Columna | Problema |
|---|---|
| `margen_operacional` | Da `0.00` para una empresa con 10,5% de rentabilidad operativa real. En otras da `200.94` o `-45.10`. |
| `cobertura_interes` | Negativa en empresas claramente rentables. |
| `per_med_cobranza` / `per_med_pago` | Mediana de 18.564 "días". Máximo de 65 millones. |
| `roe`, `roa`, `rent_neta_ventas` | **Esconden las pérdidas.** Una empresa con utilidad antes de impuestos de −$15.978.717 aparece con `roe = 0.01`, `roa = 0.01`, `rent_neta_ventas = 0.04`. Positivos. |

Las **cifras absolutas sí son confiables** (`utilidad_neta` sí registra los negativos) y cuadran al recalcular: margen bruto calculado 0.2652 contra 0.26 publicado — solo redondean a 2 decimales.

Por eso el constructor ignora las columnas de ratios y recalcula todo desde las cifras crudas **con las mismas fórmulas que usa el CFA**. Eso además resuelve un problema metodológico: comparar el `rentOp` de un cliente (fórmula del CFA) contra el `margen_operacional` de la SCVS (otra fórmula, además rota) sería comparar peras con manzanas. Ahora el cliente y el benchmark pasan por la misma fórmula.

### Ratios producidos

Calculados desde cifras absolutas:

| Ratio | Fórmula |
|---|---|
| `mcPct` | `(ingresos_ventas − costos_ventas_prod) / ingresos_ventas` |
| `rentOp` | `UAII / ingresos_ventas` |
| `rentNeta` | `utilidad_neta / ingresos_ventas` |
| `gao` | `MC / UAII` |
| `gaf` | `UAII / utilidad_an_imp` |
| `gat` | `gao × gaf` |
| `covInt` | `UAII / gastos_financieros` |
| `ende` | `(activos − patrimonio) / activos` |
| `roa` | `utilidad_neta / activos` |
| `roe` | `utilidad_neta / patrimonio` |
| `mgEbitda` | `(UAII + depreciaciones + amortizaciones) / ingresos_ventas` |
| `ventasEmp` | `ingresos_ventas / n_empleados` |

Donde `MC = ventas − costos_ventas_prod` y `UAII = MC − gastos_admin_ventas`.

Tomados directamente de la fuente, porque no publica activo ni pasivo corriente y no se pueden recalcular:

- `rc` ← `liquidez_corriente`
- `pAcida` ← `prueba_acida`

**No disponible:** múltiplos EV/EBITDA. Requieren valor de mercado y estas empresas no cotizan. Siguen viniendo del `const BM` estimado del CFA.

### Reglas de limpieza

El registro societario ecuatoriano tiene decenas de miles de compañías inscritas que no operan. Sin filtrar, los rangos salen deformados.

| Regla | Por defecto | Motivo |
|---|---|---|
| Ejercicio | 2025 | Más reciente y completo (151.778 compañías). 2024 como respaldo. Anteriores a ~2015 vienen incompletos. |
| Piso de ventas | $10.000 | Descarta cascarones y holdings inactivos. |
| Patrimonio > 0 | sí | El patrimonio negativo revienta el ROE con valores absurdos. |
| Activos > 0 | sí | Balance inconsistente. |
| Segmento válido | 1–4 | Se descarta el 0 (NO DEFINIDO). |
| Winsorización | p1–p99 | Una sola empresa con datos absurdos mueve los percentiles. |
| Muestra mínima | 30 | Un grupo con menos empresas no se publica. |

### Agregación

Cada empresa se acumula simultáneamente en todos los niveles del árbol CIIU más el agregado nacional. Se publican solo los grupos que llegan a la muestra mínima, y el CFA resuelve en tiempo de consulta bajando de lo específico a lo general:

```
clase (G4711) → grupo (G471) → división (G47) → sección (G) → TODOS
```

Los niveles salen de cortar `ciiu_n6`, que llega como `G4711.01`.

---

## Formato de salida

```js
const BM_REAL = {
  meta: { fuente, ejercicio, generado, empresas_validas, reglas, metodologia, ... },
  ciiu: { "G471": "VENTA AL POR MENOR EN COMERCIOS NO ESPECIALIZADOS" },
  g: {
    "G471|3": {
      n: 812,
      r: { roe: [0.021, 0.058, 0.094, 0.187, 0.341, 780], ... }
    }
  }
};
```

- Clave: `CODIGO_CIIU|SEGMENTO`
- Valores: `[p10, p25, p50, p75, p90, n]` — el último es cuántas empresas tenían ese ratio calculable
- Segmentos: `1` Microempresa · `2` Pequeña · `3` Mediana · `4` Grande

Peso según el detalle elegido (medido sobre una simulación pesimista de 150.000 empresas):

| Detalle | Grupos | Peso |
|---|---|---|
| Clase | ~3.700 | ~2,3 MB |
| Grupo *(por defecto)* | ~1.650 | ~1,0 MB |
| División | ~540 | ~0,3 MB |

Va embebido en `index.html`. Sin backend, sin servidor, sin costo mensual.

---

## Limitaciones conocidas

Hay que conocerlas para no vender más precisión de la que hay.

1. **La data está optimizada para el SRI.** Las PYMEs ecuatorianas subdeclaran utilidad, así que los benchmarks de rentabilidad sesgan hacia abajo. La comparación sigue siendo válida porque el sesgo es sistemático, no aleatorio — todos subdeclaran — pero no se debe presentar un percentil como precisión de laboratorio.
2. **El CIIU declarado no siempre es el que se opera.** Ruido inevitable; se mitiga con muestras grandes.
3. **`rc` y `pAcida` vienen de la fuente**, no recalculados. Se ven razonables (mediana 1,35) pero heredan lo que sea que la SCVS haya hecho.
4. **`mcPct` es un proxy.** Margen bruto usa costo de ventas; el margen de contribución del CFA usa costo variable. Parecidos, no idénticos.
5. **Sin filtro de estado.** La fuente no publica si la compañía está activa o en liquidación. Se aproxima con: presentó balances ese ejercicio + ventas sobre el piso.
6. **Un ~1‰ de filas viene corrupto** por comas mal escapadas. Se descartan.

---

## Fuente y atribución

Superintendencia de Compañías, Valores y Seguros del Ecuador — Ranking Empresarial.
Datos públicos publicados bajo el art. 20 de la Ley de Compañías. La SCVS pide que se cite la fuente; el informe del CFA debe indicar **fuente y ejercicio**.

Citar la fuente no es un trámite: es parte del argumento de venta.
