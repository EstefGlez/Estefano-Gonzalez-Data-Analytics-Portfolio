---
tags: [herramienta, pandas, python, indice]
tipo: indice-herramienta
---

# 🐍 Pandas — Índice de Capacidades

Referencia rápida de todo lo que puedes hacer con Pandas en el contexto del bootcamp. Cada entrada enlaza a la nota de operación donde está el código completo.

---

## 🗂️ Capacidades por Categoría

### 📦 Carga y Exploración
| Qué hace | Función clave | Nota |
|---|---|---|
| Cargar un CSV | `pd.read_csv()` | [[Carga_y_Exploracion#cargar-csv]] |
| Cargar múltiples fuentes | `pd.read_csv()` × N | [[Carga_y_Exploracion#cargar-multiples]] |
| Vista preliminar | `.head()` `.tail()` `.sample()` | [[Carga_y_Exploracion#vista-previa]] |
| Dimensiones y tipos de datos | `.shape` `.info()` | [[Carga_y_Exploracion#auditoria-basica]] |
| Auditoría de nulos | `.isna().sum()` `.isna().mean()` | [[Carga_y_Exploracion#auditoria-nulos]] |
| Auditoría de duplicados | `.duplicated().sum()` | [[Carga_y_Exploracion#duplicados]] |

### ⚙️ Transformación y Feature Engineering
| Qué hace | Función clave | Nota |
|---|---|---|
| Estandarizar columnas a snake_case | `re.sub()` + list comprehension | [[Transformacion_y_Feature_Engineering#snake-case]] |
| Convertir texto a fecha | `pd.to_datetime()` | [[Transformacion_y_Feature_Engineering#to-datetime]] |
| Extraer año / mes / día de fecha | `.dt.year` `.dt.month` | [[Transformacion_y_Feature_Engineering#to-datetime]] |
| Limpiar strings con `%`, `,`, `.` | `.str.replace().astype(float)` | [[Transformacion_y_Feature_Engineering#limpiar-strings]] |
| Imputar nulos con mediana | `.replace(-999, mediana)` | [[Transformacion_y_Feature_Engineering#imputar-pandas]] |
| Reemplazar marcadores desconocidos | `.replace("?", pd.NA)` | [[Transformacion_y_Feature_Engineering#imputar-pandas]] |
| Tratar fechas futuras / imposibles | `df.loc[condición] = pd.NaT` | [[Transformacion_y_Feature_Engineering#fechas-anomalas]] |
| Crear variables indicadoras (flags) | `(condición).astype(int)` | [[Transformacion_y_Feature_Engineering#flags]] |
| Segmentar numérico en categorías | `pd.cut()` | [[Transformacion_y_Feature_Engineering#pd-cut]] |
| Convertir unidades | Operación aritmética directa | [[Transformacion_y_Feature_Engineering#conversion-unidades]] |

### 🔗 Joins y Combinación
| Qué hace | Función clave | Nota |
|---|---|---|
| Inner join | `pd.merge(..., how="inner")` | [[Joins_y_Combinacion#merge-inner]] |
| Left join | `pd.merge(..., how="left")` | [[Joins_y_Combinacion#merge-left]] |
| Preseleccionar columnas antes del merge | `df[cols].copy()` | [[Joins_y_Combinacion#preselect]] |

### 📈 Agregación y Reportes
| Qué hace | Función clave | Nota |
|---|---|---|
| Agrupar y agregar | `.groupby().sum()` `.groupby().mean()` | [[Agregacion_y_Reportes#kpis-sheets]] |
| Conteo por categoría | `.value_counts()` `.value_counts(normalize=True)` | [[Carga_y_Exploracion#auditoria-basica]] |
| Estadísticos descriptivos | `.describe()` | [[Carga_y_Exploracion#auditoria-basica]] |
| Cálculo de cohortes desde cero (patrón base) | `.transform('min')` + resta de `Period` + `.groupby().nunique()` | [[Funnel_y_Cohortes_S12#cohortes-base]] |
| `.agg()` vs `.transform()` — colapsa vs. no colapsa | `.groupby().agg()` / `.groupby().transform()` | [[Groupby_Agg_Transform#agg-vs-transform]] |
| Top-N por grupo | `.transform('rank')` + filtro `<= N` | [[Groupby_Agg_Transform#transform-rank]] |
| Fórmulas custom dentro de groupby | `.agg(lambda x: ...)` / `.transform(lambda x: ...)` | [[Groupby_Agg_Transform#lambda]] |
| Filtrar antes de agrupar | `.isin()` + `.where()` / máscara booleana | [[Groupby_Agg_Transform#isin-where]] |
| Tabla de contingencia / tabla dinámica | `pd.crosstab()` / `.pivot_table()` | [[Groupby_Agg_Transform#crosstab-pivot]] |

### 🧪 Análisis Estadístico
| Qué hace | Función clave | Nota |
|---|---|---|
| Matriz de correlación Pearson | `.corr(method="pearson")` | [[Analisis_Estadistico#correlacion-pearson]] |
| Tabla de contingencia | `pd.crosstab()` | [[Analisis_Estadistico#crosstab]] |
| Prueba Chi-cuadrado | `chi2_contingency()` (SciPy) | [[Analisis_Estadistico#chi2]] |

### 📉 Visualización
| Qué hace | Función clave | Nota |
|---|---|---|
| Configuración global de estilo | `sns.set_theme()` | [[Visualizacion#config-global]] |
| Histograma + KDE | `sns.histplot(kde=True)` | [[Visualizacion#histplot]] |
| Histograma segmentado por grupo | `sns.histplot(hue=)` | [[Visualizacion#hue]] |
| Boxplot con media | `sns.boxplot(showmeans=True)` | [[Visualizacion#boxplot]] |
| Scatterplot con burbujas | `sns.scatterplot(size=)` | [[Visualizacion#scatterplot]] |
| Barplot de promedios | `sns.barplot()` | [[Visualizacion#barplot]] |
| Múltiples gráficos en una figura | `plt.subplots(filas, cols)` | [[Visualizacion#subplots]] |
| Heatmap de correlación | `sns.heatmap()` | [[Analisis_Estadistico#heatmap]] |

---

## 📌 Sprints donde se usó Pandas

| Sprint | Proyecto | Operaciones destacadas |
|---|---|---|
| S5 | LADB — Movilidad urbana LATAM | snake_case, to_datetime, merge, groupby, scatterplot con burbujas |
| S7 | ConnectaTel — Segmentación | Sentinels, MAR, imputación con mediana, flags, histplot con hue |
| S8 | NovaRetail+ — Comportamiento | pd.cut, correlación Pearson, heatmap coolwarm |
| S9 | Landing Page — Pruebas A/B | crosstab, chi2_contingency, barplot, interpretación p-value |

---

## 🔗 Conexiones Estratégicas

- **Índice Maestro:** [[Indice_Maestro]]
- **Herramienta relacionada:** [[SQL]] | [[Google_Sheets]]

---

## 📌 Actualización — Pruebas Estadísticas Completas

### scipy.stats
| Qué hace | Función clave | Nota |
|---|---|---|
| Correlación de Spearman | `spearmanr(a, b)` | [[Analisis_Estadistico#spearman]] |
| Correlación de Punto Biserial | `pointbiserialr(binaria, numerica)` | [[Analisis_Estadistico#punto-biserial]] |
| Prueba Chi-cuadrado | `chi2_contingency(tabla)` | [[Analisis_Estadistico#chi2]] |
| V de Cramér (magnitud) | `chi2` + cálculo manual con `np.sqrt` | [[Analisis_Estadistico#cramer]] |
| Prueba Exacta de Fisher | `fisher_exact(tabla_2x2)` | [[Analisis_Estadistico#fisher]] |
| T-test 2 muestras independientes | `ttest_ind(a, b, equal_var=False)` | [[Analisis_Estadistico#ttest]] |
| Levene (homogeneidad varianzas) | `levene(a, b)` | [[Analisis_Estadistico#levene]] |
| Shapiro-Wilk (normalidad) | `shapiro(data)` | [[Analisis_Estadistico#shapiro]] |

### statsmodels
| Qué hace | Función clave | Nota |
|---|---|---|
| Z-test de proporciones (A/B) | `proportions_ztest(count, nobs)` | [[Analisis_Estadistico#ztest]] |

---

## 📌 Actualización S12 — Patrones Nuevos

### 🔀 Merge y Limpieza
| Qué hace | Función clave | Nota |
|---|---|---|
| Merge sin columnas duplicadas (_x/_y) | `catalog.columns.difference(orders.columns)` | [[KPIs_Financieros_Python#merge-seguro]] |
| Pipeline de limpieza en función | `def limpiar_dataframe(df, ...)` | [[Transformacion_y_Feature_Engineering#pipeline-funcion]] |
| Verificación QA post-limpieza | `def verificar_df(df, nombre)` | [[Transformacion_y_Feature_Engineering#verificacion-post]] |

### 📈 Agregación y KPIs
| Qué hace | Función clave | Nota |
|---|---|---|
| Producto más vendido por volumen | `groupby().sum().sort_values(ascending=False)` | [[Agregacion_y_Reportes#top-producto-volumen]] |
| Resumen ejecutivo formateado | `f-string con >15 para alinear` | [[Agregacion_y_Reportes#resumen-ejecutivo]] |
| Costo total fila por fila | `(orders['cantidad'] * orders['costo_unitario']).fillna(0)` | [[KPIs_Financieros_Python#profit]] |

### 🧪 Estadística
| Qué hace | Función clave | Nota |
|---|---|---|
| V de Cramér como función reutilizable | `def cramers_v(df, col_1, col_2)` | [[Analisis_Estadistico#cramer-funcion]] |
| T-test sobre subconjunto filtrado | `df[df['converted']==1]` antes de `ttest_ind` | [[Analisis_Estadistico#ttest-filtrado]] |
| Correlación individual par a par | `df['col_a'].corr(df['col_b'], method='pearson')` | [[Analisis_Estadistico#pearson-par]] |

### 📉 Visualización
| Qué hace | Función clave | Nota |
|---|---|---|
| Subplots 2x2 con scatterplots | `plt.subplots(2, 2)` + `sns.scatterplot(ax=axes[i,j])` | [[Visualizacion#subplots-scatter]] |
| Boxplot comparativo A vs B | `sns.boxplot(x='grupo', y='métrica')` en subplots | [[Visualizacion#boxplot-ab]] |
| Barplot de tasa de conversión | `groupby().mean()` + `sns.barplot()` | [[Visualizacion#barplot-conversion]] |
