---
tags: [herramienta, sql, python, pandas, numpy, dax, powerbi, traductor, indice]
tipo: nota-traductor
herramientas: [sql, python, pandas, numpy, dax]
---

# 🔄 SQL vs Python vs DAX — Traductor de Conceptos

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
| SQL vs DAX — Traducción de conceptos | [[#sql-vs-dax]] |

---

## 🐼 SQL vs Pandas — Operaciones Base {#sql-vs-pandas-base}

| Qué hace | SQL | Pandas |
|---|---|---|
| Seleccionar columnas | `SELECT col1, col2 FROM tabla` | `df[['col1', 'col2']]` |
| Filtrar filas | `WHERE condicion` | `df[df['col'] > valor]` |
| Filtrar grupos ya agregados | `GROUP BY col HAVING COUNT(*) > N` | `.groupby('col').filter(lambda g: len(g) > N)` |
| Comparar contra nulo | `WHERE col IS NULL` / `IS NOT NULL` | `df[df['col'].isna()]` / `df[df['col'].notna()]` |
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

> [!IMPORTANT] SQL es declarativo (orden fijo interno) — Pandas es imperativo (corre como lo escribes)
> SQL tiene un orden de ejecución lógico fijo, distinto al orden en que lo escribes (`FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY`) — por eso existen reglas como "no puedes usar un alias del SELECT en el WHERE". Ver el desarrollo completo en [[Jerarquia_SQL]]. **Pandas no tiene ese problema**: se ejecuta línea por línea, exactamente en el orden que tú lo escribes — no hay "orden lógico interno" oculto. Por eso en pandas nunca te vas a topar con la restricción de "esta variable todavía no existe" — si ya la asignaste en una línea anterior, está disponible.

> [!WARNING] La trampa de LEFT JOIN con filtro mal ubicado existe en ambos lenguajes
> En SQL, filtrar la tabla derecha en `WHERE` en vez de en `ON` convierte un `LEFT JOIN` en `INNER JOIN` de facto (ver [[Jerarquia_SQL#otras-reglas]]). En pandas pasa lo mismo: hacer `pd.merge(df1, df2, how='left')` y luego filtrar con `.query()` o `df[condicion]` sobre una columna de `df2` **elimina las filas sin match** (que tienen `NaN` en esa columna), deshaciendo el propósito del `left`. Si necesitas conservar las filas sin match, filtra *antes* del merge, o usa `.loc` con una condición que explícitamente permita `NaN` (ej. `(df['col'] > 5) | (df['col'].isna())`).

---

## 🪟 SQL vs Pandas — Funciones de Ventana {#sql-vs-pandas-ventana}

**Concepto central: ninguna de las dos "colapsa" filas** — a diferencia de un `GROUP BY`/`.agg()` normal, que sí reduce el número de filas, las funciones de ventana devuelven el mismo número de filas con el agregado "pegado" en cada una.

| Qué hace | SQL | Pandas |
|---|---|---|
| Agregado global (todas las filas) | `SUM(col) OVER ()` | `df['col'].transform('sum')` |
| Agregado por grupo (sin colapsar) | `SUM(col) OVER (PARTITION BY grupo)` | `df.groupby('grupo')['col'].transform('sum')` |
| Ranking dentro de cada grupo | `RANK() OVER (PARTITION BY grupo ORDER BY col DESC)` | `df.groupby('grupo')['col'].rank(ascending=False)` |
| Número de fila secuencial por grupo | `ROW_NUMBER() OVER (PARTITION BY grupo ORDER BY col)` | `df.groupby('grupo').cumcount() + 1` |

> [!IMPORTANT] Diferencia `.agg()` vs `.transform()` — desarrollo completo en otra nota
> `.agg()` colapsa filas (una por grupo), `.transform()` no colapsa nada (mismo número de filas, valor del grupo repetido en cada una) — igual que `GROUP BY` normal vs `OVER(...)` en SQL. Ver la explicación completa, con ejemplos de lambda, `.transform('rank')` para top-N por grupo, y más casos prácticos, en [[Groupby_Agg_Transform#agg-vs-transform]].

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
| "`OVER()` es `.transform()` y `PARTITION BY` es `.agg()`" | Incorrecto — **ambos son `.transform()`**. La diferencia es si hay un `.groupby()` antes o no. `.agg()` es lo que colapsa filas, equivalente a un `GROUP BY` normal sin `OVER`. Desarrollo completo: [[Groupby_Agg_Transform#agg-vs-transform]]. |
| "Media y mediana son intercambiables para imputar nulos" | La mediana es más robusta a outliers (no se deja "jalar" por valores extremos); la media solo es segura si la distribución es simétrica. Ver [[Analisis_Estadistico]]. |
| "`RELATED()` en DAX es como un `JOIN`" | Se parece en el resultado, pero técnicamente `RELATED()` necesita **contexto de fila** para funcionar — por eso no sirve suelto dentro de `SUM()`, sí dentro de una columna calculada o dentro de `SUMX()`. |
| "Un CTE se ejecuta aparte, como una tabla temporal guardada" | No — un CTE solo existe durante esa consulta específica, se recalcula cada vez, no se guarda en la base de datos. |

---

## 🧊 SQL vs DAX — Traducción de Conceptos {#sql-vs-dax}

**Diferencia de fondo antes de la tabla:** SQL y Pandas piensan en términos de **agrupar/particionar filas**. DAX piensa en términos de **contexto de fila y contexto de filtro** — no agrupa, filtra y aísla. El resultado final suele ser equivalente, pero el mecanismo mental es distinto. Ver [[#contexto-dax]] más abajo para el desarrollo de esa idea.

| Qué hace | SQL | DAX |
|---|---|---|
| Traer un valor de una tabla relacionada | `JOIN` | `RELATED(tabla[columna])` — requiere contexto de fila, funciona en columnas calculadas o dentro de `SUMX`, no suelto en un `SUM()` |
| Calcular algo fila por fila, cruzando tablas, y sumar | No aplica directo (el `JOIN` ya lo resuelve antes de agregar) | `SUMX(tabla, expresión_por_fila)` |
| Sumar una columna que ya existe, sin cruces | `SUM(columna)` | `SUM(tabla[columna])` |
| Filtro condicional dentro de una agregación | `WHERE` (antes de agregar) o `CASE WHEN` dentro de `SUM` | `CALCULATE(medida, filtro)` — el filtro se aplica de forma dinámica, no estática |
| Agrupar/particionar sin colapsar filas | `PARTITION BY columna` | `ALLEXCEPT(tabla, tabla[columna])` dentro de un `CALCULATE` — ver [[#contexto-dax]] |
| Evitar división entre cero | `NULLIF(denominador, 0)` | `DIVIDE(numerador, denominador, [valor_alternativo])` |
| Contar valores únicos | `COUNT(DISTINCT columna)` | `DISTINCTCOUNT(tabla[columna])` |
| Referenciar un cálculo ya definido | Repetir el CTE/subquery, o usar su alias si el motor lo permite | `[NombreMedida]` — sin nombre de tabla, a diferencia de las columnas (`Tabla[Columna]`) |

> [!IMPORTANT] Columnas vs Medidas — no tienen equivalente 1 a 1 en SQL
> SQL no distingue "columna calculada" de "medida" — todo es una expresión dentro de la consulta. En DAX sí importa mucho la diferencia: una **columna calculada** se computa una vez por fila y se guarda físicamente en el modelo (como una columna más de la tabla); una **medida** se recalcula dinámicamente según el contexto de filtro del reporte (nunca se guarda con un valor fijo). Ver [[DAX_Modelado_PowerBI]] para el desarrollo completo.

### 🧠 Contexto de fila / contexto de filtro (la pieza que no existe en SQL) {#contexto-dax}

Esto no tiene equivalente directo en SQL, así que merece explicación aparte en vez de una fila de tabla:

- Por default, una columna calculada en DAX está "aislada" a su propia fila — como si cada fila tuviera un filtro invisible que solo la deja ver a ella misma.
- `ALLEXCEPT(tabla, tabla[columna])` amplía esa vista: dice "deja de estar aislado a una sola fila, ahora puedes ver todas las filas de la tabla otra vez — pero conserva el filtro de esta columna en particular" (ej. mismo `id_cliente`).
- Traducido a pandas: el estado por default es como `df.groupby(df.index)['col'].transform(...)` (cada fila es su propio grupo, inútil). `ALLEXCEPT` es como cambiar la llave de ese `groupby` de "el índice de cada fila" a la columna real que te interesa (`df.groupby('id_cliente')['col'].transform(...)`) — **nunca deja de ser un `.transform()`**, solo cambia la llave de agrupación.
- `CALCULATE()` es lo que "ejecuta" ese cambio de contexto — sin él, modificadores como `ALLEXCEPT` o `ALL` no pueden correr por sí solos.

**Ejemplo aplicado — mismo problema (mes de cohorte por cliente), 3 lenguajes:**

```sql
-- SQL
MIN(date_trunc('month', fecha)) OVER (PARTITION BY usuario_id)
```
```python
# Pandas
df.groupby('usuario_id')['mes_actividad'].transform('min')
```
```dax
// DAX
Primera compra por cliente =
CALCULATE(
    MIN(hecho_ventas[fecha_venta]),
    ALLEXCEPT(hecho_ventas, hecho_ventas[id_cliente])
)
```

Los tres calculan exactamente lo mismo: "el mínimo de esta columna, agrupado por cliente, sin colapsar filas" — solo cambia el mecanismo (particionar, agrupar+transformar, o filtrar+aislar).

> [!NOTE] Ver el caso completo de cohortes en DAX
> Ver [[DAX_Modelado_PowerBI#cohortes]] para la construcción completa de la matriz de cohortes en Power BI (incluye `FORMAT` para las etiquetas de mes y la configuración del visual tipo Matriz).

---

## 🔗 Conexiones Estratégicas

- **Índice Maestro:** [[Indice_Maestro]]
- **Herramientas relacionadas:** [[SQL]] | [[Pandas]] | [[Numpy]] | [[Power_BI]]
- **Desarrollo completo de `.agg()` vs `.transform()`:** [[Groupby_Agg_Transform#agg-vs-transform]]
- **Orden de ejecución de SQL (por qué WHERE ≠ HAVING, alias, etc.):** [[Jerarquia_SQL]]
- **Práctica aplicada de estos patrones (DataLemur):** [[Practica_DataLemur]]
- **Caso aplicado de cohortes (SQL/Pandas):** [[Funnel_y_Cohortes_S12#cohortes-base]]
- **Caso aplicado de cohortes (DAX):** [[DAX_Modelado_PowerBI#cohortes]]
- **DAX intermedio (SUMX, RELATED, DIVIDE):** [[DAX_Avanzado_S12]]
- **DAX modelado y CALCULATE + ALLEXCEPT:** [[DAX_Modelado_PowerBI#allexcept]]

