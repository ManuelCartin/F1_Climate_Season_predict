# F1 Championship Predictor: Performance and Weather Analysis 🏎️ 📊 🌦️
## Project Description
This project is an exploration of data engineering and science aimed at analyzing Formula 1 driver performance and its relationship with weather conditions. The central hypothesis is inspired by the debate surrounding the competitiveness of drivers like Max Verstappen and Oscar Piastri, and seeks to determine how factors like weather can influence a season's results. The project is based on data from the 2023-2025 seasons, a relevant period that avoids the issues of significant regulation changes.

## Project Phases
### Phase 1: Data Engineering - Creation of the Base Dataset
In this phase, the main challenge was data acquisition and preprocessing. The F1 data was obtained from 24 CSV files, while the weather data came from the historique-meteo platform.

Challenges and Solutions:

Climate Data Acquisition: The challenge was that not all cities in the circuits were available in the climate source. This was resolved through triangulation and mapping of nearby cities to obtain the most accurate climate information possible.

Load Optimization: Shell scripts and Linux commands were used to optimize the loading of the 24 CSV files, automate the creation of key columns (gp, city), and prepare the data for subsequent joining.

### Phase 2: Advanced Data Engineering - Pipeline Strengthening
The second phase focuses on optimizing and strengthening the data pipelines. The goal is to create a more complete and reliable dataset.

Technologies: Technologies such as Apache Airflow for orchestration, Kafka for managing real-time data streams, and Shell scripts for ETL (Extract, Transform, Load) tasks are planned to be implemented.

Data Sources: New sources, such as the official Formula 1 website and motorsport.es, will be incorporated to enrich the dataset with more detailed and hidden information, such as the specific reasons for DNFs (Did Not Finishes).

Next Steps (Roadmap)
### Phase 3: Data Science - Analysis and Visualization

Once the dataset is robust, data analysis will begin.

Interactive visualizations will be created to understand the correlation between driver performance and weather conditions.

A basic predictive model will be built to validate the project's initial hypothesis.

### Phase 4: AI Engineering - Future Plan

This is the final, long-term phase. Experience gained from certifications and engineering studies in Data Science and AI Engineering will be used.

More accurate and reliable weather data sources will be sought through the NOAA and METAR APIs.

A more advanced artificial intelligence model will be developed for more accurate prediction of pilot performance in different scenarios.

### Technologies and Tools Used
Phase 1: Python, Pandas, Shell/Bash, Linux.

Phase 2: Shell/Bash, Apache Airflow, Apache Kafka.

Phase 3: Python (Pandas, Matplotlib, Seaborn), Jupyter Notebook, Scikit-learn (for the basic model).

Phase 4 (Future): NOAA and METAR APIs, advanced machine learning libraries (e.g., TensorFlow, PyTorch).

---
### **Fuentes de Datos y Preprocesamiento**

Este proyecto utiliza datos de dos fuentes principales que fueron procesadas y preparadas para su análisis:

* **Formula 1 Data:**
* **Source:** 24 CSV files from the 2023-2025 seasons.
* **Challenge and Solution:** Manually uploading each file was inefficient. Shell-based scripts were created to automate the reading process, add columns (`gp`, `city`), and consolidate the data into a single dataset using Python.

* **Meteorological Data:**
* **Source:** [historique-meteo](https://www.historique-meteo.net/)
* **Challenge and Solution:** The database did not include information for all cities with circuits. A mapping and triangulation process was implemented, choosing the closest city from a set of three possible ones to ensure the best possible climatic approximation.

---

Author
[Manuel Cartin]

Contact
[manuelcartinh@gmail.com]
