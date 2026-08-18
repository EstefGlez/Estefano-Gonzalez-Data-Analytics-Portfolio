---
tags: [herramienta, sql, python, pandas, numpy, traductor, indice]
tipo: nota-traductor
herramientas: [sql, python, pandas, numpy]
---

# 🔄 SQL vs Python — Traductor de Conceptos

Nota de referencia rápida: mismo concepto, distinta sintaxis. Pensada para cuando ya entiendes la lógica pero se te cruza el "¿cómo era esto en el otro lenguaje?". Organizada por librería — cada sección es independiente, consulta solo la que necesites.

---

## 📋 Índice

| Sección | Ir a |
|---|---|
| SQL vs Pandas — Operaciones base | [[#sql-vs-pandas-base]] |
| SQL vs Pandas — Funciones de ventana | [[#sql-vs-pandas-ventana]] |
| SQL vs Pandas — Cohortes (caso aplicado) | [[#sql-vs-pandas-cohortes]] |
| SQL vs NumPy — Operaciones numéricas | [[#sql-vs-numpy]] |
| Conceptos que se confunden seguido | [[#confusiones-comunes]] |

---

## 🐼 SQL vs Pandas — Operaciones Base {#sql-vs-pandas-base}

| Qué hace | SQL | Pandas |
|---|---|---|
| Seleccionar columnas | `SELECT col1, col2 FROM tabla` | `df[['col1', 'col2']]` |
| Filtrar filas | `WHERE condicion` | `df[df['col'] > valor]` |
| Ordenar | `ORDER BY col DESC` | `df.sort_values('col', ascending=False)` |
| Limitar filas | `LIMIT 10` | `df.head(10)` |
| Quitar duplicados | `SELECT DISTINCT col` | `df['col'].unique()` / `df.drop_duplicates()` |
| Agrupar y agregar (colapsa filas) | `GROUP BY col` + `SUM()`/`COUNT()`/`AVG()` | `df.groupby('col')['col2'].sum()` |
| Unir tablas | `JOIN` / `LEFT JOIN` | `pd.merge(df1, df2, how='left', on='col')` |
| Apilar tablas (mismo esquema) | `UNION ALL` | `pd.concat([df1, df2])` |
| Contar valores únicos por grupo | `COUNT(DISTINCT col)` | `.groupby('col2')['col'].nunique()` |
| Truncar fecha a mes | `DATE_TRUNC('month', fecha)` | `fecha.dt.to_period('M')` |
| Extraer parte de una fecha | `DATE_PART('year', fecha)` | `fecha.dt.year` |
| Traer valor de la fila anterior | `LAG(col) OVER (ORDER BY ...)` | `df['col'].shift(1)` |
| Traer valor de la fila siguiente | `LEAD(col) OVER (ORDER BY ...)` | `df['col'].shift(-1)` |
| Condicional | `CASE WHEN ... THEN ... ELSE ... END` | `np.where(condicion, valor_si, valor_no)` |
| Evitar división entre cero | `NULLIF(denominador, 0)` | `.replace(0, np.nan)` antes de dividir |

> [!NOTE] CTE ≈ variable intermedia
> Un `WITH cte AS (...)` en SQL es conceptualmente lo mismo que ir guardando resultados intermedios en variables de pandas paso a paso (`df1 = ...`, `df2 = ...`). Ambos existen para dividir un problema complejo en pasos legibles, no por eficiencia.

---

## 🪟 SQL vs Pandas — Funciones de Ventana {#sql-vs-pandas-ventana}

**Concepto central: ninguna de las dos "colapsa" filas** — a diferencia de un `GROUP BY`/`.agg()` normal, que sí reduce el número de filas, las funciones de ventana devuelven el mismo número de filas con el agregado "pegado" en cada una.

| Qué hace | SQL | Pandas |
|---|---|---|
| Agregado global (todas las filas) | `SUM(col) OVER ()` | `df['col'].transform('sum')` |
| Agregado por grupo (sin colapsar) | `SUM(col) OVER (PARTITION BY grupo)` | `df.groupby('grupo')['col'].transform('sum')` |
| Ranking dentro de cada grupo | `RANK() OVER (PARTITION BY grupo ORDER BY col DESC)` | `df.groupby('grupo')['col'].rank(ascending=False)` |
| Número de fila secuencial por grupo | `ROW_NUMBER() OVER (PARTITION BY grupo ORDER BY col)` | `df.groupby('grupo').cumcount() + 1` |

### La diferencia real entre `.agg()` y `.transform()`

| | `.agg()` / `GROUP BY` normal | `.transform()` / `OVER(...)` |
|---|---|---|
| ¿Colapsa filas? | Sí — una fila por grupo | No — mismo número de filas que el original |
| ¿Cuándo usarlo? | Cuando solo necesitas el resumen (ej. "total por categoría") | Cuando necesitas comparar cada fila individual contra el agregado de su grupo, en la misma fila |
| Ejemplo | `df.groupby('cat')['x'].sum()` → 4 filas si hay 4 categorías | `df.groupby('cat')['x'].transform('sum')` → mismas filas que el original, valor repetido por grupo |

> [!IMPORTANT] Regla práctica
> Usa función de ventana (`OVER`/`transform`) cuando el siguiente paso de tu análisis necesita **el detalle de cada fila Y el agregado de su grupo al mismo tiempo** (ej. calcular cuánto se desvía cada fila del promedio de su grupo). Si solo necesitas el resumen final, `GROUP BY`/`.agg()` normal es más simple y eficiente.

---

## 🔁 SQL vs Pandas — Cohortes (Caso Aplicado) {#sql-vs-pandas-cohortes}

Ejemplo completo de cómo se ve el mismo problema (retención por cohorte mensual) resuelto en ambos lenguajes — útil como plantilla mental para cualquier problema de "comparar cada evento contra el origen de su grupo".

**SQL:**
```sql
with cohorte_base as (
    select
        usuario_id,
        date_trunc('month', fecha) as mes_actividad,
        min(date_trunc('month', fecha)) over (partition by usuario_id) as mes_cohorte
    from eventos
),
num_periodo as (
    select
        usuario_id,
        mes_cohorte,
        (date_part('year', mes_actividad) - date_part('year', mes_cohorte)) * 12 +
        (date_part('month', mes_actividad) - date_part('month', mes_cohorte)) as numero_periodo
    from cohorte_base
)
select
    mes_cohorte,
    numero_periodo,
    count(distinct usuario_id) as usuarios_unicos
from num_periodo
group by mes_cohorte, numero_periodo
order by mes_cohorte, numero_periodo;
```

**Pandas:**
```python
eventos['mes_actividad'] = eventos['fecha'].dt.to_period('M')
eventos['mes_cohorte'] = eventos.groupby('usuario_id')['mes_actividad'].transform('min')
eventos['numero_periodo'] = (eventos['mes_actividad'] - eventos['mes_cohorte']).apply(lambda x: x.n)
cohortes = eventos.groupby(['mes_cohorte', 'numero_periodo'])['usuario_id'].nunique().reset_index()
```

**Por qué cada pieza corresponde a la otra:**

| Paso lógico | SQL | Pandas |
|---|---|---|
| Mes de cada evento individual | `date_trunc('month', fecha)` | `.dt.to_period('M')` |
| Mínimo por usuario, sin colapsar filas | `MIN(...) OVER (PARTITION BY usuario_id)` | `.groupby('usuario_id')[...].transform('min')` |
| Diferencia en meses | Fórmula manual con `DATE_PART` (año×12 + mes) | Resta directa de `Period` (pandas ya sabe hacerlo) |
| Colapsar y contar únicos | `GROUP BY` + `COUNT(DISTINCT ...)` | `.groupby([...])[...].nunique()` |

> [!NOTE] Ver el desarrollo completo paso a paso
> Ver [[Funnel_y_Cohortes_S12#cohortes-base]] para la versión con explicación de cada línea.

---

## 🔢 SQL vs NumPy — Operaciones Numéricas {#sql-vs-numpy}

| Qué hace | SQL | NumPy |
|---|---|---|
| Condicional vectorizado | `CASE WHEN cond THEN a ELSE b END` | `np.where(cond, a, b)` |
| Múltiples condiciones anidadas | `CASE WHEN c1 THEN a WHEN c2 THEN b ELSE c END` | `np.select([c1, c2], [a, b], default=c)` |
| Redondear | `ROUND(col, 2)` | `np.round(col, 2)` |
| Nulo/vacío | `NULL` | `np.nan` |
| Verificar si es nulo | `IS NULL` / `IS NOT NULL` | `np.isnan(col)` / `pd.isna(col)` |
| Generar rango de números | No aplica directo (se usaría `generate_series`) | `np.arange(inicio, fin, paso)` |
| Percentil / cuantil | `PERCENTILE_CONT(0.5)` | `np.percentile(col, 50)` |

> [!NOTE] NumPy rara vez se usa "solo" en análisis de datos
> Casi siempre trabajas con NumPy **a través de** pandas (pandas usa NumPy por debajo). Lo usarás directo sobre todo para: `np.where`, `np.select`, generación de datos sintéticos (`np.random`), y operaciones matemáticas vectorizadas puras.

---

## ⚠️ Conceptos que se Confunden Seguido {#confusiones-comunes}

| Confusión típica | Aclaración |
|---|---|
| "`OVER()` es `.transform()` y `PARTITION BY` es `.agg()`" | Incorrecto — **ambos son `.transform()`**. La diferencia es si hay un `.groupby()` antes o no. `.agg()` es lo que colapsa filas, equivalente a un `GROUP BY` normal sin `OVER`. |
| "Media y mediana son intercambiables para imputar nulos" | La mediana es más robusta a outliers (no se deja "jalar" por valores extremos); la media solo es segura si la distribución es simétrica. Ver [[Analisis_Estadistico]]. |
| "`RELATED()` en DAX es como un `JOIN`" | Se parece en el resultado, pero técnicamente `RELATED()` necesita **contexto de fila** para funcionar — por eso no sirve suelto dentro de `SUM()`, sí dentro de una columna calculada o dentro de `SUMX()`. |
| "Un CTE se ejecuta aparte, como una tabla temporal guardada" | No — un CTE solo existe durante esa consulta específica, se recalcula cada vez, no se guarda en la base de datos. |

---

## 🔗 Conexiones Estratégicas

- **Índice Maestro:** [[Indice_Maestro]]
- **Herramientas relacionadas:** [[SQL]] | [[Pandas]] | [[Numpy]]
- **Caso aplicado de cohortes:** [[Funnel_y_Cohortes_S12#cohortes-base]]
- **DAX (traductor pendiente):** cuando se consolide el repaso de DAX, considerar sección "SQL vs DAX" en esta misma nota o nota aparte — `RELATED()` ≈ `JOIN`, `CALCULATE()` ≈ `WHERE` dinámico, `SUMX()` ≈ agregación con cruce de tablas.
