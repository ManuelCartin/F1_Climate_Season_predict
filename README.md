### Pipeline de Integración: Resultados de Carrera × Condiciones Meteorológicas (2019–2024)

> **Portfolio project** — Data Engineering | Data Wrangling | Feature Engineering  
> **Stack**: Python · Pandas · NumPy · Scikit-learn · Matplotlib · Seaborn · Jupyter · Google Colab

---

## Resumen

Este proyecto construye un dataset unificado que combina **resultados de carrera de F1** (2019–2024) con **condiciones meteorológicas históricas** de los 24 circuitos del calendario. El objetivo no es predecir ganadores: es demostrar que el clima es una variable causal que afecta el agarre de neumáticos, la estrategia de boxes y el rendimiento del monoplaza — y que cuantificar ese efecto requiere resolver primero un problema de ingeniería de datos no trivial.

El notebook **documenta cada reto técnico encontrado**, las decisiones de diseño tomadas y los patrones aplicados. Es una bitácora de Data Engineer, no un análisis de predicción climática.

---

## Hipótesis

> Las condiciones climáticas (temperatura, precipitación, humedad, viento) son un factor significativo e independiente que influye en la posición final de los pilotos, más allá del efecto de la posición de salida.

Para probarla primero hay que tener un dataset que una ambas fuentes — y esas fuentes no comparten ninguna clave directa.

---

## El Problema de Ingeniería

Tres fuentes de datos heterogéneas que no hablan entre sí:

| Fuente | Clave disponible | Lo que necesitamos |
|---|---|---|
| Resultados F1 (CSV, Kaggle) | Nombre de circuito (`Track`) | Fecha + condición climática |
| Calendario F1 (XLSX, manual) | Nombre de circuito + fecha | Tabla puente entre fuentes |
| Datos meteorológicos (CSV por ciudad) | Ciudad + fecha diaria | Circuito al que pertenece |

**El problema central:** los CSVs de F1 no tienen fecha, y los CSVs de clima no saben que existen circuitos de F1. Construir el puente entre ambos mundos es el trabajo real de este pipeline.

---

## Retos Técnicos y Soluciones

### Reto 1 — Los CSV de resultados no tienen fecha

Los datasets públicos de F1 identifican cada carrera por `Track = "Australia"` pero **no incluyen la fecha del Gran Premio**. Sin fecha no hay join posible con series temporales diarias.

**Solución:** calendario XLSX como tabla puente (`Track → Date`). El merge usa `how='left'` deliberadamente: un `NaN` en `Date` es una señal de alerta visible, no un dato perdido aceptable. Un `inner` join haría desaparecer esas filas en silencio.

```python
df_f1_con_fechas = pd.merge(
    df_f1_season,    # tiene Track, NO tiene Date
    df_calendar,     # tiene Track + Date
    on='Track',
    how='left'       # NaN visible si un circuito no está mapeado
)
```

---

### Reto 2 — Schema drift entre temporadas

Kaggle publicó los CSVs de 2019 a 2024 en distintos momentos. Los nombres de columna no son idénticos entre años. Al concatenar 6 temporadas se pierde trazabilidad de origen.

**Solución:** columna `season` antes de concatenar + `.copy().reset_index(drop=True)` post-concat. El `.copy()` explícito evita `SettingWithCopyWarning`. El `reset_index` produce un índice contiguo limpio que no rompe merges posteriores.

```python
df_season_2019_column['season'] = 2019
# ... para cada temporada

df_unido = pd.concat(lista_de_datasets).copy().reset_index(drop=True)
```

---

### Reto 3 — Los CSV meteorológicos tienen 3 filas de metadatos

Los exportadores de datos meteorológicos incluyen 3 filas de cabecera antes de los datos reales. Leerlos sin configuración produce nombres de columna incorrectos y tipos numéricos como `object`.

**Solución:** `skiprows=3`. La columna `Track` se asigna inmediatamente tras la carga — antes de cualquier transformación — para mantener trazabilidad geográfica desde el origen.

```python
df_weather = pd.read_csv('export-melbourne0.csv', skiprows=3)
df_weather['Track'] = 'Australia'   # trazabilidad inmediata
```

---

### Reto 4 — Ciudad ≠ Circuito: mapeo de claves entre mundos distintos

El CSV meteorológico usa nombres de ciudad (`Ciudad_Base = 'manama'`), F1 usa nombres de Gran Premio (`Track = 'Bahrain'`). No existe ninguna columna compartida. Además, los nombres del exportador vienen en francés y con encoding variable (`'barcelone'`, `'singapour'`, `'forl-'`).

**Solución:** tabla de mapeo `clima ciudad.xlsx` + normalización `.str.lower().str.strip()` + `.map()` sobre índice hash (O(1) por lookup vs O(n) del merge). El `NaN` resultante para claves sin match es explícito y auditable.

```python
# Normalizar ambos lados para absorber diferencias de case y espacios
df_map_city['Ciudad_Clave_Limpia'] = df_map_city['Ciudad del Dataset'].str.lower().str.strip()
df_climate['Ciudad_Clave_Limpia']  = df_climate['Ciudad_Base'].str.lower().str.strip()

# Mapeo O(1) — NaN visible si no hay match
mapa = df_map_city.set_index('Ciudad_Clave_Limpia')['Ciudad del Circuito']
df_climate['Track'] = df_climate['Ciudad_Clave_Limpia'].map(mapa)
```

---

### Reto 5 — El merge silencioso: tipos de fecha inconsistentes

Este es el bug más peligroso del pipeline porque **no produce ningún error**. Pandas ejecuta el merge, retorna un DataFrame con forma correcta, y solo revisando los datos se descubre que todas las columnas climáticas son `NaN`. Causa: ambas columnas de fecha son `object` pero con diferencias de formato o encoding que rompen la igualdad de strings silenciosamente.

**Regla de oro:** nunca hacer merge sobre fechas sin antes verificar `dtypes` y convertir explícitamente con `pd.to_datetime()`.

```python
# Convertir en AMBOS DataFrames antes del merge
df_season_prep['DATE']   = pd.to_datetime(df_season_prep['DATE'],   errors='coerce')
df_unido_climate['DATE'] = pd.to_datetime(df_unido_climate['DATE'], errors='coerce')

# Diagnóstico post-merge: si este número es alto, el join falló
nans = df_resultado['MAX_TEMPERATURE_C'].isna().sum()
print(f'Filas sin clima: {nans}/{len(df_resultado)} ({100*nans/len(df_resultado):.1f}%)')
```

---

### Reto 6 — Duplicados por cobertura temporal superpuesta

Los CSVs de clima cubren múltiples años desde 2009. Al consolidar varias fuentes para el mismo circuito, una combinación `Track + DATE` puede aparecer más de una vez, inflando artificialmente el dataset y cualquier estadística sobre él.

**Patrón:** inspeccionar con `keep=False` para entender el origen *antes* de eliminar.

```python
# Ver las filas duplicadas antes de decidir qué eliminar
df[df.duplicated(subset=['Track', 'DATE'], keep=False)].head(10)

# Eliminar — keep='first' preserva la fila más temprana del concat
df = df.drop_duplicates(subset=['Track', 'DATE']).reset_index(drop=True)
```

---

## Decisiones de Proximidad Geográfica

Los datos meteorológicos provienen de estaciones en ciudades cercanas a los circuitos, no de los circuitos mismos. Esta limitación está documentada y es parte del alcance de la PoC.

| Gran Premio | Ciudad del Circuito | Ciudad en CSV Meteo | Distancia |
|---|---|---|---|
| GP de Emilia Romagna | Imola, Italia | Forlì | ~40 km |
| GP de Japón | Suzuka | Nagoya | ~60 km ← mayor aproximación |
| GP de Qatar | Lusail | Doha | ~20 km |
| GP de Bélgica | Spa-Francorchamps | Durbuy | ~30 km |
| GP de Gran Bretaña | Silverstone | Northampton | ~20 km |
| GP de Baréin | Sakhir | Manama | ~25 km |
| Resto de circuitos | — | Ciudad directa o ~5–15 km | ≤15 km |

> **Nota:** circuitos en zona montañosa (Austria, Spa) tienen mayor variación de temperatura real respecto a la ciudad de referencia que lo que la distancia geográfica sugiere.

---

## Dataset Final

| Característica | Valor |
|---|---|
| Temporadas | 2019 – 2024 |
| Circuitos mapeados | 24 |
| Fuentes meteorológicas | ~25 ciudades |
| Variables climáticas por fila | ~35 |
| Registros (piloto × carrera) | ~1,200 |
| Archivo de salida | `df_all_circuit_2019_2024.csv` |

### Variables clave

| Categoría | Variables | Rol en el modelo |
|---|---|---|
| **Target** | `Position_Numeric`, `Points` | Variable a predecir |
| **Clima** | `PRECIP_TOTAL_DAY_MM`, `MAX_TEMPERATURE_C`, `WINDSPEED_MAX_KMH`, `WEATHER_CODE_*` | Features independientes |
| **Control** | `Starting Grid`, `Driver`, `Team`, `Track` | Aislar el efecto del clima |
| **Contexto** | `Laps`, `Total Time/Gap`, `season` | Variables auxiliares |

---

## Resultados del Análisis Exploratorio

Con el dataset construido se entrenaron dos modelos Random Forest para cuantificar el efecto del clima:

- **Modelo A (Baseline):** `Starting Grid` + `Driver` + `Team` + `Track`
- **Modelo B (+ Clima):** Modelo A + `MAX_TEMPERATURE_C` + `PRECIP_TOTAL_DAY_MM` + `WINDSPEED_MAX_KMH` + `HUMIDITY_MAX_PERCENT` + `CLOUDCOVER_AVG_PERCENT`

Si el Modelo B mejora el MAE respecto al Modelo A, el clima tiene poder predictivo independiente de las variables de control.

---

## Estructura del Directorio

```
f1 climate comparation data engineer/
├── README.md                          ← este archivo
├── POC_F1_weather.ipynb               ← notebook principal (narrativa DE)
│
├── f1 season poc/                     ← CSVs originales de resultados F1
│   ├── formula1_2019season_raceResults.csv
│   ├── formula1_2020season_raceResults.csv
│   ├── formula1_2021season_raceResults.csv
│   ├── Formula1_2022season_raceResults.csv
│   ├── Formula1_2023season_raceResults.csv
│   ├── Formula1_2024season_raceResults.csv
│   └── Formula1_2025Season_RaceResults.csv
│
├── f1 season with date/               ← resultados enriquecidos con fecha (salida etapa 1)
│   ├── df_f1_con_fechas_2019.csv
│   ├── df_f1_con_fechas_2020.csv
│   └── ...
│
├── fuentes meteo/                     ← CSVs meteorológicos por ciudad + consolidados
│   ├── export-melbourne0.csv          ← Australia
│   ├── export-manama0.csv             ← Baréin
│   ├── export-shanghai0.csv           ← China
│   ├── ...                            ← resto de circuitos
│   ├── clima ciudad.xlsx              ← tabla de mapeo ciudad → circuito
│   ├── df_consolidado.csv             ← clima consolidado pre-mapeo
│   └── df_climate.csv                 ← clima con columna Track (salida etapa 2)
│
├── weather track/                     ← muestras de clima por circuito (exploración inicial)
│   ├── df_weather_australia.csv
│   ├── df_weather_bahrein.csv
│   └── df_weather_china.csv
│
├── export-khobar0.csv                 ← fuente alternativa Arabia Saudita
└── climate_global_f1_circuit.csv      ← vista wide: todos los circuitos × fecha
```

---

## Reproducibilidad

### Dependencias

```bash
pip install pandas numpy matplotlib seaborn scikit-learn openpyxl jupyter
```

### Ejecución

El notebook está diseñado para Google Colab. Las rutas de carga usan `/content/sample_data/` — ajustar a la ruta local o subir los archivos al entorno Colab.

```bash
jupyter notebook POC_F1_weather.ipynb
```

Las celdas están organizadas secuencialmente por etapa del pipeline. Cada sección puede auditarse de forma independiente gracias a los archivos intermedios exportados.

---

## Lecciones de Ingeniería

| Aprendizaje | Contexto |
|---|---|
| Los datos nunca vienen listos para el join | El 70% del trabajo fue construir la tabla ciudad→circuito y homologar nombres |
| El merge silencioso es el peor tipo de error | Pandas no lanza excepción si el join produce cero matches — la validación post-merge es obligatoria |
| Los archivos intermedios son aliados | Guardar cada etapa por separado permite auditar el pipeline sin debuggear el sistema completo |
| Trazabilidad desde el origen | Columnas `Track`, `season`, `Ciudad_Base` se agregan en el momento de carga y no se eliminan hasta el paso final |
| `how='left'` como herramienta de diagnóstico | Un `NaN` post-merge es información, no ruido — indica exactamente qué clave no tiene match |

---

## Próximos Pasos

| Fase | Acción | Impacto |
|---|---|---|
| Datos | Reemplazar ciudades aproximadas con datos METAR/NOAA del aeropuerto más cercano | Mayor precisión meteorológica |
| Datos | Clima por hora (no diario) para capturar cambios durante la carrera | Variable target más precisa |
| Datos | Integrar FIA live timing API: Safety Car, VSC, lluvia durante carrera | Variables de condición real |
| Modelo | Validación temporal con `TimeSeriesSplit` — no `train_test_split` aleatorio | Evitar leakage entre temporadas |
| Modelo | Comparar Modelo A (baseline) vs Modelo B (+ clima) con intervalo de confianza | Cuantificar efecto del clima |

---

## Autor

Proyecto de portafolio — Data Engineering & Data Science  
Parte de la colección de proyectos en **análisis de datos aplicado a motorsport**
[manuelcartinh@gmail.com]
