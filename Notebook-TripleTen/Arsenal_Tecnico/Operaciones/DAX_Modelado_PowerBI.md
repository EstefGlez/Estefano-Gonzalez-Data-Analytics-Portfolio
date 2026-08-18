---
tags: [operacion, dax, power-bi, modelado, star-schema, cohortes, sprint-11]
tipo: nota-operacion
herramientas: [power-bi, dax]
---

# 📊 DAX — Modelado y Cálculos en Power BI

Lenguaje de fórmulas de Power BI para crear tablas calculadas, medidas dinámicas y cálculos de inteligencia de tiempo. Es el equivalente a SQL dentro del entorno de Business Intelligence visual.

---

## 📋 Índice de Operaciones

| Operación | Ir a |
|---|---|
| Crear tabla calendario dinámica | [[#tabla-calendario]] |
| Medidas base de agregación | [[#medidas-base]] |
| Participación % con CALCULATE + ALL | [[#participacion]] |
| Acumulados temporales YTD / MTD | [[#time-intelligence]] |
| Crecimiento Year over Year (YoY) | [[#yoy]] |
| Columnas calculadas para cohortes | [[#cohortes]] |
| CALCULATE + ALLEXCEPT explicado | [[#allexcept]] |

---

## 📅 Tabla Calendario Dinámica {#tabla-calendario}

**Cuándo:** Siempre al construir un modelo en Power BI. Es obligatoria para usar inteligencia de tiempo (YTD, MTD, YoY).

```dax
dim_fecha = 
ADDCOLUMNS (
    CALENDAR (
        MIN(hecho_ventas_propiedades[fecha_venta]),
        MAX(hecho_ventas_propiedades[fecha_venta])
    ),
    "Año",           YEAR([Date]),
    "Número de mes", MONTH([Date]),
    "Mes (nombre)",  FORMAT([Date], "MMMM"),
    "Año-Mes",       FORMAT([Date], "YYYY-MM")
)
```

| Función | Qué hace |
|---|---|
| `CALENDAR(inicio, fin)` | Genera fechas consecutivas entre dos fechas |
| `MIN(...[fecha_venta])` | Fecha más antigua del historial — inicio dinámico |
| `MAX(...[fecha_venta])` | Fecha más reciente — fin dinámico |
| `ADDCOLUMNS(tabla, ...)` | Extiende la tabla con columnas calculadas |
| `FORMAT([Date], "MMMM")` | Nombre del mes en texto |
| `FORMAT([Date], "YYYY-MM")` | Formato año-mes para ordenamiento cronológico |

> [!IMPORTANT] Usar MIN/MAX dinámico — nunca fechas fijas
> `CALENDAR(DATE(2023,1,1), DATE(2025,12,31))` es frágil — si llegan datos nuevos fuera de ese rango, la tabla no los cubre. `MIN` y `MAX` se actualizan automáticamente.

**Contexto real:** S11 — tabla `dim_fecha` del modelo de ventas inmobiliarias.

---

## 📐 Medidas Base de Agregación {#medidas-base}

**Cuándo:** Primer paso al construir cualquier modelo. Son los bloques sobre los que se construyen todas las métricas avanzadas.

```dax
// Ingreso total de ventas
Ingreso Total = SUM(hecho_ventas_propiedades[precio_venta])

// Volumen de transacciones
Cantidad de Ventas = COUNT(hecho_ventas_propiedades[id_venta])

// Ticket promedio — DIVIDE protege contra división por cero
Ticket Promedio = DIVIDE([Ingreso Total], [Cantidad de Ventas])

// Comisión retenida por la empresa
Comisión Total = SUM(hecho_ventas_propiedades[monto_comision])
```

> [!TIP] Buenas prácticas DAX
> Siempre especificar `Tabla[Columna]` — nunca solo `[Columna]`. Evita ambigüedades cuando el modelo tiene múltiples tablas.
> Usar `DIVIDE` en lugar de `/` — devuelve BLANK en lugar de error cuando el denominador es 0. Equivale al `NULLIF` de SQL.
> Reutilizar medidas base en medidas compuestas — `Ticket Promedio = DIVIDE([Ingreso Total], [Cantidad de Ventas])` es mejor que `AVERAGE(precio_venta)`.

**Contexto real:** S11 — KPIs del dashboard de ventas inmobiliarias. Formato $  a medidas monetarias desde Herramientas de medición.

---

## 📊 Participación % con CALCULATE + ALL {#participacion}

**Cuándo:** Para calcular qué porcentaje representa cada categoría sobre el total general, ignorando el filtro visual activo.

```dax
// Por tipo de propiedad
Participación ingresos x propiedad = 
DIVIDE (
    [Ingreso Total], 
    CALCULATE ( [Ingreso Total], ALL ( dim_propiedades[tipo_propiedad] ) )
)

// Por canal de venta
Participación ingresos x canal venta = 
DIVIDE (
    [Ingreso Total], 
    CALCULATE ( [Ingreso Total], ALL ( hecho_ventas_propiedades[canal_venta] ) )
)

// Por segmento de cliente
Participación ingresos x segmento cliente = 
DIVIDE (
    [Ingreso Total], 
    CALCULATE ( [Ingreso Total], ALL ( dim_clientes[segmento_comprador] ) )
)
```

**Anatomía del patrón:**

```
DIVIDE(
    [Medida],                    ← numerador: valor filtrado de la categoría actual
    CALCULATE(                   ← modifica el contexto de filtro
        [Medida],
        ALL(tabla[columna])      ← ignora el filtro → devuelve el total general
    )
)
```

> [!NOTE] Equivalencia con SQL
> Es el equivalente DAX de: `valor_categoria / SUM(total) OVER()` con funciones de ventana.

> [!WARNING] Formato obligatorio
> Estas medidas devuelven decimales (0.0 a 1.0). Aplicar formato `%` desde Herramientas de medición → símbolo `%`.

**Uso en el dashboard:** Arrastrar al cajón "Información sobre herramientas" de los gráficos de barras para mostrar la participación como tooltip flotante sin saturar el diseño.

**Contexto real:** S11 — participación de ingresos por tipo de propiedad, canal de venta y segmento de cliente.

---

## ⏱️ Inteligencia de Tiempo — YTD y MTD {#time-intelligence}

**Cuándo:** Para mostrar el progreso acumulado del período actual en dashboards ejecutivos.

```dax
// Acumulado del año en curso (Year to Date)
Ventas Year to Date = TOTALYTD([Ingreso Total], dim_fecha[Date])

// Acumulado del mes en curso (Month to Date)
Ventas Month to Date = TOTALMTD([Ingreso Total], dim_fecha[Date])

// Ingresos del mismo período del año anterior
Ventas del año anterior = 
CALCULATE([Ingreso Total], SAMEPERIODLASTYEAR(dim_fecha[Date]))
```

| Función | Reinicia | Cuándo usarlo |
|---|---|---|
| `TOTALYTD(medida, fechas)` | 1 de enero cada año | Seguimiento de metas anuales |
| `TOTALMTD(medida, fechas)` | 1 de cada mes | Seguimiento operativo mensual |
| `SAMEPERIODLASTYEAR(fechas)` | — | Comparar mismo período año anterior |

> [!IMPORTANT] Requisito obligatorio
> Estas funciones **requieren** la tabla calendario `dim_fecha` con columna de fechas continuas sin gaps. Sin ella no funcionan.

**Contexto real:** S11 — KPIs de acumulado anual y mensual en el dashboard inmobiliario.

---

## 📈 Crecimiento Year over Year (YoY) {#yoy}

**Cuándo:** Para medir si el negocio está creciendo o contrayéndose respecto al mismo período del año anterior. Se visualiza en gráfico de eje dual junto con el Ingreso Total.

```dax
// Primero crear la medida del año anterior
Ventas del año anterior = 
CALCULATE([Ingreso Total], SAMEPERIODLASTYEAR(dim_fecha[Date]))

// Luego calcular el crecimiento porcentual
Crecimiento YoY = 
DIVIDE(
    [Ingreso Total] - [Ventas del año anterior], 
    [Ventas del año anterior]
)
```

**Uso en el dashboard:** Visual de barras + líneas con eje dual:
- Eje Y principal (barras) → `[Ingreso Total]`
- Eje Y secundario (línea) → `[Crecimiento YoY]`

Formato `%` para la medida de crecimiento.

**Contexto real:** S11 — gráfico "Ingreso Total y Crecimiento % YoY por Año-Mes" en la vista Overview Ejecutivo.

---

## 🔁 Columnas Calculadas para Análisis de Cohortes {#cohortes}

**Cuándo:** Para construir una matriz de cohortes que muestra si los clientes vuelven a comprar después de su primera transacción.

> [!IMPORTANT] Columnas calculadas, NO medidas
> Estos cálculos van en `hecho_ventas_propiedades` como **columnas calculadas** (clic derecho sobre la tabla → Nueva columna), no como medidas. Deben crearse en el lienzo principal de Power BI, no en Power Query.

```dax
// 1. Fecha de la primera compra de cada cliente
// ALLEXCEPT aísla las filas del cliente actual → MIN encuentra su mínimo
Primera compra por cliente = 
CALCULATE (
    MIN ( hecho_ventas_propiedades[fecha_venta] ),
    ALLEXCEPT ( hecho_ventas_propiedades, hecho_ventas_propiedades[id_cliente] )
)

// 2. Mes de la primera compra → fila de la matriz de cohortes
Mes Cohorte = FORMAT( [Primera compra por cliente], "YYYY-MM" )

// 3. Mes de cada transacción individual → columna de la matriz
Mes Venta = FORMAT( hecho_ventas_propiedades[fecha_venta], "YYYY-MM" )
```

**Cómo funciona `ALLEXCEPT` en columnas calculadas:**

```
Power BI recorre la tabla fila por fila.
En cada fila mira el id_cliente (ej. CUST001).
ALLEXCEPT reduce la tabla a solo las filas de ese cliente.
MIN encuentra la fecha más antigua de ese grupo.
Ese resultado se repite en TODAS las filas del mismo cliente.

Resultado:
id_venta | id_cliente | fecha_venta | Primera compra
SALE001  | CUST001    | 01/01/2024  | 01/01/2024  ← mismo valor
SALE002  | CUST001    | 15/03/2024  | 01/01/2024  ← repetido
SALE003  | CUST001    | 20/07/2024  | 01/01/2024  ← repetido
```

**Construcción de la matriz en Power BI:**
- Visual: **Matriz**
- Filas: `Mes Cohorte`
- Columnas: `Mes Venta`
- Valores: `Cantidad de Ventas`
- Formato condicional: Color de fondo → escala semáforo (verde alto, rojo bajo)

**Contexto real:** S11 — matriz de retención de clientes inmobiliarios 2023-2024. Cohortes de 2023 mostraron mayor salud transaccional (zonas naranja/amarillo) vs. caída acelerada en cohortes 2024.

---

## 🧠 CALCULATE + ALLEXCEPT — Explicado {#allexcept}

**La analogía:**
- `ALLEXCEPT` = orden de cateo firmada por un juez (especifica qué aislar)
- `CALCULATE` = oficial de policía que ejecuta la orden

Sin `CALCULATE`, `ALLEXCEPT` solo genera una tabla en memoria — no puede ejecutarse solo. Power BI lanzará error: *"La función ALLEXCEPT solo se puede usar como argumento de filtro en CALCULATE"*.

**Diferencia clave con `ALL`:**

| Función | Qué ignora |
|---|---|
| `ALL(tabla[columna])` | Solo el filtro de esa columna específica |
| `ALLEXCEPT(tabla, tabla[columna])` | Todos los filtros EXCEPTO el de esa columna |

**Dónde viven las columnas calculadas vs. Power Query:**

```
[Origen de datos] → [Power Query (M)] → [Modelo de datos (DAX)]
```
Power Query está antes en la cadena — nunca verá las columnas calculadas de DAX. Las columnas DAX solo existen en el modelo de datos (panel de Datos de la interfaz principal).

---

## 🔗 Conexiones Estratégicas

- **Índice Maestro:** [[Indice_Maestro]]
- **Herramienta:** [[Power_BI]]
- **Concepto base:** [[Modelado_Star_Schema]]
- **Visualizaciones avanzadas:** [[DAX_Visualizaciones_PowerBI]]
- **Traductor SQL/Pandas ↔ DAX (ALLEXCEPT ≈ PARTITION BY):** [[SQL_vs_Python_vs_DAX#sql-vs-dax]]
- **Sprint de referencia:** S11 — Ventas de propiedades inmobiliarias (Grupo Andes)
