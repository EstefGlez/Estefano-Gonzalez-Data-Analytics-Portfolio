---
tags: [herramienta, sql, orden-ejecucion, fundamentos]
tipo: nota-referencia
herramientas: [sql]
---

# 🧩 SQL — Jerarquía y Orden Real de Ejecución

La mayoría de las reglas de SQL que se sienten "arbitrarias" dejan de serlo en cuanto entiendes esto: **SQL no se ejecuta en el orden en que lo escribes.** Existe un orden lógico interno fijo — casi todas las dudas de "¿por qué esto sí y esto no?" se resuelven ubicando en qué paso de ese orden está cada cláusula.

---

## 📋 Índice

| Sección | Ir a |
|---|---|
| El orden real de ejecución | [[#orden-ejecucion]] |
| Por qué cada regla "rara" existe | [[#reglas-explicadas]] |
| CTEs — por qué se comportan distinto | [[#ctes]] |
| Otras reglas que se sienten arbitrarias | [[#otras-reglas]] |

---

## 🔢 El Orden Real de Ejecución {#orden-ejecucion}

```
1. FROM / JOIN       → arma la tabla combinada (todas las filas posibles)
2. ON                → condición de cada JOIN (se aplica al armar esa tabla)
3. WHERE             → filtra filas individuales de esa tabla combinada
4. GROUP BY          → agrupa lo que sobrevivió al WHERE
5. HAVING            → filtra grupos ya agregados
6. Window Functions  → calculan sobre lo que sobrevivió a HAVING
7. SELECT            → arma las columnas finales (aquí "nacen" los alias)
8. DISTINCT          → quita duplicados del resultado ya armado
9. ORDER BY          → ordena el resultado final
10. LIMIT / OFFSET   → corta el resultado ya ordenado
```

> [!IMPORTANT] Lo escribes en un orden, se ejecuta en otro
> Escribes `SELECT ... FROM ... WHERE ... GROUP BY ... HAVING ... ORDER BY`, pero el motor ejecuta `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY`. `SELECT` está arriba en el texto, pero corre casi al final. Esa desalineación entre "orden escrito" y "orden ejecutado" es la raíz de casi todas las reglas confusas.

---

## 🧠 Por Qué Cada Regla "Rara" Existe {#reglas-explicadas}

### ¿Por qué no puedo usar un alias del SELECT en el WHERE?

```sql
-- ❌ Esto falla
SELECT (precio * cantidad) AS total
FROM ventas
WHERE total > 100;

-- ✅ Hay que repetir la expresión completa
SELECT (precio * cantidad) AS total
FROM ventas
WHERE (precio * cantidad) > 100;
```

`WHERE` es el **paso 3**. `SELECT` (donde nace el alias `total`) es el **paso 7**. Cuando `WHERE` se ejecuta, `total` todavía no existe — por eso hay que repetir la expresión completa, o usar una subquery/CTE (ver más abajo).

### ¿Por qué sí puedo usar el alias en ORDER BY?

```sql
-- ✅ Esto sí funciona
SELECT (precio * cantidad) AS total
FROM ventas
ORDER BY total DESC;
```

`ORDER BY` es el **paso 9** — para cuando le toca correr, `SELECT` (paso 7) ya se ejecutó y `total` ya existe como columna real del resultado.

### ¿Por qué HAVING sí puede usar COUNT()/SUM() pero WHERE no?

```sql
-- ❌ Esto falla
SELECT categoria, COUNT(*)
FROM productos
WHERE COUNT(*) > 5
GROUP BY categoria;

-- ✅ Esto funciona
SELECT categoria, COUNT(*)
FROM productos
GROUP BY categoria
HAVING COUNT(*) > 5;
```

`GROUP BY` es el **paso 4**, `HAVING` es el **paso 5** — los agregados (`COUNT`, `SUM`, etc.) ya están calculados cuando `HAVING` los necesita. `WHERE` es el **paso 3**, antes de que exista cualquier agrupación — no hay nada que agregar todavía.

### ¿Por qué las funciones de ventana "funcionan aunque tengan una agregación adentro"?

```sql
SUM(ingreso) OVER (PARTITION BY categoria)
```

Esto **no** es una agregación anidada dentro de otra en el mismo nivel — son dos pasos distintos, uno después del otro:
1. Si hay `GROUP BY` en la consulta, `SUM(ingreso)` ya se resolvió como parte del **paso 4**.
2. La función de ventana (`OVER(...)`) es el **paso 6**, y opera **sobre el resultado que ya salió de los pasos 1-5** — no al mismo tiempo que ellos.

Por eso puedes tener algo como `AVG(SUM(ventas)) OVER (...)` — el `SUM` ya resolvió su nivel (agrupado), y el `AVG` de la ventana trabaja sobre ese resultado ya calculado, como si fuera una tabla nueva.

> [!WARNING] Consecuencia práctica: no puedes usar una función de ventana dentro de WHERE o GROUP BY
> Las funciones de ventana corren en el **paso 6**, después de `WHERE` (paso 3) y `GROUP BY` (paso 4) — por eso intentar filtrar directo con `WHERE RANK() OVER (...) = 1` da error. Hay que envolver la consulta en una subquery o CTE, y filtrar **afuera**, donde el resultado de la ventana ya existe como columna normal. Ver [[Practica_DataLemur#window-functions]] y [[Practica_DataLemur#supercloud]] para el patrón aplicado.

---

## 🧱 CTEs — Por Qué se Comportan Distinto {#ctes}

```sql
with resumen as (
    SELECT categoria, SUM(ventas) AS total
    FROM productos
    GROUP BY categoria
)
SELECT categoria, total
FROM resumen
WHERE total > 1000;  -- ✅ esto SÍ funciona
```

Un CTE se ejecuta como si fuera una **tabla completamente independiente y ya terminada**, antes de que la consulta principal (la que lo usa) empiece a correr. Para cuando el `WHERE` de la consulta externa se ejecuta, `resumen` ya existe completo, con la columna `total` ya calculada — se comporta como cualquier tabla real, sin las restricciones del orden de ejecución interno de una sola consulta.

> [!NOTE] Por qué esto resuelve el problema del alias
> Dentro de una sola consulta, `total` (alias del SELECT) no existe todavía cuando el `WHERE` de esa misma consulta corre. Pero si mueves ese cálculo a un CTE, y filtras **en la consulta de afuera**, el CTE ya se resolvió por completo — el alias ya es una columna real, no una promesa pendiente.

---

## ⚠️ Otras Reglas que se Sienten Arbitrarias (y no lo son) {#otras-reglas}

### NULL nunca se compara con `=`

```sql
-- ❌ Nunca filtra nada, ni siquiera los que sí son NULL
WHERE columna = NULL

-- ✅ Correcto
WHERE columna IS NULL
```

`NULL` representa "desconocido" — comparar algo desconocido con `=` siempre da como resultado otro "desconocido" (ni verdadero ni falso), nunca `TRUE`. Por eso existe la sintaxis especial `IS NULL` / `IS NOT NULL`.

### JOIN ON vs WHERE — no siempre da lo mismo con LEFT JOIN

```sql
-- Versión A: filtro en el ON
SELECT *
FROM clientes c
LEFT JOIN pedidos p ON c.id = p.cliente_id AND p.estado = 'Completado';

-- Versión B: filtro en el WHERE
SELECT *
FROM clientes c
LEFT JOIN pedidos p ON c.id = p.cliente_id
WHERE p.estado = 'Completado';
```

Con `INNER JOIN` da igual. Con `LEFT JOIN` **no**: el filtro en `ON` (paso 2) decide qué filas de `pedidos` se pegan a cada cliente — los clientes sin pedidos completados igual aparecen, con `NULL` en las columnas de `pedidos`. El filtro en `WHERE` (paso 3) se aplica **después** de que el `LEFT JOIN` ya trajo todo, así que elimina por completo a los clientes sin pedidos completados — convirtiendo, sin querer, tu `LEFT JOIN` en un `INNER JOIN` de facto.

> [!IMPORTANT] Esto es un error muy común y silencioso
> Filtrar condiciones de la tabla derecha en el `WHERE` en vez de en el `ON` es una de las causas más comunes de que un `LEFT JOIN` "no funcione como debería" — el motor no da error, simplemente te devuelve menos filas de las esperadas.

### GROUP BY y ORDER BY sí aceptan alias en PostgreSQL — pero es una extensión, no el estándar

```sql
SELECT categoria, COUNT(*) AS total
FROM productos
GROUP BY categoria
ORDER BY total DESC;  -- funciona en Postgres
```

Aunque por el orden de ejecución `GROUP BY` (paso 4) corre antes que `SELECT` (paso 7), PostgreSQL específicamente permite usar el alias del SELECT en `GROUP BY` y `ORDER BY` como una conveniencia extra — no es parte del estándar SQL, y algunos motores (dependiendo de configuración) no lo permiten. `WHERE` y `HAVING` nunca lo permiten, en ningún motor.

### COUNT(*) vs COUNT(columna) — los NULL se ignoran distinto

```sql
COUNT(*)         -- cuenta TODAS las filas, incluyendo las que tienen NULL en cualquier columna
COUNT(columna)   -- cuenta solo las filas donde ESA columna no es NULL
```

Todas las funciones de agregación (`SUM`, `AVG`, `COUNT(columna)`, `MAX`, `MIN`) **ignoran automáticamente los `NULL`** al calcular — excepto `COUNT(*)`, que cuenta filas sin importar su contenido.

### DISTINCT aplica a la fila completa, no a una sola columna

```sql
SELECT DISTINCT categoria, ciudad
FROM productos;
```

Esto no da "categorías únicas" y "ciudades únicas" por separado — da combinaciones únicas de `(categoria, ciudad)` juntas. Si quieres únicos de una sola columna, esa debe ser la única en el `SELECT`.

---

## 🔗 Conexiones Estratégicas

- **Índice Maestro:** [[Indice_Maestro]]
- **Herramienta base:** [[SQL]]
- **Aplicación práctica (window functions, HAVING, JOINS):** [[Practica_DataLemur]]
- **Traductor SQL ↔ Pandas ↔ DAX:** [[SQL_vs_Python_vs_DAX]]
