---
tags: [operacion, dax, power-bi, sumx, related, distinctcount, sprint-12]
tipo: nota-operacion
herramientas: [power-bi, dax]
---

# 📐 DAX Avanzado — SUMX, RELATED y Medidas Financieras

Medidas DAX para calcular costos cruzando tablas relacionadas, construir KPIs financieros compuestos y contar transacciones únicas. Complementa [[DAX_Modelado_PowerBI]] con patrones nuevos del S12.

---

## 📋 Índice de Operaciones

| Operación | Ir a |
|---|---|
| Costo total con SUMX + RELATED | [[#sumx-related]] |
| Profit total (medida compuesta) | [[#profit]] |
| ROI con DIVIDE | [[#roi]] |
| Conteo de transacciones con DISTINCTCOUNT | [[#distinctcount]] |
| Ticket promedio compuesto | [[#ticket]] |
| Cantidad promedio por orden | [[#cantidad-promedio]] |
| Gasto total de marketing | [[#gasto]] |

---

## ⚙️ Costo Total con SUMX + RELATED {#sumx-related}

**Cuándo:** Cuando el costo unitario de cada producto vive en una tabla de dimensión (`catalog`) y necesitas multiplicarlo por la cantidad vendida en la tabla de hechos (`orders`), fila por fila.

```dax
Costo total = 
SUMX(
    orders_clean,
    orders_clean[cantidad] * RELATED(catalog_clean[costo_unitario])
)
```

**Anatomía del patrón:**

| Función | Qué hace |
|---|---|
| `SUMX(tabla, expresión)` | Recorre la tabla fila por fila, evalúa la expresión en cada fila y suma los resultados |
| `RELATED(tabla[columna])` | Trae el valor de una columna de una tabla relacionada (equivale a un VLOOKUP automático) |
| `orders_clean[cantidad]` | Cantidad vendida en esa fila específica |
| `catalog_clean[costo_unitario]` | Costo del producto según el catálogo, traído vía la relación del modelo |

> [!IMPORTANT] SUMX vs SUM — cuándo usar cada uno
> - `SUM(tabla[columna])` → suma una columna que YA existe en la tabla
> - `SUMX(tabla, expresión)` → calcula algo fila por fila y luego suma. Obligatorio cuando necesitas multiplicar columnas de **tablas diferentes**

> [!NOTE] Equivalencia con Python
> Este patrón en DAX es el equivalente de:
> ```python
> orders['costo_total'] = orders['cantidad'] * orders['costo_unitario']  # tras el merge
> costo_total = orders['costo_total'].sum()
> ```
> La diferencia: en DAX no necesitas hacer el merge explícitamente — la relación del modelo estrella lo hace automáticamente vía `RELATED`.

**Contexto real:** S12 — `catalog_clean` tenía `costo_unitario` y `orders_clean` tenía `cantidad`. Sin `SUMX + RELATED`, no había forma de calcular el costo total sin hacer un merge previo en Power Query.

---

## 💰 Profit Total (Medida Compuesta) {#profit}

**Cuándo:** Para calcular la utilidad neta del negocio en el dashboard, reutilizando medidas base ya creadas.

```dax
Profit total = [Revenue total] - [Costo total]
```

**Fórmula de negocio:**
$$\text{Profit} = \text{Revenue} - \text{COGS (Costo de productos)}$$

> [!TIP] Siempre construir medidas compuestas sobre medidas base
> En lugar de `SUM(monto_total) - SUMX(...)` en una sola medida, crea primero `Revenue total` y `Costo total` por separado. Esto hace que el código sea más legible, fácil de depurar y que las medidas sean reutilizables en otros contextos.

**Contexto real:** S12 — KPI central del Overview Ejecutivo del dashboard de RappiPlus.

---

## 📈 ROI con DIVIDE {#roi}

**Cuándo:** Para medir la eficiencia de la inversión en marketing: cuánto profit se genera por cada unidad monetaria gastada.

```dax
ROI % = 
DIVIDE(
    [Profit total],
    [Gasto total marketing],
    0
)
```

$$\text{ROI \%} = \frac{\text{Profit}}{\text{Gasto en Marketing}}$$

**Parámetros de DIVIDE:**
- Primer argumento → numerador (`Profit total`)
- Segundo argumento → denominador (`Gasto total marketing`)
- Tercer argumento → valor alternativo si el denominador es 0 (devuelve `0` en lugar de error)

> [!IMPORTANT] Formato obligatorio
> Seleccionar la medida `ROI %` → pestaña **Herramientas de medición** → botón `%` para que se muestre como porcentaje en los visuales.

**Contexto real:** S12 — KPI de eficiencia del dashboard. Permite comparar el ROI por categoría de producto o por mes.

---

## 🔢 Conteo de Transacciones con DISTINCTCOUNT {#distinctcount}

**Cuándo:** Cuando la tabla de hechos no tiene una columna de ID de orden único, o cuando quieres contar el número de pedidos distintos (no de líneas de producto).

```dax
-- Si existe columna de ID de orden
Cantidad total ordenes = DISTINCTCOUNT(orders_clean[id_pedido])

-- Si cada fila es una línea de venta (sin ID de orden)
Cantidad total ventas = COUNTROWS(orders_clean)
```

**Diferencia clave:**

| Función | Cuándo usar |
|---|---|
| `DISTINCTCOUNT(col)` | Cuando hay duplicados y quieres contar valores únicos (ej. pedidos únicos) |
| `COUNTROWS(tabla)` | Cuando cada fila es una observación única (ej. líneas de producto) |
| `COUNT(col)` | Cuenta filas no nulas de una columna específica |

> [!WARNING] Sin ID de orden — usar COUNTROWS
> En el S12, `orders_clean` no tenía `id_pedido`. Usar `DISTINCTCOUNT` sobre `id_usuario` hubiera dado el número de clientes únicos, no de transacciones. `COUNTROWS` es la opción correcta cuando cada fila representa una venta.

**Contexto real:** S12 — `orders_clean` tenía líneas de producto sin ID de orden. Se usó `COUNTROWS` para la métrica "Cantidad total ventas".

---

## 🎫 Ticket Promedio Compuesto {#ticket}

**Cuándo:** Para calcular el valor promedio por transacción, reutilizando las medidas de Revenue y cantidad ya creadas.

```dax
Ticket promedio = 
DIVIDE(
    [Revenue total],
    [Cantidad total ventas],
    0
)
```

> [!WARNING] No usar AVERAGE directamente
> `AVERAGE(orders_clean[precio_unitario])` calcula el precio promedio por **línea de producto**, no el valor promedio por **transacción**. Si una orden tiene 10 productos baratos, el promedio sale artificialmente bajo. La forma correcta es dividir el Revenue total entre el número de transacciones.

**Contexto real:** S12 — KPI de ticket promedio del Overview. Permite identificar si el valor por transacción está creciendo o cayendo con el tiempo.

---

## 📦 Cantidad Promedio por Orden {#cantidad-promedio}

**Cuándo:** Para medir el UPT (Units Per Transaction) — cuántas unidades lleva en promedio cada cliente por compra.

```dax
Cantidad promedio productos = 
DIVIDE(
    SUM(orders_clean[cantidad]),
    [Cantidad total ventas],
    0
)
```

**Contexto real:** S12 — Métrica de comportamiento de compra del dashboard de RappiPlus.

---

## 📣 Gasto Total de Marketing {#gasto}

**Cuándo:** Medida base para calcular el ROI y analizar la inversión por canal o período.

```dax
Gasto total marketing = SUM(marketing_clean[gasto])
```

> [!NOTE] Verificar el nombre de columna
> El nombre de la columna de gasto varía por proyecto (`gasto`, `monto_inversion`, `spend`). Verificar con `marketing_clean.columns` antes de crear la medida.

**Contexto real:** S12 — Base para el cálculo del ROI y la tarjeta KPI de "Gasto total marketing" en el Overview.

---

## 🔗 Conexiones Estratégicas

- **Índice Maestro:** [[Indice_Maestro]]
- **Herramienta:** [[Power_BI]]
- **Base de este patrón:** [[DAX_Modelado_PowerBI]]
- **Visualizaciones relacionadas:** [[DAX_Visualizaciones_PowerBI]]
- **Equivalente en Python:** [[KPIs_Financieros_Python]]
- **Traductor SQL/Pandas ↔ DAX (SUMX, RELATED, DIVIDE):** [[SQL_vs_Python_Traductor#sql-vs-dax]]
- **Sprint de referencia:** S12 — Proyecto Final RappiPlus
