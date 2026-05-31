---
title: "3-Week Databricks Learning Plan for Data Engineers"
audience: "Data Engineers moving into Databricks AI / GenAI work"
duration: "3 weeks"
estimated_time: "5–7 hours per week"
created: "2026-05-31"
---

# 3-Week Databricks Learning Plan for Data Engineers

This plan is designed for a data engineer who wants to move from **Databricks for data pipelines** toward **Databricks for AI / GenAI workloads**.

## Learning Stack

Focus on this stack:

**Lakehouse → Delta Lake → Spark / PySpark → Auto Loader → Lakeflow / Declarative Pipelines → Unity Catalog → Workflows → Databricks SQL → Mosaic AI / Vector Search / RAG**

## Table of Contents

- [Learning Stack](#learning-stack)
- [Best Official Learning Resources](#best-official-learning-resources)
- [Best YouTube / Conference-Style Videos](#best-youtube--conference-style-videos)
- [Best eBooks / Deeper Reading](#best-ebooks--deeper-reading)
- [3-Week Learning Plan](#3-week-learning-plan)
- [Recommended Weekly Schedule](#recommended-weekly-schedule)
- [Priority Topics for Data Engineers](#what-i-would-prioritize-for-you-as-a-data-engineer)


Databricks’ current data-engineering direction is centered around **Lakeflow**, which is described as an end-to-end data engineering solution for ingestion, transformation, and orchestration. ([docs.databricks.com](https://docs.databricks.com/aws/en/data-engineering/)) Unity Catalog is also central because it governs both **data and AI assets**, including access control, lineage, auditing, discovery, and AI governance. ([docs.databricks.com](https://docs.databricks.com/aws/en/data-governance/unity-catalog/))

---

## Best official learning resources

### 1. Databricks Academy — start here

Use the **Data Engineer** learning path and later the **Generative AI Engineer** path. Databricks has role-based learning paths for Data Engineer, Generative AI Engineer, ML Engineer, Analyst, and Architect. ([databricks.com](https://www.databricks.com/learn/training/home))

Recommended order:

1. Databricks Lakehouse fundamentals  
2. Data Engineering with Databricks  
3. Delta Lake / Delta tables  
4. Databricks SQL basics  
5. Workflows / Jobs  
6. Unity Catalog  
7. Generative AI Engineering / Mosaic AI

---

### 2. Official documentation to read systematically

Use these as your “textbook”:

| Topic | Why it matters |
|---|---|
| **Data engineering with Databricks** | Main overview for modern Databricks ETL, Lakeflow, orchestration, ingestion, and transformations. ([docs.databricks.com](https://docs.databricks.com/aws/en/data-engineering/)) |
| **Databricks components / concepts** | Good mental model of workspaces, compute, data objects, Unity Catalog, SQL warehouses, and AI/ML assets. ([docs.databricks.com](https://docs.databricks.com/aws/en/getting-started/concepts)) |
| **Auto Loader** | Important for incremental ingestion from cloud object storage; it uses the `cloudFiles` Structured Streaming source. ([docs.databricks.com](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/)) |
| **Structured Streaming with Delta tables** | Core for batch + streaming architecture on Delta Lake. ([docs.databricks.com](https://docs.databricks.com/aws/en/structured-streaming/delta-lake)) |
| **Unity Catalog** | Required for enterprise governance, data discovery, lineage, auditing, sharing, and AI governance. ([docs.databricks.com](https://docs.databricks.com/aws/en/data-governance/unity-catalog/)) |
| **Unity Catalog with Structured Streaming** | Useful when building governed streaming/incremental pipelines. ([docs.databricks.com](https://docs.databricks.com/aws/en/structured-streaming/unity-catalog)) |
| **Unity Catalog table types** | Learn managed, external, and foreign tables before designing production data layers. ([docs.databricks.com](https://docs.databricks.com/aws/en/tables/delta-table)) |

---

## Best YouTube / conference-style videos

These are close to AWS re:Invent-style sessions: longer, practical, and architecture-oriented.

### Core Databricks / Lakehouse / Data Engineering

1. **Introducing Lakeflow: The Future of Data Engineering on Databricks**  
   Good high-level view of the new Databricks data-engineering direction. ([youtube.com](https://www.youtube.com/watch?v=lq8w9kpbGB4&utm_source=chatgpt.com))

2. **Simplify ETL pipelines on the Databricks Lakehouse**  
   Good for understanding why Databricks positions the lakehouse as a unified ETL platform. ([youtube.com](https://www.youtube.com/watch?v=SfNglvSeOoA&utm_source=chatgpt.com))

3. **Best of Data Engineering — Data + AI Summit 2025 playlist**  
   This is probably the closest match to AWS re:Invent data-engineering sessions. It collects top Data + AI Summit engineering talks around reliable, scalable pipelines. ([youtube.com](https://www.youtube.com/playlist?list=PLTPXxbhUt-YWlKZVNyTURC9xBctbsoEQc&utm_source=chatgpt.com))

4. **Data + AI Summit 2025 — Data Engineering + Streaming playlist**  
   Strong playlist for streaming, Lakeflow, declarative pipelines, and modern pipeline patterns. ([m.youtube.com](https://m.youtube.com/playlist?list=PLdcsMc3thIRwNCkghQx8mOQ-DpXpkQoqE&utm_source=chatgpt.com))

5. **Authoring Data Pipelines With the New Lakeflow Declarative Pipelines**  
   Useful for hands-on Lakeflow Declarative Pipelines and code-first data pipeline authoring. ([youtube.com](https://www.youtube.com/watch?v=eF34sUvYdxw&utm_source=chatgpt.com))

6. **Simplifying streaming pipelines with Lakeflow**  
   Good for understanding batch + streaming convergence in Databricks. ([youtube.com](https://www.youtube.com/watch?v=QAsQx9EjLT4&utm_source=chatgpt.com))

7. **Building reliable ETL pipelines with built-in observability**  
   Useful for production thinking: observability, reliability, and pipeline operations. ([youtube.com](https://www.youtube.com/watch?v=XiOY58yrZbQ&utm_source=chatgpt.com))

---

### Delta Lake / Storage / Lakehouse fundamentals

1. **Delta Lake — Explained — Full Tutorial**  
   Good for Delta Lake concepts, interview preparation, and certification-style learning. ([youtube.com](https://www.youtube.com/watch?v=fkWxiesfrgk&utm_source=chatgpt.com))

2. **Simplify and Scale Data Engineering Pipelines with Delta Lake**  
   Good for understanding why Delta Lake matters for scalable pipelines. ([youtube.com](https://www.youtube.com/watch?v=exczhYoB5vc&utm_source=chatgpt.com))

3. **Delta Lake official site**  
   Delta Lake is an open-source storage framework for building lakehouse architectures and now supports integrations across engines such as Spark, Trino, Flink, Snowflake, BigQuery, Athena, Redshift, and others. ([delta.io](https://delta.io/))

---

### Unity Catalog / Governance

1. **Databricks Unity Catalog Demo: ABAC, Lineage, Lakehouse Federation**  
   Practical demo of Unity Catalog basics, lineage, federation, and attribute-based access control. ([youtube.com](https://www.youtube.com/watch?v=lWzh7HmiynA&utm_source=chatgpt.com))

2. **A Technical Deep Dive into Unity Catalog’s Practitioner Guide**  
   Better for enterprise governance, data + AI governance, and hands-on best practices. ([youtube.com](https://www.youtube.com/watch?v=LzmmObc_Bmw&utm_source=chatgpt.com))

---

### End-to-end project videos

1. **Databricks End-To-End Project: Streaming, AI, Lakeflow, Unity Catalog, AI/BI**  
   A 4+ hour hands-on project covering streaming, batch, Unity Catalog, Lakeflow, Mosaic AI sentiment analysis, and AI/BI dashboards. Good capstone material. ([youtube.com](https://www.youtube.com/watch?v=vy4G86q1rQY&utm_source=chatgpt.com))

2. **Medallion Lakehouse with Databricks Lakeflow Declarative Pipelines**  
   Good for building Bronze/Silver/Gold architecture using Lakeflow Declarative Pipelines. ([youtube.com](https://www.youtube.com/watch?v=8v150KsR5ak&utm_source=chatgpt.com))

---

## Best eBooks / deeper reading

### 1. The Big Book of Data Engineering — 4th Edition

This is the main Databricks data-engineering eBook. The current Databricks page says the 4th edition covers modern approaches for building pipelines faster and delivering high-quality data for AI, BI, and analytics workloads. ([databricks.com](https://www.databricks.com/resources/ebook/big-book-of-data-engineering))

Use it as your main deep-reading resource.

### 2. Databricks Solution Accelerators

Databricks’ Solution Accelerators are functional notebooks for common, high-impact industry use cases and are intended as starting points for new data use cases and product development. ([github.com](https://github.com/databricks-industry-solutions))

Use these after week 2, when you want realistic notebooks instead of toy examples.

---

# 3-week learning plan

Assumption: **5–7 hours per week**.  
Goal: after 3 weeks, you should be able to build a small **production-style Databricks pipeline** and understand how it connects to AI/RAG use cases.

---

## Week 1 — Databricks foundation + Delta Lake + PySpark

### Goal

Understand how Databricks is organized and how Delta Lake powers the Lakehouse.

### Learn

Focus on:

- Databricks workspace, notebooks, clusters/serverless compute, SQL warehouse
- Lakehouse architecture
- Delta tables
- Managed vs external tables
- Bronze / Silver / Gold medallion architecture
- Basic PySpark DataFrame operations
- Basic SQL on Delta tables

### Watch

1. **Introducing Lakeflow: The Future of Data Engineering on Databricks** ([youtube.com](https://www.youtube.com/watch?v=lq8w9kpbGB4&utm_source=chatgpt.com))  
2. **Simplify ETL pipelines on the Databricks Lakehouse** ([youtube.com](https://www.youtube.com/watch?v=SfNglvSeOoA&utm_source=chatgpt.com))  
3. **Delta Lake — Explained — Full Tutorial** ([youtube.com](https://www.youtube.com/watch?v=fkWxiesfrgk&utm_source=chatgpt.com))  

### Read

- Databricks components / concepts ([docs.databricks.com](https://docs.databricks.com/aws/en/getting-started/concepts))  
- Data engineering with Databricks ([docs.databricks.com](https://docs.databricks.com/aws/en/data-engineering/))  
- Delta Lake official overview ([delta.io](https://delta.io/))  
- Unity Catalog table types: managed, external, foreign tables ([docs.databricks.com](https://docs.databricks.com/aws/en/tables/delta-table))  

### Hands-on work

Build a small pipeline:

1. Upload CSV or JSON data into Databricks.
2. Create a **Bronze Delta table**.
3. Clean and deduplicate into a **Silver table**.
4. Aggregate into a **Gold table**.
5. Query it with Databricks SQL.

Example project:

> Ingest raw sales/order/customer data → clean customer and order records → create daily revenue and customer activity tables.

### Deliverable

By the end of week 1, you should have:

- One notebook for ingestion
- One notebook for transformation
- Three Delta tables: Bronze, Silver, Gold
- One SQL query or dashboard table

---

## Week 2 — Production data engineering: Auto Loader, streaming, Lakeflow, Workflows, Unity Catalog

### Goal

Move from notebook-style ETL to production-style pipelines.

### Learn

Focus on:

- Auto Loader
- Structured Streaming
- Lakeflow Declarative Pipelines
- Databricks Workflows / Jobs
- Pipeline observability
- Data quality expectations
- Unity Catalog permissions and lineage

Auto Loader is important because it incrementally processes new files arriving in cloud storage using the `cloudFiles` source. ([docs.databricks.com](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/)) Delta tables are also commonly used as streaming sources and sinks with Spark Structured Streaming. ([docs.databricks.com](https://docs.databricks.com/aws/en/structured-streaming/delta-lake))

### Watch

1. **Authoring Data Pipelines With the New Lakeflow Declarative Pipelines** ([youtube.com](https://www.youtube.com/watch?v=eF34sUvYdxw&utm_source=chatgpt.com))  
2. **Simplifying streaming pipelines with Lakeflow** ([youtube.com](https://www.youtube.com/watch?v=QAsQx9EjLT4&utm_source=chatgpt.com))  
3. **Building reliable ETL pipelines with built-in observability** ([youtube.com](https://www.youtube.com/watch?v=XiOY58yrZbQ&utm_source=chatgpt.com))  
4. **Unity Catalog Demo: ABAC, Lineage, Lakehouse Federation** ([youtube.com](https://www.youtube.com/watch?v=lWzh7HmiynA&utm_source=chatgpt.com))  

### Read

- Auto Loader docs ([docs.databricks.com](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/))  
- Structured Streaming with Delta tables ([docs.databricks.com](https://docs.databricks.com/aws/en/structured-streaming/delta-lake))  
- Unity Catalog docs ([docs.databricks.com](https://docs.databricks.com/aws/en/data-governance/unity-catalog/))  
- Unity Catalog with Structured Streaming ([docs.databricks.com](https://docs.databricks.com/aws/en/structured-streaming/unity-catalog))  

### Hands-on work

Upgrade your week 1 project:

1. Replace manual file loading with **Auto Loader**.
2. Add incremental ingestion.
3. Add a streaming or near-real-time Silver table.
4. Add data quality rules.
5. Schedule the pipeline using **Databricks Workflows**.
6. Register all tables in **Unity Catalog**.
7. Check lineage from Bronze → Silver → Gold.

### Deliverable

By the end of week 2, you should have:

- Incremental ingestion pipeline
- Basic streaming or micro-batch pattern
- Scheduled workflow/job
- Unity Catalog-managed tables
- Data lineage visible in Catalog Explorer
- Data quality checks

---

## Week 3 — Databricks for AI: feature tables, Vector Search, RAG, Mosaic AI, AI/BI

### Goal

Understand how a data engineer supports AI workloads on Databricks.

For AI work, your role is not only “train models.” A data engineer’s AI role is usually:

- prepare reliable data
- govern data access
- create high-quality feature or knowledge tables
- build embedding pipelines
- support RAG applications
- monitor data quality and lineage
- serve curated data to ML/AI teams

### Learn

Focus on:

- Mosaic AI basics
- Embeddings
- Vector Search
- RAG architecture
- Model Serving
- AI/BI dashboards
- How Unity Catalog governs AI assets

Unity Catalog now covers governance for both data and AI assets, including AI governance and AI Gateway capabilities. ([docs.databricks.com](https://docs.databricks.com/aws/en/data-governance/unity-catalog/)) Databricks Solution Accelerators are also useful here because they provide working notebooks for real use cases. ([github.com](https://github.com/databricks-industry-solutions))

### Watch

1. **Databricks End-To-End Project: Streaming, AI, Lakeflow, Unity Catalog, AI/BI** ([youtube.com](https://www.youtube.com/watch?v=vy4G86q1rQY&utm_source=chatgpt.com))  
2. **Best of Data Engineering — Data + AI Summit 2025 playlist** ([youtube.com](https://www.youtube.com/playlist?list=PLTPXxbhUt-YWlKZVNyTURC9xBctbsoEQc&utm_source=chatgpt.com))  
3. **Data Engineering + Streaming — Data + AI Summit 2025 playlist** ([m.youtube.com](https://m.youtube.com/playlist?list=PLdcsMc3thIRwNCkghQx8mOQ-DpXpkQoqE&utm_source=chatgpt.com))  

### Read

- The Big Book of Data Engineering — 4th Edition ([databricks.com](https://www.databricks.com/resources/ebook/big-book-of-data-engineering))  
- Databricks Solution Accelerators GitHub organization ([github.com](https://github.com/databricks-industry-solutions))  
- Mosaic AI / Vector Search / RAG-related Databricks materials, especially for retrieval quality and reranking. Databricks has published guidance on improving RAG quality with Mosaic AI Vector Search reranking. ([databricks.com](https://www.databricks.com/blog/reranking-mosaic-ai-vector-search-faster-smarter-retrieval-rag-agents))  

### Hands-on work

Extend your project into an AI-style use case:

Option A — **RAG over business documents**

1. Load PDFs, markdown, support tickets, or product documents.
2. Clean and chunk the text.
3. Generate embeddings.
4. Store embeddings in Vector Search.
5. Build a simple RAG notebook.
6. Add metadata filters using Unity Catalog-governed tables.

Option B — **AI analytics**

1. Build Gold tables for business metrics.
2. Create AI/BI dashboard.
3. Add natural-language business questions.
4. Use Databricks SQL for validation.

Option C — **ML feature pipeline**

1. Create customer/order/product features.
2. Save feature-ready Delta tables.
3. Track lineage and quality.
4. Prepare data for model training or serving.

### Deliverable

By the end of week 3, you should have one mini-capstone:

> A governed Databricks Lakehouse pipeline that ingests raw data, builds Bronze/Silver/Gold Delta tables, schedules the pipeline, tracks lineage in Unity Catalog, and supports either a dashboard or a small RAG/AI use case.

---

# Recommended weekly schedule

## Week 1 schedule

| Day | Task |
|---|---|
| Day 1 | Watch Lakehouse / Lakeflow overview. Read Databricks components. |
| Day 2 | Learn Delta Lake basics. Create first Delta table. |
| Day 3 | Practice PySpark transformations. |
| Day 4 | Build Bronze → Silver → Gold manually. |
| Day 5 | Query with SQL and document your architecture. |

---

## Week 2 schedule

| Day | Task |
|---|---|
| Day 1 | Learn Auto Loader. Replace manual ingestion. |
| Day 2 | Learn Structured Streaming with Delta. |
| Day 3 | Learn Lakeflow Declarative Pipelines. |
| Day 4 | Add Workflows and schedule the job. |
| Day 5 | Add Unity Catalog, permissions, lineage, and quality checks. |

---

## Week 3 schedule

| Day | Task |
|---|---|
| Day 1 | Watch end-to-end project video. |
| Day 2 | Learn Mosaic AI / Vector Search / RAG basics. |
| Day 3 | Build either RAG, AI dashboard, or feature pipeline. |
| Day 4 | Add governance and documentation. |
| Day 5 | Final review: architecture diagram, README, cost/performance notes. |

---

# What I would prioritize for you as a data engineer

Your priority should be:

1. **Delta Lake deeply** — transaction log, schema evolution, partitioning, optimization, time travel.
2. **Lakeflow / Declarative Pipelines** — this is Databricks’ modern direction for ETL.
3. **Auto Loader + Structured Streaming** — important for real-world ingestion.
4. **Unity Catalog** — required for enterprise data and AI governance.
5. **Workflows** — production scheduling and orchestration.
6. **Databricks SQL** — analytics layer.
7. **Mosaic AI + Vector Search** — AI path for data engineers.

The main mindset shift is: **Databricks is not only Spark notebooks. It is a governed data + AI platform.** Your long-term value as a data engineer is building reliable, governed, AI-ready data products.

---

## Final Capstone Checklist

Use this checklist to confirm that your 3-week learning project is complete:

- [ ] Raw data ingestion implemented
- [ ] Bronze Delta table created
- [ ] Silver cleaned/deduplicated table created
- [ ] Gold analytics table created
- [ ] Auto Loader or incremental ingestion added
- [ ] Workflow/job scheduled
- [ ] Unity Catalog used for tables and governance
- [ ] Data quality checks added
- [ ] Lineage reviewed in Catalog Explorer
- [ ] AI/BI dashboard, RAG prototype, or feature pipeline completed
- [ ] README written with architecture, assumptions, and next steps

---

## Suggested Repository Structure

```text
databricks-learning-capstone/
├── README.md
├── notebooks/
│   ├── 01_bronze_ingestion.py
│   ├── 02_silver_transform.py
│   ├── 03_gold_analytics.py
│   └── 04_ai_or_rag_extension.py
├── sql/
│   └── dashboard_queries.sql
├── configs/
│   └── pipeline_config.yml
├── docs/
│   └── architecture.md
└── data_samples/
    └── sample_orders.csv
```
