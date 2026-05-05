🏎️ F1 Championship Performance & Weather System
📌 Overview

This project was born from a real debate:

Can driver performance be understood just by looking at the championship standings?

While some argue that top drivers alone explain results, this project explores a deeper hypothesis:

External factors like weather conditions can significantly influence race outcomes and season performance.

To validate this, I built a data engineering pipeline and analytical system combining Formula 1 race data with meteorological information.

🧠 Core Idea

This project challenges simplified interpretations of performance by introducing:

Environmental context (weather)
Data-driven validation
Systematic analysis over intuition
🏗️ System Evolution
🔬 Phase 0: Exploratory Experiment

Initial prototype developed in:
➡️ F1 Climate Comparison

Focused on 4 circuits (Spa, Bahrain, Miami, Singapore)
Key challenge: city mapping and data cleaning
Outcome: validated feasibility and revealed scaling challenges
⚙️ Phase 1: Data Engineering Foundation
Integration of 24 race datasets (2023–2025)
Consolidation of meteorological data
Key Challenges
Missing city data
→ Solved through triangulation and nearest-city mapping
Data ingestion inefficiency
→ Automated using Shell scripts and Linux pipelines
🧱 Phase 2: Pipeline Strengthening (In Progress)
Transition from static dataset to scalable pipeline

Planned improvements:

Apache Airflow → workflow orchestration
Apache Kafka → real-time data streams
Expanded data sources (DNFs, race context)
📊 Data Science Layer (Upcoming)
Correlation analysis between weather and performance
Interactive visualizations
Baseline predictive modeling
🤖 AI Engineering Vision
Integration with high-quality weather APIs (NOAA, METAR)
Advanced predictive models for race scenarios
Context-aware performance analysis
⚙️ Tech Stack
Python, Pandas
Linux / Shell scripting
Apache Airflow (planned)
Apache Kafka (planned)
Scikit-learn (planned)
🧩 Data Engineering Highlights
📂 Formula 1 Data
24 CSV files (2023–2025 seasons)
Automated ingestion and transformation
🌦️ Meteorological Data
Source: historique-meteo
City mapping via triangulation approach
Handling incomplete geographic coverage
🎯 Engineering Focus

This project emphasizes:

Data engineering as the backbone of analysis
The importance of contextual variables in predictive systems
Scaling from experiment → pipeline → intelligent system
🧭 Key Insight

Performance is not only a function of skill, but also of context.

This project demonstrates how external variables can reshape interpretations of competitive systems.

---

Author
[Manuel Cartin]

Contact
[manuelcartinh@gmail.com]
