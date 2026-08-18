---
tags: [indice, maestro, arsenal-tecnico]
tipo: indice-maestro
---

# 🗺️ Índice Maestro — Arsenal Técnico

Mapa central de todas las herramientas y operaciones del bootcamp. Punto de entrada para cualquier consulta técnica o para dar contexto a una IA.

---

## ⚙️ Por Tipo de Operación

| Operación | Herramientas | Nota |
|---|---|---|
| 📦 Carga y Exploración | Pandas, SQL, Sheets | [[Carga_y_Exploracion]] |
| 🧹 Limpieza y Normalización | Sheets, Pandas | [[Limpieza_y_Normalizacion]] |
| ⚙️ Transformación y Feature Engineering | Pandas, SQL | [[Transformacion_y_Feature_Engineering]] |
| 🔗 Joins y Combinación de Datasets | Pandas, SQL | [[Joins_y_Combinacion]] |
| 📈 Agregación y Reportes | Sheets (QUERY), SQL, Pandas | [[Agregacion_y_Reportes]] |
| 🧮 Groupby — Agg, Transform, Lambda, Crosstab | Pandas | [[Groupby_Agg_Transform]] |
| 💰 SQL — Métricas Financieras, Embudos y Retención | SQL | [[SQL_Financiero_y_Metricas]] |
| 🧪 Análisis Estadístico y Pruebas de Hipótesis | Pandas, SciPy | [[Analisis_Estadistico]] |
| 📉 Visualización de Datos | Matplotlib, Seaborn | [[Visualizacion]] |
| 🛒 Modelado Analítico en Sheets (S2 Walmart) | Google Sheets | [[Modelado_Analitico_Sheets]] |
| ⭐ Star Schema — Modelado Relacional BI | Power BI | [[Modelado_Star_Schema]] |
| 📊 DAX — Modelado y Cálculos Power BI | Power BI / DAX | [[DAX_Modelado_PowerBI]] |
| 🎨 Visualizaciones y Diseño Avanzado BI | Power BI | [[DAX_Visualizaciones_PowerBI]] |

---

## 🔧 Por Herramienta

| Herramienta | Sprints | Nota |
|---|---|---|
| 📗 Google Sheets | S1, S2 | [[Google_Sheets]] |
| 🐍 Pandas (Python) | S5, S7, S8, S9 | [[Pandas]] |
| 🗄️ SQL (PostgreSQL) | S3, S4 | [[SQL]] |
| 📊 Power BI | S10, S11 | [[Power_BI]] |
| ⚙️ Jupyter en VS Code | S5, S7, S8, S9 | [[Jupyter_VSCode]] |
| 📉 Matplotlib & Seaborn | S5, S7, S8, S9 | [[Matplotlib_Seaborn]] |
| 🔢 NumPy | S5, S7, S8, S9 | [[Numpy]] |
| 🔌 Python + SQL (sqlalchemy) | S12 | [[Python_SQL]] |
| 💹 KPIs Financieros en Python | S12 | [[KPIs_Financieros_Python]] |
| 🌊 Funnel y Cohortes S12 | S12 | [[Funnel_y_Cohortes_S12]] |
| 📐 DAX Avanzado (SUMX, RELATED) | S12 | [[DAX_Avanzado_S12]] |
| 🔌 Python + SQL | S12 | [[Python_SQL]] |
| 🌿 Git & GitHub | Todos | [[Git_GitHub]] |
| 🔄 Traductor SQL ↔ Python | SQL, Pandas, NumPy | [[SQL_vs_Python_Traductor]] |

---

## 🚀 Prompt de Contexto para IAs

> Copia y pega este bloque al inicio de cualquier chat de IA para que entienda tu stack y nivel inmediatamente, sin explicaciones extra.

```
Soy Estefano, estudiante de Análisis de Datos (TripleTen, Sprint actual: [X]).
Stack técnico: Google Sheets, Python (Pandas, NumPy, Matplotlib, Seaborn, SciPy, statsmodels), SQL (PostgreSQL), Power BI.
Entorno: Jupyter Notebooks en VS Code con kernel virtual env (Python 3.14.6).
Control de versiones: Git + GitHub (Portafolio-Sprints-TripleTen).

Estándar de trabajo:
- Código modular con celdas funcionales bien etiquetadas (🧩 Paso X)
- Documentación en Markdown con hipótesis en LaTeX cuando aplica (H₀/H₁, α = 0.05)
- Imputación determinista cuando existe dependencia matemática entre columnas (nunca promedio si hay fórmula exacta)
- Chi-cuadrado siempre con frecuencias absolutas reales, nunca con porcentajes normalizados
- Conclusiones orientadas al negocio, no solo al dato

Herramienta en uso: [INDICAR]
Objetivo de la sesión: [DESCRIBIR]
```

---

## 🗓️ Mapa de Sprints del Bootcamp

| Sprint | Proyecto | Herramienta | Operaciones clave |
|---|---|---|---|
| S1 | VentaExpress Q4 — Limpieza transaccional | Google Sheets | Imputación determinista, NOMPROPIO, QUERY, SPLIT |
| S2 | Walmart — Cuadro de mando de ventas | Google Sheets | BUSCARV, Star Schema, Ventas/m², Participación %, QA pipeline |
| S3 | Análisis de ROI y Márgenes | SQL (PostgreSQL) | LEFT JOIN multi-tabla, COALESCE, Margen%, ROI% |
| S4 | Embudo de conversión y cohortes MercadoLibre | SQL (PostgreSQL) | CTEs, DISTINCT, Retención D7-D28, Cohortes mensuales |
| S5 | Movilidad urbana LATAM — LADB | Python / Pandas | Carga múltiple, snake_case, merge, datetime, groupby, scatterplot |
| S7 | Segmentación ConnectaTel | Python / Pandas | Sentinels, MAR, imputación con mediana, flags, histplot con hue |
| S8 | Comportamiento NovaRetail+ | Python / Pandas + SciPy | pd.cut, Pearson par a par, Spearman, Punto Biserial, V de Cramér (función), subplots 2x2 scatter |
| S9 | Pruebas A/B Landing Page | Python / SciPy + statsmodels | Chi-cuadrado, T-test filtrado, Z-test proporciones, boxplot A/B, barplot conversión |
| S10 | Andes Retail Group | Power BI | 2 vistas, Nivel_Venta, KPIs macro, SCQA, análisis por segmento |
| S11 | Ventas inmobiliarias — BI Comercial | Power BI / DAX | Star Schema, CALENDAR, CALCULATE+ALL, ALLEXCEPT, YTD, MTD, YoY, Cohortes, Eje dual, Semáforo |
| S12 | RappiPlus — Proyecto Final | Python + SQL + Power BI | pd.read_sql, merge seguro, KPIs financieros, funnel SQL, cohortes LEFT JOIN, SUMX+RELATED, drill-through |
