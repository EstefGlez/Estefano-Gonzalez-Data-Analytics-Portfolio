---
tags: [operacion, pandas, groupby, agg, transform, lambda, kaggle, practica]
tipo: nota-operacion
herramientas: [pandas]
---

# 🧮 Groupby — `.agg()` vs `.transform()`, Lambda, `.isin()`/`.where()` y Crosstab

Operaciones de agrupación en Pandas practicadas con el dataset **Superstore Sales** (Kaggle), como paso previo a las funciones de ventana de SQL (`#cohortes-base` en [[Funnel_y_Cohortes_S12]]). El objetivo de esta nota es tener claro cuándo un `groupby` debe **colapsar** el resultado y cuándo debe **conservar el detalle de fila**.

---

## 📋 Índice de Operaciones

| Operación | Ir a |
|---|---|
| `.agg()` vs `.transform()` — diferencia conceptual | [[#agg-vs-transform]] |
| `.agg()` con múltiples funciones y nombres custom | [[#agg-multiple]] |
| `.transform()` para comparar fila vs. grupo | [[#transform-diferencia]] |
| `.transform('rank')` — top-N por grupo | [[#transform-rank]] |
| Funciones lambda en `.agg()` / `.transform()` | [[#lambda]] |
| Negación `~.isin()` y combinación con `.notna()` | [[#negacion-isin]] |
| `.isin()` + `.where()` vs. filtro con máscara | [[#isin-where]] |
| `pd.crosstab()` y `pivot_table()` | [[#crosstab-pivot]] |

---

## ⚖️ `.agg()` vs `.transform()` — Diferencia Conceptual {#agg-vs-transform}

**Cuándo:** Punto de partida obligatorio antes de usar cualquiera de los dos — la elección depende de si necesitas un resumen (reporte) o un detalle enriquecido (fila por fila).

```python
import pandas as pd

df = pd.DataFrame({
    'categoria': ['A', 'A', 'B', 'B', 'B'],
    'ventas':    [10,  20,  5,   15,  25]
})

# --- agg(): COLAPSA, 1 fila por grupo ---
resultado_agg = df.groupby('categoria')['ventas'].agg('mean')
# categoria
# A    15.0
# B    15.0

# --- transform(): NO colapsa, misma forma que df original ---
df['promedio_categoria'] = df.groupby('categoria')['ventas'].transform('mean')
#   categoria  ventas  promedio_categoria
# 0         A      10                15.0
# 1         A      20                15.0
# 2         B       5                15.0
# 3         B      15                15.0
# 4         B      25                15.0
```

**Puntos clave:**
- `.agg()` → resumen/reporte. Ideal para "ventas totales por región".
- `.transform()` → conserva el largo original del DataFrame. Ideal para comparar cada fila contra el agregado de su propio grupo, sin perder el detalle.
- Ambos aceptan los mismos strings: `'mean'`, `'sum'`, `'count'`, `'min'`, `'max'`, `'std'`, `'median'`, `'first'`, `'last'`, `'nunique'`. Exclusivos de `transform`: `'rank'`, `'cumsum'`, `'pct_change'` (necesitan devolver un valor por fila, no un resumen).

> [!NOTE] Equivalencia con SQL — Window Functions
> `.transform()` es, conceptualmente, el mismo patrón que `AVG(...) OVER (PARTITION BY categoria)` en SQL: agrega sin colapsar filas. Es la misma lógica que se usa en `MIN(...) OVER (PARTITION BY usuario_id)` del patrón de cohortes — ver [[Funnel_y_Cohortes_S12#cohortes-base]].

**Contexto real:** Práctica en Kaggle con el dataset Superstore Sales, como paso previo a DataLemur/StrataScratch (window functions en SQL).

---

## 📊 `.agg()` con Múltiples Funciones y Nombres Custom {#agg-multiple}

**Cuándo:** Para un reporte ejecutivo donde se necesitan varias métricas a la vez, agrupando por una o más columnas.

```python
# Varias funciones sobre la misma columna
df.groupby('Category')['Sales'].agg(['mean', 'sum', 'max', 'min'])

# Con nombres de columna custom (recomendado para reportes legibles)
reporte = df.groupby('Category')['Sales'].agg(
    promedio='mean',
    total='sum',
    maximo='max'
)

# Distintas funciones por columna, y groupby de dos niveles
resumen = df.groupby(['Region', 'Category']).agg({
    'Sales': 'sum',
    'Quantity': 'max'
}).sort_values('Sales', ascending=False)
```

**Puntos clave:**
- El diccionario `{'columna': 'funcion'}` permite aplicar agregaciones distintas por columna en una sola llamada.
- `.sort_values()` después del `agg()` es necesario si se pide un orden específico — el `groupby` por default ordena alfabéticamente por las claves de agrupación, no por el valor agregado.
- `.count()` sobre un ID único de fila (ej. `Row ID`) cuenta **filas**; `.nunique()` sobre un ID de entidad (ej. `Order ID`) cuenta **entidades distintas**. No son intercambiables si una entidad puede tener varias filas (ej. una orden con varios productos).

**Contexto real:** Kaggle Superstore — reporte ejecutivo de ventas totales y promedio por `Category` y por `Region x Category`.

---

## 🔍 `.transform()` para Comparar Fila vs. Grupo {#transform-diferencia}

**Cuándo:** Cuando se necesita detectar valores atípicos dentro de un grupo (ej. productos que venden muy por debajo del promedio de su Sub-Category), sin perder el detalle de cada fila.

```python
# 1. Promedio de Sales de la Sub-Category de cada fila
df['promedio_subcategoria'] = df.groupby('Sub-Category')['Sales'].transform('mean')

# 2. Diferencia de cada venta individual contra ese promedio
df['diferencia_vs_promedio'] = df['Sales'] - df['promedio_subcategoria']

# 3. Filtrar productos que venden muy por debajo de lo normal en su Sub-Category
productos_debiles = df[df['diferencia_vs_promedio'] < -50]
```

> [!WARNING] Cuidado con la columna de agrupación
> Es fácil confundir columnas similares (ej. `Category` de 3 valores vs. `Sub-Category` de ~17 valores). Agrupar por la columna equivocada no lanza error — simplemente da un resultado menos granular y silenciosamente incorrecto. Siempre verificar con `df['columna'].nunique()` antes de agrupar si hay duda.

**Contexto real:** Kaggle Superstore — identificación de productos con ventas muy por debajo del promedio de su propia Sub-Category (diferencia < -50).

---

## 🏆 `.transform('rank')` — Top-N por Grupo {#transform-rank}

**Cuándo:** Para rankear filas dentro de cada grupo y quedarse con los N mejores de cada uno — patrón muy común en SQL con `RANK() OVER (PARTITION BY ... ORDER BY ...)`.

```python
# Ranking de cada orden dentro de su propia Region, por Sales (mayor = rank 1)
df['rank_en_region'] = df.groupby('Region')['Sales'].transform('rank', ascending=False)

# Top 3 de cada región
top_3 = df[df['rank_en_region'] <= 3]
top_3_ordenado = top_3.sort_values(by=['Region', 'rank_en_region'])
```

**Puntos clave:**
- `.transform('rank', ascending=False)` da rank 1 al valor más alto de cada grupo — igual que `RANK() OVER (PARTITION BY Region ORDER BY Sales DESC)` en SQL.
- El patrón "top-N por grupo" siempre es: `transform('rank')` → filtrar con `<= N` → (opcional) `sort_values()` para presentación.

**Contexto real:** Kaggle Superstore — top 3 órdenes por `Sales` dentro de cada `Region`. Patrón directamente reutilizable para preguntas de DataLemur tipo "top N per category".

---

## 🐑 Funciones Lambda en `.agg()` / `.transform()` {#lambda}

**Cuándo:** Cuando la operación necesaria no existe como string (`'mean'`, `'sum'`, etc.) porque es una fórmula custom de una sola línea.

**Qué es una lambda:** una función sin nombre, de una sola expresión. Es la forma corta de escribir una función con `def`:

```python
# Función normal
def cuadrado(x):
    return x ** 2

# Misma función como lambda
cuadrado = lambda x: x ** 2
```

Sintaxis: `lambda argumentos: expresión_que_se_retorna`. No lleva `return` explícito — lo que sigue a `:` se retorna automáticamente, y solo puede ser una expresión (no varias líneas ni statements).

**Uso dentro de groupby:** pandas le pasa a la lambda la Serie de valores de **un grupo a la vez**.

```python
# Rango (max - min) por grupo -> colapsa, es un agg
df.groupby('Sub-Category')['Sales'].agg(lambda x: x.max() - x.min())

# Z-score (estandarización) dentro de cada grupo -> no colapsa, es un transform
df['z_score'] = df.groupby('Category')['Sales'].transform(
    lambda x: (x - x.mean()) / x.std()
)
```

**Regla mental:** si la operación es una sola palabra que ya existe (`'mean'`, `'sum'`, `'rank'`) → usarla directo como string, es más rápido. Si hay que combinar varias operaciones en una fórmula → lambda.

**Contexto real:** Kaggle Superstore — practicado como extensión de las Tareas 3 y 4 (diferencia vs. promedio y ranking), reemplazando el string por una fórmula custom.

---

## 🚫 Negación con `~.isin()` y Combinación con `.notna()` {#negacion-isin}

**Cuándo:** Para el caso inverso de `.isin()` — cuando se necesita **excluir** un subconjunto de categorías en vez de incluirlo, antes de un `groupby`.

```python
# ~ niega la máscara booleana devuelta por .isin() -> "todo EXCEPTO estos"
mascara = ~df["director"].isin(["director 1", "director 2", "director 3"])
resultado = df[mascara].groupby("director")["otra_columna"].mean()
```

**Puntos clave:**
- `~` es el operador de negación booleana y solo se aplica al **resultado** de `.isin()` (una Serie booleana) — nunca dentro de la lista. `df["col"].isin(~["a", "b"])` lanza `TypeError`, porque `~` no es válido sobre una lista de strings.
- `.isin([...])` sin negar ya excluye automáticamente los `NaN` de la columna filtrada (NaN nunca es igual a nada, ni a otro NaN). Al negar con `~.isin([...])`, ese comportamiento se invierte: los `NaN` de esa columna **sí pasan** el filtro, porque no están en la lista de exclusión.

**Combinación con `.notna()` para excluir también los NaN explícitamente:**

```python
mascara = ~df["director"].isin(["director 1", "director 2", "director 3"]) & df["director"].notna()
resultado = df[mascara].groupby("director")["otra_columna"].mean()
```

> [!NOTE] `.notna()` de una columna vs. `.dropna()` del DataFrame
> `df["columna"].notna()` sólo filtra por esa columna específica. `df.dropna()` sin `subset` es más agresivo: elimina cualquier fila con `NaN` en **cualquier** columna del DataFrame, no solo en la que interesa. Para que ambos sean equivalentes: `df.dropna(subset=["columna"])`.

> [!TIP] En la práctica, el `.notna()` explícito casi nunca hace falta aquí
> Tanto `.isin()` (por el lado negado) como `groupby()` (que descarta por default el grupo `NaN` de la columna de agrupación) hacen que el resultado final de `groupby().mean()` sea el mismo con o sin el `.notna()` extra. Solo importa si se necesita conservar esas filas de `NaN` en el DataFrame para otro propósito antes de agrupar.

**Contexto real:** Práctica personal — variante de exclusión del patrón `.isin()` + máscara (ver [[#isin-where]]), explorando el comportamiento de `NaN` bajo negación booleana.

---

## 🎯 `.isin()` + `.where()` vs. Filtro con Máscara {#isin-where}

**Cuándo:** Para condicionar un `groupby` a un subconjunto específico de categorías — hay dos formas válidas, cada una con un uso distinto.

```python
# Forma A: .where() — conserva TODAS las filas, marca con NaN las que no cumplen
condicion = df["director"].isin(["director 1", "director 2", "director 3"])
df["columna_filtrada"] = df["director"].where(condicion, np.nan)
df.groupby("columna_filtrada")["otra_columna"].mean()

# Forma B: máscara booleana — filtra PRIMERO, nunca genera NaN
mascara = df["director"].isin(["director 1", "director 2", "director 3"])
resultado = df[mascara].groupby("director")["otra_columna"].mean()
```

**Diferencia práctica:**

| Forma | Qué hace | Cuándo usarla |
|---|---|---|
| `.isin()` + `.where()` | Conserva la columna marcada en el DataFrame (con `NaN` para lo excluido) | Cuando se quiere inspeccionar o graficar después qué quedó fuera |
| `.isin()` + máscara (`df[mascara]`) | Filtra antes de agrupar, sin dejar rastro de lo excluido | Cuando solo interesa el resultado agregado final |

> [!NOTE] `groupby` ignora `NaN` por default
> En la Forma A, el groupby resultante va a excluir automáticamente el grupo `NaN` (valores no incluidos en el `.isin()`), así que el resultado numérico final suele coincidir con la Forma B. La diferencia es solo si se necesita conservar la columna intermedia.

**Contexto real:** Práctica personal fuera de Kaggle — patrón de filtrado condicional antes de un `groupby().mean()`.

---

## 🔀 `pd.crosstab()` y `pivot_table()` {#crosstab-pivot}

**Cuándo:** Para cruzar dos columnas categóricas y ver el resultado en formato "ancho" (tipo tabla dinámica de Excel), en vez del formato "largo" apilado que da un `groupby` normal.

```python
# groupby normal -> formato LARGO (apilado)
df.groupby(['Category', 'Region'])['Sales'].sum()
# Category      Region
# Furniture     Central   160317
#               East      206461
# ...

# crosstab -> formato ANCHO, conteo de frecuencias por default
tabla = pd.crosstab(df["Category"], df["Region"])

# crosstab agregando una columna continua (no solo contando)
tabla_ventas = pd.crosstab(
    df["Category"], df["Region"],
    values=df["Sales"], aggfunc="sum"
)

# pivot_table -> más flexible, mismo resultado que crosstab con values/aggfunc
df.pivot_table(
    values='Sales', index='Category', columns='Region', aggfunc='sum'
)
```

**Puntos clave:**
- `crosstab` por default **cuenta frecuencias** — es el equivalente de `groupby([...]).size().unstack()`. Solo agrega valores continuos si se le pasan `values` y `aggfunc` explícitamente.
- Los argumentos de `crosstab` deben ser **columnas del DataFrame** (`df["Category"]`), no strings sueltos (`"Category"`) — pasar el nombre como texto literal cruza las palabras, no los datos, y da una tabla sin sentido.
- `pivot_table` es la versión más flexible y general; `crosstab` es un atajo pensado originalmente para tablas de frecuencia.
- Ninguno de los dos reemplaza a `.transform()`: ambos colapsan el resultado, igual que `.agg()`.

> [!WARNING] Error común: pasar strings en vez de columnas
> `pd.crosstab(["Category"], ["Sales"])` (con corchetes y comillas, sin `df[...]`) no cruza las columnas del dataset — cruza literalmente las palabras "Category" y "Sales", dando una tabla 1x1 sin utilidad. Siempre usar `df["columna"]`.

**Contexto real:** Kaggle Superstore — tabla de contingencia `Category` x `Region` para contar órdenes, y variante con `aggfunc='sum'` sobre `Sales`.

---

## 🔗 Conexiones Estratégicas

- **Índice Maestro:** [[Indice_Maestro]]
- **Herramienta completa:** [[Pandas]]
- **Operación relacionada (window functions SQL):** [[Funnel_y_Cohortes_S12#cohortes-base]]
- **Tabla rápida de equivalencias SQL ↔ Pandas (todas las operaciones, no solo agg/transform):** [[SQL_vs_Python_vs_DAX]]
- **Práctica externa:** Kaggle — Superstore Sales Dataset, previo a ejercicios de window functions en DataLemur/StrataScratch.
