## ⛏️ Mining Data Lakehouse (Databricks + Azure)

End-to-end data platform for underground polymetallic mining operations. 
This project designs a modern Data Lakehouse architecture based on a real-world mining scenario, integrating:

- Mine production systems
- Geological data (ore grades)
- Haulage and transport (IoT sensors)
- Plant processing
- Contractor costing and valuation

The solution follows a **Medallion Architecture (Bronze → Silver → Gold)** using Azure Databricks to transform raw operational data into business-ready insights.

[![Azure](https://img.shields.io/badge/Azure-Databricks-FF3621?logo=databricks&logoColor=white)](https://databricks.com)
[![Delta Lake](https://img.shields.io/badge/Delta-Lake-003366?logo=apachespark&logoColor=white)](https://delta.io)
[![Kafka](https://img.shields.io/badge/Apache-Kafka-231F20?logo=apachekafka&logoColor=white)](https://kafka.apache.org)
[![Airflow](https://img.shields.io/badge/Apache-Airflow-017CEE?logo=apacheairflow&logoColor=white)](https://airflow.apache.org)
[![Power BI](https://img.shields.io/badge/Power-BI-F2C811?logo=powerbi&logoColor=black)](https://powerbi.microsoft.com)

## 🏢 Real Company · Real Operation

**Pan American Silver Corp** (NASDAQ: PAAS) · **Huarón Mine, Pasco, Peru.**
Underground silver-zinc-lead-copper mine · 2,500 TPD · 3.33 MOz Ag produced (2025)

> Architecture proposal based on public data and first-hand operational experience in underground polymetallic mining. Not Pan American Silver's official internal system.

## 💡 Business Context

This project is inspired by real experience working in underground mining operations, where I developed:
- Mine production systems
- Geological data integration
- Contractor valuation systems

The goal is to simulate how modern data platforms can integrate these domains to answer key business questions such as:
- Cost per ton
- Ore grade quality by zone
- Contractor performance
- Production efficiency


## 🏗️ Architecture Diagram

###  Executive Summary — Stack & Tools
![Executive Summary](architecture/huaron_resumen_ejecutivo.png)

### Medallion Architecture (Bronze → Silver → Gold)
![Medallion Architecture](architecture/huaron_medallion_v4OK.png)

> Note: Data volumes are estimated based on typical underground mining operations.

## 🧩 The Hard Problem

**Multi-latency grade reconciliation** — joining three sources with 
radically different update frequencies:

| Source | Frequency |
|---|---|
| Truck production stream (IoT) | Every 30 seconds |
| Lab grade results (LIMS) | Every 8 hours |
| Geological block model | Weekly |

This **stream-to-static join pattern** is one of the most technically 
demanding problems in industrial Data Engineering — and the most 
critical KPI in mining operations.

It requires combining real-time IoT ingestion, batch lab processing 
and end-to-end contractor cost calculation into a single reconciled 
pipeline — built on Azure Databricks Structured Streaming + Delta Lake.


## ⚡ What this pipeline does

| Layer | Tables | Key transformation |
|---|---|---|
| **Bronze** | `viajes_camion_raw` · `geologia_muestras_raw` · `produccion_mina_raw` · `planta_raw` · `contratas_raw` | Raw · immutable · NI 43-101 audit trail |
| **Silver** | `viajes_camion` · `geologia_leyes` · `produccion_turno` · `planta_procesamiento` · `contratas_validadas` | Validated · grade reconciliation · VPT · contractor costing |
| **Gold** | `KPI Producción` · `KPI Ley Mineral` · `KPI Recuperación` · `KPI Transporte` · `KPI Valorización` | Business KPIs · daily/weekly/monthly aggregates |


## 🛠️ Tech Stack

```
Sources          →  Ingestion           →  Processing         →  Consumption
─────────────────────────────────────────────────────────────────────────────
IoT Sensors         Azure Event Hubs       Azure Databricks      Power BI
SCADA / PI          Apache Kafka           Delta Lake            Stream Analytics
ERP SAP             Debezium CDC           Apache Spark          MLflow
LIMS Lab            Apache Airflow         Unity Catalog         Teams / SMS
Block Model         Azure Data Factory     Azure ADLS Gen2       Databricks SQL
```

This stream-to-static join pattern is one of the most technically demanding problems in industrial Data Engineering — and the most valuable KPI in mining.

---
## 👩‍💻 About the Author

**Esmeralda Pimentel** — Business-driven Data Engineer · fluent in both mining operations and modern data stack

With **8+ years designing and implementing operational systems for underground and open-pit mining** across Peru's most demanding operations, I bring a rare combination: deep mining domain knowledge + modern Data Engineering skills.

### Mining Systems Built from Scratch

| Company | Operation | Mineral | System Developed |
|---|---|---|---|
| **Pan American Silver** | Morococha · 4,300 masl · Underground | Ag · Cu · Pb · Zn | **PAS-SIM** — Mine Operation Control System (production & geological grade dilution) · **PAS-VAL** — Contractor Valuation System |
| **Hochschild Mining** | Lima · Underground | Au · Ag (Doré + concentrate) | **SIO** — Mine Information System (reserves, performance & operational management modules) |
| **IntiGold Mining** | Arequipa · Open-pit | Au | **PRD** — Mine Production Management System · **VAL** — Contractor Costing System (feasibility) |
| **Minera Los Quenuales** | Iscaycruz · Underground | Pb · Zn | **SICEM** — Mine Equipment & Plant Control System |

### What this means for Data Engineering

Every pipeline in this project — IoT ingestion, grade reconciliation, contractor costing — is modeled after systems I personally designed and implemented in real mines. The data problems are not academic exercises. They are problems I solved operationally, now rebuilt with a modern cloud-native stack.

> *"I don't model mining data. I've lived it at 4,300 meters above sea level, on a 14×7 rotation."*

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Esmeralda%20Mendoza-0A66C2?logo=linkedin&logoColor=white)](https://linkedin.com/in/tu-perfil)
[![GitHub](https://img.shields.io/badge/GitHub-lali2105-181717?logo=github&logoColor=white)](https://github.com/lali2105)

---
*AI Data Engineer Bootcamp · DataHackers Academy · 2026*

