---
tags: [operacion, sql, funnel, cohortes, retencion, lag, case-when, sprint-12]
tipo: nota-operacion
herramientas: [sql, python]
---

# 🌊 Funnel de Conversión y Cohortes de Retención — SQL + Python

Operaciones para analizar el comportamiento de usuarios en un funnel de compra y medir la retención por cohortes, combinando SQL para la extracción y Python para la visualización. Patrón del S12 con `pd.read_sql`.

---

## 📋 Índice de Operaciones

| Operación | Ir a |
|---|---|
| Funnel: conteo de usuarios por etapa | [[#funnel-conteo]] |
| Funnel: tasa de conversión con LAG() | [[#funnel-conversion]] |
| Cohortes: retención semanal con LEFT JOIN + CASE WHEN | [[#cohortes-sql]] |
| Cohortes: transformación y heatmap en Python | [[#cohortes-python]] |
| Cohortes: patrón base con Window Function (práctica) | [[#cohortes-base]] |

---

## 📊 Funnel — Conteo de Usuarios por Etapa {#funnel-conteo}

**Cuándo:** Para saber cuántos usuarios únicos llegan a cada paso del proceso de compra (visita → carrito → pago → compra).

```python
query_funnel = '''
SELECT 
    nombre_evento,
    COUNT(DISTINCT id_usuario) AS total_usuarios
FROM events
WHERE nombre_evento IN (
    'first_visit', 'select_item', 'add_to_cart',
    'begin_checkout', 'add_payment_info', 'purchase'
)
GROUP BY nombre_evento
ORDER BY 
    CASE nombre_evento
        WHEN 'first_visit'       THEN 1
        WHEN 'select_item'       THEN 2
        WHEN 'add_to_cart'       THEN 3
        WHEN 'begin_checkout'    THEN 4
        WHEN 'add_payment_info'  THEN 5
        WHEN 'purchase'          THEN 6
    END;
'''

funnel_data = pd.read_sql(query_funnel, con=engine)
funnel_data
```

**Puntos clave:**
- `COUNT(DISTINCT id_usuario)` — cuenta usuarios únicos, no eventos. Un usuario puede visitar 3 veces pero cuenta como 1
- `WHERE nombre_evento IN (...)` — filtra solo las etapas del funnel, elimina eventos de ruido
- `CASE` en el `ORDER BY` — ordena el funnel en secuencia lógica, no alfabéticamente

> [!IMPORTANT] Sin el filtro WHERE el funnel no tiene sentido
> Si no filtras los eventos relevantes, aparecen etapas con más usuarios que la etapa anterior (tasas > 100%), lo que rompe la lógica del funnel.

**Contexto real:** S12 — Funnel de 6 etapas para el análisis de conversión de RappiPlus/MercadoLibre.

---

## 📉 Funnel — Tasa de Conversión con LAG() {#funnel-conversion}

**Cuándo:** Para calcular qué porcentaje de usuarios pasa de una etapa a la siguiente, e identificar el "cuello de botella" donde se pierde más gente.

```python
query_conversion = '''
WITH funnel_counts AS (
    SELECT 
        nombre_evento,
        COUNT(DISTINCT id_usuario) AS total_usuarios
    FROM events
    WHERE nombre_evento IN (
        'first_visit', 'select_item', 'add_to_cart',
        'begin_checkout', 'add_payment_info', 'purchase'
    )
    GROUP BY nombre_evento
),
ordered_funnel AS (
    SELECT 
        nombre_evento,
        total_usuarios,
        LAG(total_usuarios) OVER (
            ORDER BY 
                CASE nombre_evento
                    WHEN 'first_visit'       THEN 1
                    WHEN 'select_item'       THEN 2
                    WHEN 'add_to_cart'       THEN 3
                    WHEN 'begin_checkout'    THEN 4
                    WHEN 'add_payment_info'  THEN 5
                    WHEN 'purchase'          THEN 6
                END
        ) AS usuarios_paso_anterior
    FROM funnel_counts
)
SELECT 
    nombre_evento,
    total_usuarios,
    ROUND(
        total_usuarios::numeric / NULLIF(usuarios_paso_anterior, 0), 4
    ) AS conversion_rate
FROM ordered_funnel
ORDER BY 
    CASE nombre_evento
        WHEN 'first_visit'       THEN 1
        WHEN 'select_item'       THEN 2
        WHEN 'add_to_cart'       THEN 3
        WHEN 'begin_checkout'    THEN 4
        WHEN 'add_payment_info'  THEN 5
        WHEN 'purchase'          THEN 6
    END;
'''

conversion = pd.read_sql(query_conversion, con=engine)
conversion
```

**Anatomía del patrón:**

| Función | Qué hace |
|---|---|
| `LAG(total_usuarios) OVER (ORDER BY CASE...)` | Trae el valor de la fila anterior según el orden del funnel |
| `::numeric` | Convierte entero a decimal para evitar división entera |
| `NULLIF(..., 0)` | Evita error de división por cero si una etapa tiene 0 usuarios |
| `ROUND(..., 4)` | 4 decimales para ver el porcentaje con precisión |

> [!NOTE] LAG() es el equivalente SQL de .shift(1) en Pandas
> `LAG(col)` trae el valor de la fila anterior en la ventana ordenada. Es la herramienta estándar para calcular variaciones entre filas consecutivas.

**Contexto real:** S12 — Tasa de conversión del funnel de MercadoLibre. Se detectó que algunos pasos tenían tasas > 100% porque los usuarios entran al carrito desde múltiples puntos de entrada (no solo desde `select_item`).

---

## 🔁 Cohortes — Retención Semanal con SQL {#cohortes-sql}

**Cuándo:** Para medir qué porcentaje de usuarios registrados en un mes sigue activo en las semanas siguientes. Estructura diferente al S4 (que usaba CTEs de cohortes mensuales puras).

```python
query_cohort = '''
SELECT 
    DATE_TRUNC('month', CAST(u.fecha_registro AS DATE)) AS cohort_month,
    COUNT(DISTINCT u.id_usuario)                         AS total_usuarios,
    
    -- Retención semanal: activo = 1 dentro del rango de días
    COUNT(DISTINCT CASE 
        WHEN a.dias_despues_registro BETWEEN 7  AND 13 AND a.activo = 1 
        THEN u.id_usuario END) AS retenido_w1,
    
    COUNT(DISTINCT CASE 
        WHEN a.dias_despues_registro BETWEEN 14 AND 20 AND a.activo = 1 
        THEN u.id_usuario END) AS retenido_w2,
    
    COUNT(DISTINCT CASE 
        WHEN a.dias_despues_registro BETWEEN 21 AND 27 AND a.activo = 1 
        THEN u.id_usuario END) AS retenido_w3

FROM users u
LEFT JOIN user_activity a ON u.id_usuario = a.id_usuario
GROUP BY 1
ORDER BY 1;
'''

cohorte_data = pd.read_sql(query_cohort, con=engine)
```

**Por qué este patrón es diferente al S4:**

| S4 (MercadoLibre) | S12 (RappiPlus) |
|---|---|
| CTE + JOIN de retención D7/D14/D28 | CASE WHEN directo con rangos de días |
| Columna `day_after_signup` calculada | Columna `dias_despues_registro` ya existe |
| Conteo de usuarios activos por período | Mismo resultado, sintaxis más directa |
| `COUNT(DISTINCT CASE WHEN day >= N AND active = 1)` | `COUNT(DISTINCT CASE WHEN dias BETWEEN X AND Y AND activo = 1)` |

> [!IMPORTANT] LEFT JOIN es obligatorio aquí
> Con `INNER JOIN`, los usuarios que se registraron pero nunca tuvieron actividad desaparecen del resultado. El `total_usuarios` quedaría inflado (solo contaría activos), haciendo que las tasas de retención superen el 100%. El `LEFT JOIN` conserva a TODOS los registrados, incluso sin actividad.

> [!WARNING] El filtro `activo = 1` va DENTRO del CASE WHEN
> Si pones `WHERE activo = 1` en el filtro principal, eliminas a los inactivos antes de contar, arruinando el denominador. El filtro debe ir dentro de cada `CASE WHEN` para que solo afecte al numerador.

**Contexto real:** S12 — La tabla `user_activity` ya tenía `dias_despues_registro` y `activo`, lo que permitió simplificar la query vs. el patrón del S4.

---

## 🐍 Cohortes — Transformación y Heatmap en Python {#cohortes-python}

**Cuándo:** Una vez que `pd.read_sql` devuelve la tabla de cohortes (en formato "ancho"), calcular los porcentajes y visualizar como mapa de calor.

```python
# 1. Convertir la columna de fecha a datetime
cohorte_data['cohort_month'] = pd.to_datetime(cohorte_data['cohort_month'])

# 2. Usar cohort_month como índice
cohorte_data.set_index('cohort_month', inplace=True)

# 3. Formatear índice a 'YYYY-MM' para que sea legible en el eje Y
cohorte_data.index = cohorte_data.index.strftime('%Y-%m')

# 4. Calcular porcentajes de retención (dividir retenidos / total)
retention_df = cohorte_data.copy()
retention_df['w1'] = retention_df['retenido_w1'] / retention_df['total_usuarios']
retention_df['w2'] = retention_df['retenido_w2'] / retention_df['total_usuarios']
retention_df['w3'] = retention_df['retenido_w3'] / retention_df['total_usuarios']

# 5. Seleccionar solo las columnas de porcentaje
heatmap_data = retention_df[['w1', 'w2', 'w3']]

# 6. Visualizar
import seaborn as sns
import matplotlib.pyplot as plt

plt.figure(figsize=(10, 6))
sns.heatmap(heatmap_data, annot=True, fmt='.1%', cmap='YlGnBu')
plt.title('Retención de usuarios por cohorte mensual (semanas 1-3)')
plt.ylabel('Mes de registro')
plt.show()
```

**¿Por qué este flujo es más simple que el S4?**
- La SQL ya entrega el resultado en formato "ancho" (`retenido_w1`, `retenido_w2`, `retenido_w3`)
- No necesitas `pivot_table` — la tabla ya está pivotada desde SQL
- Solo calculas porcentajes dividiendo columnas y graficar

> [!WARNING] Error 1970-01 en el índice
> Si el índice del heatmap muestra `1970-01`, significa que Pandas está interpretando la fecha como milisegundos (timestamp Unix). Solución: `pd.to_datetime(cohorte_data['cohort_month'])` antes de `set_index`.

**Contexto real:** S12 — Heatmap de retención de usuarios de RappiPlus con cohortes mensuales y retención semanal W1/W2/W3.

---

## 🔗 Conexiones Estratégicas

- **Índice Maestro:** [[Indice_Maestro]]
- **Herramientas:** [[SQL]] | [[Python_SQL]] | [[Pandas]]
- **Operación relacionada S4:** [[SQL_Financiero_y_Metricas#embudo-ctes]] | [[SQL_Financiero_y_Metricas#cohortes]]
- **Visualización:** [[Visualizacion#heatmap]]
- **Sprint de referencia:** S12 — Proyecto Final RappiPlus

---

## 🔁 Cohortes Acumuladas — Patrón con >= N días {#cohortes-acumuladas}

**Cuándo:** Alternativa al patrón `BETWEEN` cuando las instrucciones piden retención acumulada (usuarios activos **desde** el día N en adelante, no solo en esa ventana).

```python
query_cohortes_acumuladas = '''
SELECT
    DATE_TRUNC('month', CAST(u.fecha_registro AS DATE)) AS cohort_month,
    COUNT(DISTINCT u.id_usuario) AS total_usuarios,

    -- Retención acumulada: activo EN O DESPUÉS del día N
    COUNT(DISTINCT CASE 
        WHEN a.dias_despues_registro >= 7  AND a.activo = 1 
        THEN u.id_usuario END) AS retenido_d7,

    COUNT(DISTINCT CASE 
        WHEN a.dias_despues_registro >= 14 AND a.activo = 1 
        THEN u.id_usuario END) AS retenido_d14,

    COUNT(DISTINCT CASE 
        WHEN a.dias_despues_registro >= 21 AND a.activo = 1 
        THEN u.id_usuario END) AS retenido_d21

FROM users u
LEFT JOIN user_activity a ON u.id_usuario = a.id_usuario
GROUP BY 1
ORDER BY 1;
'''

cohorte_acum = pd.read_sql(query_cohortes_acumuladas, con=engine)
```

**Diferencia clave entre los dos patrones:**

| Patrón | Sintaxis SQL | Qué mide |
|---|---|---|
| Ventana semanal | `BETWEEN 7 AND 13` | Usuarios activos **solo** en esa semana |
| Acumulado | `>= 7` | Usuarios activos **desde** ese día en adelante |

> [!NOTE] ¿Cuál usar?
> - `BETWEEN` → retención por período (¿volvieron esa semana específica?)
> - `>= N` → retención acumulada (¿siguen activos después de N días?) — más común en análisis de producto

**Contexto real:** S12 — Patrón visto en el contexto del S4 (MercadoLibre D7/D14/D21/D28) y aplicado también en RappiPlus.

---

## 🔀 Funnel con INTERSECT — Patrón Alternativo {#funnel-intersect}

**Cuándo:** Para construir un funnel estricto donde cada etapa solo cuenta usuarios que **también** pasaron por todas las etapas anteriores. Diferente al patrón de `LEFT JOIN` que cuenta usuarios únicos por etapa independientemente.

```python
query_funnel_intersect = '''
-- Usuarios que llegaron a cada etapa (y también pasaron por las anteriores)
SELECT 'first_visit'      AS etapa, COUNT(DISTINCT id_usuario) AS usuarios, 1 AS orden
FROM events WHERE nombre_evento = 'first_visit'

UNION ALL

SELECT 'select_item', COUNT(DISTINCT id_usuario), 2
FROM events WHERE nombre_evento = 'select_item'
AND id_usuario IN (SELECT DISTINCT id_usuario FROM events WHERE nombre_evento = 'first_visit')

UNION ALL

SELECT 'add_to_cart', COUNT(DISTINCT id_usuario), 3
FROM events WHERE nombre_evento = 'add_to_cart'
AND id_usuario IN (SELECT DISTINCT id_usuario FROM events WHERE nombre_evento = 'select_item')

UNION ALL

SELECT 'purchase', COUNT(DISTINCT id_usuario), 4
FROM events WHERE nombre_evento = 'purchase'
AND id_usuario IN (SELECT DISTINCT id_usuario FROM events WHERE nombre_evento = 'add_to_cart')

ORDER BY orden;
'''

funnel_estricto = pd.read_sql(query_funnel_intersect, con=engine)
```

**Diferencia entre los dos patrones de funnel:**

| Patrón | Cómo cuenta | Cuándo usarlo |
|---|---|---|
| `COUNT(DISTINCT) + WHERE IN (...)` | Solo usuarios que pasaron por etapas previas | Funnel estricto y secuencial |
| `COUNT(DISTINCT) + GROUP BY evento` | Todos los usuarios únicos por etapa, independientemente | Funnel de alcance (más común) |

> [!NOTE] Los resultados pueden diferir significativamente
> El patrón con `IN` siempre da números menores o iguales al patrón de `GROUP BY`, porque filtra usuarios que saltaron etapas. Elige según lo que quiera medir el negocio.

**Contexto real:** S12 — Patrón alternativo para análisis de funnel estricto donde cada etapa es prerequisito de la siguiente.

----
### 🐍 Cohortes: Cálculo 100% en Pandas {#cohortes-pandas}

**Cuándo:** Cuando el dataset es pequeño y prefieres realizar todo el procesamiento en memoria sin depender de una query SQL compleja.

```python
# 1. Asegurar formato datetime
df['fecha_registro'] = pd.to_datetime(df['fecha_registro'])
df['fecha_actividad'] = pd.to_datetime(df['fecha_actividad'])

# 2. Truncar a mes (Period)
df['mes_registro'] = df['fecha_registro'].dt.to_period('M')
df['mes_actividad'] = df['fecha_actividad'].dt.to_period('M')

# 3. Identificar la cohorte (mes de la primera actividad)
df['mes_cohorte'] = df.groupby('id_usuario')['mes_registro'].transform('min')

# 4. Calcular el índice de periodo (meses transcurridos)
df['periodo'] = (df['mes_actividad'] - df['mes_cohorte']).apply(lambda x: x.n)

# 5. Agrupar y pivotar para obtener la matriz de retención
cohortes = df.groupby(['mes_cohorte', 'periodo'])['id_usuario'].nunique().reset_index()
matriz_cohortes = cohortes.pivot(index='mes_cohorte', columns='periodo', values='id_usuario')

# 6. Calcular porcentajes (dividir por la primera columna)
retention_matrix = matriz_cohortes.divide(matriz_cohortes.iloc[:, 0], axis=0)

# 7. Visualizar
sns.heatmap(retention_matrix, annot=True, fmt='.1%', cmap='YlGnBu')
```

**Puntos clave:**

- `.transform('min')`: Es la forma más eficiente de asignar la fecha de registro a cada fila del usuario sin colapsar el DataFrame.
- `.apply(lambda x: x.n)`: Extrae el número entero de la diferencia de periodos.
- `.divide(..., axis=0)`: Divide cada fila por su valor inicial (la cohorte), convirtiendo los conteos en tasas de retención.

---

## 🔁 Cohortes — Patrón Base de Práctica: SQL con Window Function + Pandas paso a paso {#cohortes-base}

**Cuándo:** Patrón mínimo y didáctico para reforzar la lógica de cohortes desde cero, sin depender de columnas ya calculadas (`dias_despues_registro`, `activo`) como en los patrones S12 de arriba. Útil para practicar en plataformas como DataLemur/StrataScratch o para explicarle el concepto a alguien más.

### Versión SQL (CTEs + Window Function)

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

**Puntos clave:**
- `MIN(...) OVER (PARTITION BY usuario_id)` reemplaza el patrón "CTE + LEFT JOIN" del S4 — calcula el mínimo sin colapsar filas, ahorrando un CTE completo.
- La fórmula `(año_actividad - año_cohorte) * 12 + (mes_actividad - mes_cohorte)` es necesaria porque restar fechas da diferencia en días, no en meses — hay que armar la conversión a mano con `DATE_PART`.

### Versión Pandas (mismo resultado, sin SQL)

```python
import pandas as pd

# Paso 1: mes de cada evento individual
eventos['mes_actividad'] = eventos['fecha'].dt.to_period('M')

# Paso 2: mes de cohorte (primera actividad) de cada usuario
eventos['mes_cohorte'] = eventos.groupby('usuario_id')['mes_actividad'].transform('min')

# Paso 3: número de periodo (meses de diferencia)
eventos['numero_periodo'] = eventos['mes_actividad'] - eventos['mes_cohorte']
eventos['numero_periodo'] = eventos['numero_periodo'].apply(lambda x: x.n)

# Paso 4: agrupar y contar usuarios únicos por cohorte y periodo
cohortes = eventos.groupby(['mes_cohorte', 'numero_periodo'])['usuario_id'].nunique()
cohortes = cohortes.reset_index()
```

### Tabla de equivalencias SQL ↔ Pandas

| Objetivo | SQL | Pandas |
|---|---|---|
| Truncar fecha a mes | `date_trunc('month', fecha)` | `.dt.to_period('M')` |
| Mínimo por grupo sin colapsar filas | `MIN(...) OVER (PARTITION BY usuario_id)` | `.groupby('usuario_id')[...].transform('min')` |
| Diferencia en meses | `(año_act - año_coh)*12 + (mes_act - mes_coh)` | Resta directa de `Period` + `.apply(lambda x: x.n)` |
| Agrupar y contar únicos | `GROUP BY ... ; COUNT(DISTINCT usuario_id)` | `.groupby([...])[...].nunique()` |

> [!NOTE] Diferencia con el patrón `cohortes-pandas` de arriba
> Esta versión es el "esqueleto" mínimo (sin pivot, sin heatmap, sin porcentajes) — pensado para practicar la lógica base. El patrón `cohortes-pandas` de la sección anterior ya incluye el pivot (`.pivot()`) y el cálculo de retención en % (`.divide()`), listo para producción/heatmap.

