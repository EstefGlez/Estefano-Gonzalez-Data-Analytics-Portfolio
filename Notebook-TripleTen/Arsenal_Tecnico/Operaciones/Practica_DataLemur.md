---
tags: [operacion, sql, practica, datalemur, entrevistas]
tipo: nota-operacion
herramientas: [sql]
---

# 🍋 Práctica SQL — DataLemur (Ejercicios Resueltos)

Ejercicios de entrevista resueltos en DataLemur, organizados por patrón/concepto en vez de por orden cronológico — para poder buscar "¿cómo resolví algo con HAVING?" sin tener que recordar en qué ejercicio específico fue.

---

## 📋 Índice

| Patrón | Ir a |
|---|---|
| GROUP BY + agregaciones básicas | [[#group-by-basico]] |
| Operaciones aritméticas simples (sin agrupación) | [[#aritmetica-simple]] |
| HAVING (filtrar después de agrupar) | [[#having]] |
| COUNT DISTINCT | [[#count-distinct]] |
| CASE WHEN — categorización y conteo condicional | [[#case-when]] |
| JOINS | [[#joins]] |
| Fechas — rangos y diferencias | [[#fechas]] |
| Window Functions — ranking y top-N por grupo | [[#window-functions]] |
| Window Functions — patrón "cumple con TODAS las categorías" | [[#supercloud]] |

---

## 📊 GROUP BY + Agregaciones Básicas {#group-by-basico}

**Patrón:** agrupar por una dimensión y sacar min/max/count — la base de todo lo demás.

```sql
-- Precio de apertura más bajo histórico, por cada stock
SELECT
    ticker,
    MIN(open) AS mini
FROM stock_prices
GROUP BY ticker
ORDER BY mini DESC;

-- Cuántos candidatos tienen cada skill
SELECT
    skill,
    COUNT(candidate_id) AS cantidad
FROM candidates
GROUP BY skill
ORDER BY cantidad DESC;

-- Diferencia entre mes con más y menos tarjetas emitidas, por tarjeta
SELECT
    card_name,
    MAX(issued_amount) - MIN(issued_amount) AS diferencia
FROM monthly_cards_issued
GROUP BY card_name
ORDER BY diferencia DESC;
```

> [!NOTE] Regla de siempre
> Toda columna en el `SELECT` que no esté dentro de una función de agregación debe estar en el `GROUP BY` — o PostgreSQL tira error.

**Caso especial — cuando la agrupación es fácil de pasar por alto:**

```sql
-- Top 3 drogas más rentables (Profit = Total Sales - COGS)
SELECT
    drug,
    SUM(total_sales - cogs) AS total_profit
FROM pharmacy_sales
GROUP BY drug
ORDER BY total_profit DESC
LIMIT 3;
```

> [!WARNING] Por qué este SUM + GROUP BY es necesario aquí
> Si `drug` puede tener más de un `product_id` (ej. distintas presentaciones del mismo medicamento), calcular `(total_sales - cogs)` fila por fila **sin** `SUM`+`GROUP BY` da la ganancia de cada producto individual, no la ganancia total de la droga — el resultado sería incorrecto si una misma droga tiene varias filas. En un dataset donde cada droga aparece en una sola fila, `SELECT drug, (total_sales - cogs) AS profit ... ORDER BY profit DESC LIMIT 3` (sin agregación) también funcionaría — pero `SUM`+`GROUP BY` es la versión robusta que funciona sin importar cuántas filas tenga cada droga, así que es la que conviene memorizar como patrón por default.

---

## 🧮 Operaciones Aritméticas Simples (sin agrupación) {#aritmetica-simple}

```sql
-- Costo por unidad de cada droga de un fabricante específico, redondeado hacia arriba
SELECT
    drug,
    CEIL(total_sales / units_sold) AS unit_cost
FROM pharmacy_sales
WHERE manufacturer = 'Merck'
ORDER BY unit_cost;
```

> [!NOTE] CEIL vs ROUND
> `CEIL()` siempre redondea **hacia arriba** (2.01 → 3), a diferencia de `ROUND()` que redondea al más cercano. Úsalo cuando el enunciado dice explícitamente "redondeado hacia arriba" o "al siguiente entero" — usar `ROUND` ahí sería un error sutil que da resultados distintos solo en los casos límite.

```sql
-- Piezas que empezaron el ensamblaje pero no lo han terminado (sin fecha de fin)
SELECT
    part,
    assembly_step
FROM parts_assembly
WHERE finish_date IS NULL;
```

> [!NOTE] IS NULL, no = NULL
> `NULL` no se compara con `=` porque `NULL` representa "desconocido", y comparar algo desconocido con `=` siempre da `NULL` (ni verdadero ni falso) — nunca filtra nada. Por eso existe la sintaxis especial `IS NULL` / `IS NOT NULL`.

---

## 🔍 HAVING — Filtrar Después de Agrupar {#having}

**Patrón:** cuando el filtro depende de un resultado ya agregado (no se puede usar `WHERE`, porque `WHERE` filtra antes de agrupar).

```sql
-- Stocks cuyo precio de apertura SIEMPRE fue mayor a $100
SELECT
    ticker,
    MIN(open)
FROM stock_prices
GROUP BY ticker
HAVING MIN(open) > 100;

-- Candidatos con más de 2 skills
SELECT candidate_id
FROM candidates
GROUP BY candidate_id
HAVING COUNT(candidate_id) > 2;

-- Usuarios que publicaron 2+ veces en 2021, con días entre su primer y último post
SELECT
    user_id,
    DATE_PART('day', MAX(post_date) - MIN(post_date)) AS days_between
FROM posts
WHERE post_date BETWEEN '2021-01-01' AND '2021-12-31'
GROUP BY user_id
HAVING COUNT(post_id) > 1;

-- Páginas con exactamente 0 likes (usando LEFT JOIN + HAVING sobre el conteo)
SELECT
    p.page_id
FROM pages p
LEFT JOIN page_likes pl ON p.page_id = pl.page_id
GROUP BY p.page_id
HAVING COUNT(pl.page_id) = 0
ORDER BY p.page_id ASC;
```

> [!IMPORTANT] WHERE vs HAVING
> `WHERE` filtra filas **antes** de agrupar (no puede usar `COUNT`, `SUM`, etc.). `HAVING` filtra grupos **después** de agrupar (sí puede usar funciones de agregación). Por eso "candidatos con más de 2 skills" necesita `HAVING COUNT(...) > 2`, no `WHERE`.

> [!NOTE] Truco del "0 likes" con LEFT JOIN
> Para encontrar registros **sin** correspondencia en otra tabla (páginas sin ningún like), se usa `LEFT JOIN` + `HAVING COUNT(columna_de_la_tabla_derecha) = 0` — el `LEFT JOIN` conserva las páginas sin match (con nulls), y el `COUNT` de la columna de la derecha da 0 cuando no hubo match.

---

## 🔢 COUNT DISTINCT {#count-distinct}

```sql
-- Productos únicos vendidos por categoría
SELECT
    category,
    COUNT(DISTINCT product)
FROM product_spend
GROUP BY category;
```

> [!NOTE] Diferencia con COUNT normal
> `COUNT(product)` cuenta cada transacción; `COUNT(DISTINCT product)` cuenta cuántos productos **diferentes** hay, sin importar cuántas veces se repite cada uno.

---

## 🔀 CASE WHEN — Categorización y Conteo Condicional {#case-when}

**Patrón A — crear una columna categórica:**
```sql
SELECT
    actor,
    character,
    platform,
    avg_likes,
    CASE
        WHEN avg_likes >= 15000 THEN 'Super Likes'
        WHEN avg_likes BETWEEN 5000 AND 14999 THEN 'Good Likes'
        WHEN avg_likes < 5000 THEN 'Low Likes'
    END AS categoria
FROM marvel_avengers
ORDER BY avg_likes DESC;
```

**Patrón B — conteo condicional (pivotear categorías a columnas):**
```sql
-- Total de vistas por laptop vs. (tablet + phone combinados)
SELECT
    SUM(CASE WHEN device_type = 'laptop' THEN 1 ELSE 0 END) AS laptop_views,
    SUM(CASE WHEN device_type IN ('tablet', 'phone') THEN 1 ELSE 0 END) AS mobile_views
FROM viewership;

-- Total de órdenes completadas por ciudad (cruzando con JOIN)
SELECT
    u.city,
    SUM(CASE WHEN t.status = 'Completed' THEN 1 ELSE 0 END) AS total_orders
FROM users u
LEFT JOIN trades t ON u.user_id = t.user_id
GROUP BY u.city
ORDER BY total_orders DESC
LIMIT 3;
```

> [!IMPORTANT] SUM(CASE WHEN...) es el patrón para "contar con condición" agrupado
> Cuando necesitas contar cuántas filas cumplen una condición específica, **dentro de un GROUP BY existente**, `SUM(CASE WHEN condicion THEN 1 ELSE 0 END)` es el patrón estándar — equivalente a un `COUNT(DISTINCT CASE WHEN...)` cuando no hay riesgo de duplicados, y es la base de los patrones de funnel/cohortes en [[Funnel_y_Cohortes_S12]].

---

## 🔗 JOINS {#joins}

```sql
-- FULL JOIN — conserva filas de ambas tablas, con o sin match
SELECT *
FROM trades t
FULL JOIN users u ON t.user_id = u.user_id;

-- LEFT JOIN + agregación (top 3 ciudades con más órdenes completadas)
SELECT
    u.city,
    SUM(CASE WHEN t.status = 'Completed' THEN 1 ELSE 0 END) AS total_orders
FROM users u
LEFT JOIN trades t ON u.user_id = t.user_id
GROUP BY u.city
ORDER BY total_orders DESC
LIMIT 3;
```

> [!NOTE] Cuándo usar FULL JOIN vs LEFT JOIN
> `LEFT JOIN` conserva **todas** las filas de la tabla izquierda, aunque no tengan match. `FULL JOIN` conserva **todas** las filas de ambas tablas, aunque no tengan match en ninguna dirección. Si la pregunta pide "todo lo de la tabla A" → `LEFT JOIN`. Si pide "todo de ambas, incluyendo lo que no tiene relación en ningún lado" → `FULL JOIN`.

---

## 📅 Fechas — Rangos y Diferencias {#fechas}

```sql
-- Filtrar por rango de fechas + diferencia en días entre primer/último evento
SELECT
    user_id,
    DATE_PART('day', MAX(post_date) - MIN(post_date)) AS days_between
FROM posts
WHERE post_date BETWEEN '2021-01-01' AND '2021-12-31'
GROUP BY user_id
HAVING COUNT(post_id) > 1;
```

> [!NOTE] DATE_PART('day', fecha1 - fecha2)
> Restar dos fechas/timestamps en PostgreSQL da un intervalo; `DATE_PART('day', ...)` extrae ese intervalo como número de días. Distinto del patrón de cohortes donde se usa `DATE_PART('year', ...)` y `DATE_PART('month', ...)` por separado para convertir a meses — aquí no hace falta esa conversión porque el resultado ya se quiere en días. Ver [[SQL_vs_Python_vs_DAX#sql-vs-pandas-cohortes]] para el caso de meses.

---

## 🪟 Window Functions — Ranking y Top-N por Grupo {#window-functions}

**El patrón más preguntado en entrevistas: "el top N de cada categoría".**

```sql
-- Top 1 artista por revenue-per-member, dentro de cada género
with primero as (
    SELECT
        artist_name,
        genre,
        concert_revenue,
        number_of_members,
        (concert_revenue / number_of_members) AS revenue_per_member,
        RANK() OVER (
            PARTITION BY genre
            ORDER BY (concert_revenue / number_of_members) DESC
        ) AS ranked
    FROM concerts
),
segundo AS (
    SELECT artist_name, concert_revenue, genre, number_of_members, revenue_per_member, ranked
    FROM primero
    WHERE ranked = 1
)
SELECT artist_name, concert_revenue, genre, number_of_members, revenue_per_member
FROM segundo
ORDER BY revenue_per_member DESC;
```

> [!NOTE] Plantilla mental para "top N por grupo"
> 1. CTE que calcula el valor a rankear + `RANK() OVER (PARTITION BY categoria ORDER BY valor DESC)`.
> 2. Segundo CTE (o el SELECT final) que filtra `WHERE ranked <= N`.
> 3. Ordenar el resultado final si el enunciado lo pide explícitamente.
> Mismo patrón exacto que usamos para cohortes con `MIN() OVER (PARTITION BY usuario_id)` — ver [[SQL_vs_Python_vs_DAX#sql-vs-dax]] para la comparación con `ALLEXCEPT` en DAX.

---

## 🌩️ Window Functions — Patrón "Cumple con TODAS las Categorías" {#supercloud}

**Caso más avanzado:** encontrar clientes que compraron de **cada** categoría existente (no solo "al menos una").

```sql
SELECT customer
FROM (
    SELECT
        c.customer_id AS customer,
        COUNT(DISTINCT p.product_category) AS cuantas_categorias,
        MAX(COUNT(DISTINCT p.product_category)) OVER () AS max_categorias
    FROM customer_contracts c
    JOIN products p ON c.product_id = p.product_id
    GROUP BY customer
) AS t
WHERE cuantas_categorias = max_categorias;
```

**Cómo se arma el razonamiento:**
1. Por cliente, cuenta cuántas categorías distintas compró (`COUNT(DISTINCT categoria)`, con `GROUP BY`).
2. Con `MAX(...) OVER ()` (sin `PARTITION BY` — ventana global), encuentra cuál es el número más alto de categorías que compró **cualquier** cliente — ese número representa "compró de todas las categorías que existen".
3. Filtra en la subquery externa donde el conteo del cliente sea igual a ese máximo.

> [!IMPORTANT] Por qué se necesita subquery aquí
> No se puede poner `HAVING cuantas_categorias = MAX(...) OVER ()` directo, porque `HAVING` no puede combinarse con funciones de ventana en el mismo nivel de la consulta — por eso el resultado agrupado se envuelve en una subquery, y el filtro final se aplica **afuera**, donde la columna de la ventana ya existe como un valor normal.

### Versión alternativa — CTE + subquery escalar (más simple)

```sql
with primero as (
    SELECT
        c.customer_id AS customer,
        COUNT(DISTINCT p.product_category) AS cuantos
    FROM customer_contracts c
    JOIN products p ON c.product_id = p.product_id
    GROUP BY customer
)
SELECT customer
FROM primero
WHERE cuantos = (SELECT MAX(cuantos) FROM primero);
```

**Cómo se arma:**
1. El CTE `primero` hace exactamente lo mismo que antes: cuenta categorías distintas por cliente.
2. La subquery `(SELECT MAX(cuantos) FROM primero)` es una **subquery escalar** — devuelve un solo número (el máximo de categorías entre todos los clientes), y ese número se puede usar directo en un `WHERE columna = (subquery)`, sin necesitar función de ventana ni envolver todo en otra subquery de tabla.

> [!NOTE] ¿Cuál versión usar?
> Ambas dan el mismo resultado. La versión con `MAX() OVER()` evita tener que "leer" el CTE dos veces (una vez para agrupar, otra para la subquery), lo cual puede ser más eficiente en tablas grandes — pero la versión con subquery escalar es más fácil de leer y razonar en una entrevista, porque separa claramente "calcular por cliente" de "comparar contra el máximo global" en dos pasos distintos. Cualquiera de las dos es una respuesta válida y demuestra el mismo entendimiento del problema.

---

## 🔗 Conexiones Estratégicas

- **Índice Maestro:** [[Indice_Maestro]]
- **Herramienta base:** [[SQL]]
- **Traductor SQL ↔ Pandas ↔ DAX:** [[SQL_vs_Python_vs_DAX]]
- **Patrón de window functions aplicado a cohortes:** [[Funnel_y_Cohortes_S12#cohortes-base]]
- **Patrón de conteo condicional aplicado a funnels:** [[Funnel_y_Cohortes_S12#cohortes-sql]]
