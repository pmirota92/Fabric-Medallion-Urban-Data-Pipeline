# Warsaw Urban Data Flow: Near Real-Time GPS Analytics

<p align="center">
  <img src="https://img.shields.io/badge/Platform-Microsoft%20Fabric-blue?style=for-the-badge&logo=microsoft" alt="Microsoft Fabric">
  <img src="https://img.shields.io/badge/Language-Python%20%2F%20PySpark-yellow?style=for-the-badge&logo=python" alt="Python/PySpark">
  <img src="https://img.shields.io/badge/Storage-OneLake%20%28Delta%20Lake%29-green?style=for-the-badge&logo=databricks" alt="OneLake Delta">
  <img src="https://img.shields.io/badge/Viz-Power%20BI-orange?style=for-the-badge&logo=power-bi" alt="Power BI">
</p>

## 📊 Executive Summary

This repository hosts an end-to-end Data Engineering solution that ingests, processes, and visualizes near real-time GPS telemetry from the City of Warsaw's public transit fleet (buses and trams).

The project implements a full **Medallion Architecture** within the **Microsoft Fabric** SaaS data platform. The solution decouples local data extraction from cloud processing, showcasing a resilient, enterprise-ready data flow.

[Here: Upload and Insert Architecture Diagram `docs/architecture.png`]

## ⚙️ Tech Stack & Key Concepts

* **Extraction:** Local Python Script (CURL/HTTP).
* **Orchestration:** Fabric Data Pipelines.
* **Storage (Data Lake):** OneLake (Delta Lake format).
* **Processing:** Fabric Spark Notebooks (PySpark).
* **Visualization:** Power BI (via **Direct Lake** technology).

---

## 🛠️ Engineering Hurdles & Senior Problem-Solving

This project was not a straightforward tutorial deployment. Several critical real-world infrastructure and security restrictions were encountered and resolved.

### 1. The 430 Capacity Throttling Challenge
**Hurdle:** Fabric Trial Capacity experienced significant throttling (`430: TooManyRequestsForCapacity`) when executing standard Notebook-to-Notebook calls rapidly.
**Senior Solution:** Instead of manual retries, I implemented automated resilience by configuring the **Data Pipeline Retry Policy** with an exponential backoff (3 retries, 60s intervals). This smoothed capacity unit consumption and insured pipeline stability.

### 2. Initial Ingestion Architecture (Service Principal blockade)
**Plan A:** I intended to build a fully programmatic cloud ingestion layer using the Azure Identity SDK and a Registered App (Service Principal) to push data directly to OneLake API.
**Hurdle:** Organizational Tenant Security Policies restricted the use of Service Principals for data modification operations. I received `403 Forbidden` errors despite correct artifact-level permissions.

### 3. Final Ingestion Solution (Hybrid Sync Pivot)
**Plan B (Implemented):** To bypass tenant lockdown without compromising security, I pivoted to a **User-Identity flow**.
**Solution:** I utilized **OneLake File Explorer** to map the Fabric Lakehouse directly to the local file system. The local Python extraction script was updated to save dynamically-named JSON files (with timestamp) directly into the mapped OneLake folder. Windows handled the User-Identity authentication in the background, syncing files to the cloud near-instantaneously. This decoupled the restricted API flow from the ingestion need, allowing the project to proceed.

---

## 🏗️ Data Architecture (Medallion Flow)

1.  📁 **Files (input_data/gps):** Near real-time JSON dumps from local extractor.
2.  🥉 **Bronze Layer (lh_bronze):** Spark loads all pending raw JSONs, partitioning them by extraction date for optimized storage.
3.  🥈 **Silver Layer (lh_silver):** Spark reads Bronze data and performs a **Delta MERGE (Upsert)** operation. This layer guarantees zero duplicates and provides a "Current State" table of the fleet by vehicle ID. Duplicates are handled, and current location is updated.
4.  🥇 **Gold Layer (lh_gold):** (Optional/Analytical). Currently, Silver provides the live feed, but Gold can store daily/hourly aggregated punctuality metrics.

## 📈 Direct Lake Visualization

The Power BI dashboard connects directly to the Gold/Silver Delta tables in OneLake. Because Microsoft Fabric optimizes the interaction between Delta and Power BI, **we avoid expensive import/refresh processes**. When the Pipeline MERGE completes, the map points in Power BI move upon report refresh.

[Here: Upload and Insert Screenshot of Power BI Map `docs/map_screenshot.png`]

---

## 🚀 How to Run

1.  **Prerequisites:** Microsoft Fabric Capacity, local Python environment, City of Warsaw API Key.
2.  **Local Setup:** Install OneLake File Explorer, clone this repo, edit `scripts/config.py` (with API Key and OneLake path), run `jupyter notebook scripts/WarsawBusTracker.ipynb`.
3.  **Cloud Setup:** Create Fabric Items (Lakehouse, Notebooks, Pipeline), connect notebooks to Lakehouse.
4.  **Power BI:** Open `reports/Warsaw_Live.pbix`, change the data source to your Fabric workspace's SQL endpoint.
