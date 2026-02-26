# 📚 Academic Paper Trend Tracker

![Azure](https://img.shields.io/badge/Azure-Data%20Factory-0078D4?style=flat&logo=microsoftazure)
![Storage](https://img.shields.io/badge/Azure-Data%20Lake%20Gen2-0078D4?style=flat&logo=microsoftazure)
![KeyVault](https://img.shields.io/badge/Azure-Key%20Vault-0078D4?style=flat&logo=microsoftazure)
![LogicApps](https://img.shields.io/badge/Azure-Logic%20Apps-0078D4?style=flat&logo=microsoftazure)

An end-to-end automated data pipeline that ingests, transforms, and loads academic paper data from the **Semantic Scholar API** into Azure Data Lake Storage Gen2 — with daily scheduling, Parquet conversion, and failure email alerting via Logic Apps.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Pipeline Flow](#pipeline-flow)
- [Project Structure](#project-structure)
- [Azure Resources](#azure-resources)
- [Setup Instructions](#setup-instructions)
- [Pipeline Details](#pipeline-details)
- [Data Transformation](#data-transformation)
- [Monitoring & Alerting](#monitoring--alerting)
- [Data Schema](#data-schema)

---

## 🔍 Overview

This project demonstrates a complete modern data engineering workflow on Azure — from raw API ingestion all the way to analytics-ready Parquet files, automated daily scheduling, and failure notifications.

**Key capabilities:**
- Automatically fetches the latest academic papers from Semantic Scholar every day at 8:00 AM WIB
- Transforms and cleans raw JSON data into a structured, analytics-ready format
- Converts processed data to Parquet format for efficient querying
- Sends email alerts via Azure Logic Apps when any pipeline activity fails
- Stores all credentials securely in Azure Key Vault — zero hardcoded secrets

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Azure Data Factory                          │
│                                                                 │
│  ┌──────────┐    ┌───────────────┐    ┌──────────────────────┐ │
│  │ Web      │    │ Copy Activity │    │ Data Flow            │ │
│  │ Activity │───▶│ (API Ingest)  │───▶│ (Transform & Clean)  │ │
│  │ (KV Key) │    │               │    │                      │ │
│  └──────────┘    └───────────────┘    └──────────────────────┘ │
│                                                ▼                │
│                                    ┌───────────────────────┐   │
│                                    │ Copy Activity         │   │
│                                    │ (Convert to Parquet)  │   │
│                                    └───────────────────────┘   │
│                       On Failure ▼                              │
│                    ┌──────────────────┐                         │
│                    │  Web Activity    │                         │
│                    │  (notifyFailure) │                         │
│                    └──────────────────┘                         │
└─────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────┐    ┌─────────────────┐   ┌────────────────┐
│ Azure Key   │    │ ADLS Gen2       │   │ Azure Logic    │
│ Vault       │    │                 │   │ Apps           │
│             │    │ raw/            │   │                │
│ API Key     │    │  └─ papers/     │   │ Email Alert    │
│ Conn String │    │ processed/      │   │ on Failure     │
│             │    │  ├─ papers/     │   └────────────────┘
└─────────────┘    │  └─ parquet/    │
                   └─────────────────┘
```

**Medallion Architecture:**

| Layer | Container | Format | Description |
|---|---|---|---|
| 🥉 Bronze | `raw/papers/` | JSON | Raw data langsung dari API |
| 🥈 Silver | `processed/papers/` | JSON | Data setelah transformasi & cleaning |
| 🥇 Gold | `processed/parquet/` | Parquet | Analytics-ready columnar format |

---

## 🛠️ Tech Stack

| Technology | Purpose |
|---|---|
| **Azure Data Factory** | Pipeline orchestration & scheduling |
| **Azure Data Lake Storage Gen2** | Scalable data storage (Bronze/Silver/Gold) |
| **Azure Key Vault** | Secure secrets management |
| **Azure Logic Apps** | Failure notification workflow |
| **Semantic Scholar API** | Source data — academic papers |
| **GitHub** | Version control & ADF pipeline source of truth |

---

## 🔄 Pipeline Flow

```
Schedule Trigger (Daily 08:00 WIB)
         │
         ▼
1. getApiKey
   └─ Web Activity → Azure Key Vault
   └─ Retrieves: semantic-scholar-api-key
         │
         ▼
2. copy_papers_from_api
   └─ Copy Activity → HTTP Source → ADLS Gen2 Sink
   └─ Endpoint: GET /graph/v1/paper/search?query=machine+learning
   └─ Output: raw/papers/part-xxxxx.json
         │
         ▼
3. transform_papers
   └─ Data Flow (df_transform_papers)
   └─ Flatten → Select → Derived Column → Select → Sink
   └─ Output: processed/papers/part-xxxxx.json
         │
         ▼
4. convert_to_parquet
   └─ Copy Activity → JSON Source → Parquet Sink
   └─ Merges all part files → single papers.parquet
   └─ Output: processed/parquet/papers.parquet
         │
    On Failure (any activity)
         ▼
5. notifyFailure
   └─ Web Activity → Azure Logic Apps HTTP Trigger
   └─ Sends: pipelineName, status, errorMessage, runId, time
   └─ Output: Email notification
```

---

## 📁 Project Structure

```
azure-data-factory-academic-tracker/
│
├── pipelines/                          # ADF definitions (auto-synced from ADF Studio)
│   ├── pl_ingest_academic_papers.json  # Main pipeline
│   ├── df_transform_papers.json        # Data Flow transformation
│   ├── dataset/
│   │   ├── ds_http_papers_source.json
│   │   ├── ds_adls_papers_raw.json
│   │   ├── ds_adls_papers_processed.json
│   │   └── ds_adls_papers_parquet.json
│   └── linkedService/
│       ├── ls_http_semantic_scholar.json
│       ├── ls_adls_academic_tracker.json
│       └── ls_keyvault_academic_tracker.json
│
├── .env.example                        # Environment variable template
├── requirements.txt                    # Python dependencies
└── README.md
```

> **Note:** All pipeline definitions are auto-committed to this repository by ADF Studio whenever changes are saved — no manual sync required.

---

## ☁️ Azure Resources

| Resource | Name | Purpose |
|---|---|---|
| Resource Group | `rg-academic-tracker-dev-eastus` | Container for all resources |
| Storage Account | `acadtrackerdev001` | ADLS Gen2 data storage |
| Azure Data Factory | `adf-academic-tracker-dev` | Pipeline orchestration |
| Key Vault | `kv-academic-tracker-dev` | Secrets management |
| Logic App | `logic-academic-tracker-dev` | Failure notification workflow |

**Key Vault Secrets:**

| Secret Name | Purpose |
|---|---|
| `semantic-scholar-api-key` | API key for Semantic Scholar API |
| `storage-connection-string` | Connection string for Storage Account |

**RBAC Assignments:**

| Principal | Role | Scope |
|---|---|---|
| Developer account | Key Vault Secrets Officer | kv-academic-tracker-dev |
| ADF Managed Identity | Key Vault Secrets User | kv-academic-tracker-dev |

---

## ⚙️ Setup Instructions

### Prerequisites
- Azure subscription (Azure for Students or Pay-As-You-Go)
- Azure CLI
- Git

### 1. Clone Repository
```bash
git clone https://github.com/AlmaNurulSalma-dev/azure-data-factory-academic-tracker.git
cd azure-data-factory-academic-tracker
```

### 2. Configure Environment Variables
```bash
cp .env.example .env
# Edit .env with your actual values
```

### 3. Login to Azure CLI
```bash
az login
az account set --subscription "your-subscription-id"
```

### 4. Provision Azure Resources
```bash
# Resource Group
az group create --name rg-academic-tracker-dev-eastus --location eastus

# Storage Account (ADLS Gen2)
az storage account create --name acadtrackerdev001 --resource-group rg-academic-tracker-dev-eastus --location eastus --sku Standard_LRS --kind StorageV2 --hns true

# Containers
az storage container create --name raw --account-name acadtrackerdev001
az storage container create --name processed --account-name acadtrackerdev001

# Azure Data Factory
az datafactory create --name adf-academic-tracker-dev --resource-group rg-academic-tracker-dev-eastus --location eastus

# Key Vault
az keyvault create --name kv-academic-tracker-dev --resource-group rg-academic-tracker-dev-eastus --location eastus --enable-rbac-authorization true
```

### 5. Store Secrets in Key Vault
```bash
az keyvault secret set --vault-name kv-academic-tracker-dev --name "semantic-scholar-api-key" --value "your-api-key"
az keyvault secret set --vault-name kv-academic-tracker-dev --name "storage-connection-string" --value "your-connection-string"
```

### 6. Assign RBAC Permissions to ADF
```bash
az role assignment create --role "Key Vault Secrets User" --assignee "$(az datafactory show --name adf-academic-tracker-dev --resource-group rg-academic-tracker-dev-eastus --query identity.principalId -o tsv)" --scope "/subscriptions/{subscription-id}/resourcegroups/rg-academic-tracker-dev-eastus/providers/Microsoft.KeyVault/vaults/kv-academic-tracker-dev"
```

### 7. Connect ADF to GitHub
In ADF Studio → Manage → Git configuration:
- Repository: `azure-data-factory-academic-tracker`
- Collaboration branch: `main`
- Publish branch: `adf_publish`
- Root folder: `/pipelines`

---

## 🔧 Pipeline Details

### Pipeline: `pl_ingest_academic_papers`

| Activity | Type | Description |
|---|---|---|
| `getApiKey` | Web Activity | Retrieve API key from Key Vault |
| `copy_papers_from_api` | Copy Activity | Fetch papers from Semantic Scholar API |
| `transform_papers` | Data Flow | Transform and clean raw data |
| `convert_to_parquet` | Copy Activity | Convert JSON to Parquet format |
| `notifyFailure` | Web Activity | Send failure notification via Logic Apps |

### Trigger: `tr_daily_ingest_papers`

| Setting | Value |
|---|---|
| Type | Schedule |
| Frequency | Daily |
| Time | 08:00 WIB (UTC+7) |

### API Endpoint
```
GET https://api.semanticscholar.org/graph/v1/paper/search
  ?query=machine+learning
  &fields=title,authors,year,citationCount,abstract
  &limit=10
```

---

## 🔀 Data Transformation

### Data Flow: `df_transform_papers`

```
rawPapers (Source)
    │ Read JSON from raw/papers/
    ▼
flattenData (Flatten)
    │ Unroll nested "data" array → individual rows
    ▼
selectColumns (Select)
    │ Keep: paperId, title, year, citationCount, authors_raw, abstract
    │ Drop: openAccessPdf, total, next, offset
    ▼
transformColumns (Derived Column)
    │ authors     → flatten complex array to string
    │ abstract    → replace NULL with 'N/A'
    │ ingested_at → add currentTimestamp()
    ▼
dropRawColumns (Select)
    │ Drop: authors_raw
    ▼
sinkProcessed (Sink)
    └─ Write to processed/papers/
```

### Transformation Expressions

**Authors:**
```
case(isNull(authors_raw), 'N/A',
  case(size(authors_raw) == 1, toString(authors_raw[1].name),
  case(size(authors_raw) == 2,
    concat(toString(authors_raw[1].name), ', ', toString(authors_raw[2].name)),
    concat(toString(authors_raw[1].name), ', ', toString(authors_raw[2].name),
           ', ', toString(authors_raw[3].name)))))
```

**Abstract null handling:**
```
iif(isNull(abstract), 'N/A', abstract)
```

**Ingestion timestamp:**
```
currentTimestamp()
```

---

## 📊 Monitoring & Alerting

### ADF Built-in Alert
- **Alert name:** `alert-pipeline-failed`
- **Condition:** Pipeline Failed Runs > 0
- **Severity:** Sev1
- **Notification:** Email via Action Group `ag-academic-tracker-dev`

### Logic Apps Notification
- **Logic App:** `logic-academic-tracker-dev`
- **Trigger:** HTTP POST from ADF `notifyFailure` activity
- **Action:** Send email via Office 365 Outlook

**Payload sent by ADF on failure:**
```json
{
    "pipelineName": "@{pipeline().Pipeline}",
    "status": "Failed",
    "runId": "@{pipeline().RunId}",
    "errorMessage": "@{activity('copy_papers_from_api').error.message}",
    "time": "@{utcNow()}"
}
```

**Email received:**
```
Subject: [ALERT] Pipeline Failed: pl_ingest_academic_papers

Pipeline: pl_ingest_academic_papers
Run ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
Status: Failed
Error: [error message]
Time: 2026-02-25T08:00:00Z
```

---

## 📐 Data Schema

### Raw Layer (Bronze) — `raw/papers/`
```json
{
  "total": 876,
  "offset": 0,
  "next": 10,
  "data": [
    {
      "paperId": "string",
      "title": "string",
      "year": integer,
      "citationCount": integer,
      "abstract": "string | null",
      "authors": [{ "authorId": "string", "name": "string" }],
      "openAccessPdf": {}
    }
  ]
}
```

### Silver & Gold Layer — `processed/`

| Column | Type | Description |
|---|---|---|
| `paperId` | String | Unique paper identifier |
| `title` | String | Paper title |
| `year` | Integer | Publication year |
| `citationCount` | Integer | Number of citations |
| `authors` | String | Author names (comma-separated or JSON string) |
| `abstract` | String | Paper abstract ('N/A' if originally null) |
| `ingested_at` | Timestamp | Pipeline ingestion timestamp |

---

## 👩‍💻 Author

**Alma Nurul Salma**  
GitHub: [@AlmaNurulSalma-dev](https://github.com/AlmaNurulSalma-dev)

---

*Built as a first end-to-end Azure Data Factory project — February 2026*