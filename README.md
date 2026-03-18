<p align="center">
  <img src="docs/banner.png" alt="Revenue Intelligence Platform" width="100%"/>
</p>

<h1 align="center">Revenue Intelligence Platform</h1>

<p align="center">
  <strong>End-to-End Data Warehouse for Influencer Marketing Agency</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.11-3776AB?style=flat-square&logo=python&logoColor=white"/>
  <img src="https://img.shields.io/badge/PostgreSQL-14-4169E1?style=flat-square&logo=postgresql&logoColor=white"/>
  <img src="https://img.shields.io/badge/dbt-Core-FF694B?style=flat-square&logo=dbt&logoColor=white"/>
  <img src="https://img.shields.io/badge/Mage.ai-Orchestrator-7C3AED?style=flat-square"/>
  <img src="https://img.shields.io/badge/Docker-Compose-2496ED?style=flat-square&logo=docker&logoColor=white"/>
  <img src="https://img.shields.io/badge/PowerBI-Dashboard-F2C811?style=flat-square&logo=powerbi&logoColor=black"/>
</p>

<p align="center">
  <a href="docs/DE_blueprint_2026.md">📋 Blueprint</a> · 
  <a href="docs/architecture_design.md">🏗️ Architecture</a> · 
  <a href="docs/business_architecture.md">🏢 Business Context</a>
</p>

---

## Giới thiệu

Dự án xây dựng **Data Warehouse tự động hóa hoàn toàn** cho một Influencer Marketing Agency (~100-150 nhân sự, 2 văn phòng HN + HCM). Pipeline chạy hàng ngày lúc 7:00 AM, tổng hợp dữ liệu từ nhiều nguồn phân tán (Excel, Inhouse System, Influencer Platform) vào một **Star Schema** duy nhất, phục vụ ra quyết định kinh doanh cho CEO, Finance, và Account Team.

### Tại sao dự án này tồn tại?

Agency đang bị **thất thoát doanh thu** và **chậm ra quyết định** vì dữ liệu nằm rải rác ở 3 nơi khác nhau, không ai biết bản nào đúng:

| Vấn đề | Ảnh hưởng | Pipeline giải quyết |
|:---|:---|:---|
| Job hoàn thành nhưng quên thu tiền | ~5% doanh thu/tháng bị thất thoát | Flag `is_revenue_leakage` tự động |
| CEO cần data KOL ROI để pitch | Mất 2-3 ngày tổng hợp thủ công | Dashboard sẵn sàng lúc 8:30 AM |
| Tên KOL không nhất quán giữa các nguồn | 30% bản ghi bị duplicate | KOL Name Resolution tự động |
| 3 nguồn Excel song song, không đồng bộ | Quyết định sai vì data sai | Single Source of Truth (Gold Layer) |

---

## Kết quả đo lường được

```
✓ Loại bỏ 20 giờ/tháng công việc thủ công
✓ Phát hiện $2,000+ tiền chưa thu ngay tháng đầu tiên
✓ Giảm thời gian pitch từ 3 ngày xuống 1 giờ  
✓ Data accuracy từ 70% lên 99.9%
✓ Zero data loss nhờ Idempotent pipeline + DQ Database
```

---

## Kiến trúc Hệ thống

### Data Engineering Lifecycle

Dự án tuân theo **Data Engineering Lifecycle** (Reis & Housley, 2022):

```
Generation → Ingestion → Storage → Transformation → Serving
                                        ↕
                              Undercurrents:
                    Security · DataOps · Data Architecture
                    Orchestration · Data Management
```

### Pipeline Architecture

```mermaid
flowchart LR
    subgraph SRC["Source Systems"]
        S1["Job Tracking<br/>(Excel)"]
        S2["Revenue Data<br/>(Excel)"]
        S3["KOL Data<br/>(Platform)"]
    end

    subgraph ING["Ingestion (Python)"]
        P1["Data Profiler"]
        P2["Extract Module"]
        P3["Great Expectations"]
        P4["UPSERT Loader"]
    end

    subgraph STR["PostgreSQL (Docker)"]
        B["Bronze<br/>raw_jobs, raw_revenue, raw_kols"]
        SV["Silver<br/>stg_jobs, int_job_revenue"]
        G["Gold<br/>fact_daily_profit<br/>dim_influencer (SCD2)"]
        DQ["DQ Database<br/>dq_rejected_records"]
    end

    subgraph TRF["Transformation (dbt)"]
        D1["Staging Models"]
        D2["Business Logic"]
        D3["Star Schema"]
    end

    subgraph SRV["Serving"]
        BI1["Revenue Leakage<br/>Dashboard"]
        BI2["KOL ROI<br/>Ranking"]
        BI3["DQ Monitor"]
    end

    SRC --> ING
    P3 -->|rejected| DQ
    ING --> B
    B --> TRF
    TRF --> G
    G --> SRV

    style SRC fill:#fff3cd,stroke:#ffc107
    style ING fill:#d1ecf1,stroke:#0dcaf0
    style STR fill:#d4edda,stroke:#198754
    style TRF fill:#f8d7da,stroke:#dc3545
    style SRV fill:#cfe2ff,stroke:#0d6efd
```

### Data Model (Star Schema — Kimball)

```mermaid
erDiagram
    dim_influencer ||--o{ fact_daily_profit : "kol_sk"
    dim_brand ||--o{ fact_daily_profit : "brand_id"
    dim_date ||--o{ fact_daily_profit : "date_id"
    dim_influencer ||--o{ fact_kol_performance : "kol_sk"
    dim_brand ||--o{ fact_kol_performance : "brand_id"
    dim_date ||--o{ fact_kol_performance : "date_id"

    dim_influencer {
        int kol_sk PK "Surrogate Key"
        int kol_id "Natural Key"
        varchar kol_name_clean
        varchar platform "TikTok/IG/YT"
        varchar tier "Mega/Macro/Micro/Nano"
        date valid_from "SCD Type 2"
        date valid_to "SCD Type 2"
        boolean is_current "SCD Type 2"
    }

    fact_daily_profit {
        int job_id PK
        int kol_sk FK
        int brand_id FK
        int invoice_date_id FK
        decimal gross_revenue
        decimal net_profit
        boolean is_revenue_leakage
    }

    fact_kol_performance {
        int performance_id PK
        int kol_sk FK
        decimal roi_score
        decimal cost_per_engagement
        bigint total_reach
    }
```

> Chi tiết đầy đủ: [Architecture Design](docs/architecture_design.md) (Component, Sequence, Activity, ERD, Deployment diagrams + ADR + SLA)

---

## Tech Stack

| Layer | Technology | Vai trò |
|:---|:---|:---|
| **Ingestion** | Python 3.11 (Pandas, OpenPyxl) | Extract từ Excel, xử lý merged headers, encoding |
| **Validation** | Great Expectations | Data Quality gate — chặn data bẩn trước khi vào hệ thống |
| **Storage** | PostgreSQL 14 (Docker) | Medallion Architecture: Bronze → Silver → Gold |
| **Transformation** | dbt-core | Business logic (SQL + Jinja), unit test, Data Catalog |
| **Orchestration** | Mage.ai | Pipeline DAG, cron scheduling, job monitoring |
| **BI / Serving** | PowerBI | Dashboards: Revenue Leakage, KOL ROI, DQ Monitor |
| **Containerization** | Docker Compose | postgres + pgAdmin + Mage.ai |
| **Testing** | pytest + dbt test | Python unit tests + SQL data tests |
| **Alerting** | Slack Webhook | Pipeline success/failure notifications |

---

## Nền tảng Lý thuyết

Dự án được thiết kế dựa trên các framework từ 7 cuốn sách Data Engineering:

| Concept | Áp dụng | Nguồn |
|:---|:---|:---|
| **Data Engineering Lifecycle** | 5 giai đoạn: Generation → Ingestion → Storage → Transformation → Serving | *Fundamentals of DE — Reis & Housley* |
| **Undercurrents** | Security (Postgres Roles), DataOps (CI/CD), Data Management (dbt docs) | *Fundamentals of DE* |
| **Star Schema** | Fact + Dimension tables tối ưu cho BI read queries | *The Data Warehouse Toolkit — Kimball* |
| **SCD Type 2** | Lưu lịch sử thay đổi tier KOL (Micro → Macro) | *Kimball, Ch.5* |
| **DQ Database** | Tách riêng data bẩn, không xóa — audit trail đầy đủ | *Building A Data Warehouse — Rainardi* |
| **Idempotent Pipeline** | UPSERT thay INSERT, chạy lại không duplicate | *Rebuilding Reliable Pipelines — Malaska* |
| **Lambda Architecture** | Batch-first, thiết kế sẵn cho Speed Layer (Kafka) | *Big Data — Nathan Marz* |

---

## Roadmap

```mermaid
gantt
    title Implementation Roadmap
    dateFormat YYYY-MM-DD
    axisFormat %d/%m

    section Phase 1 — Ingestion
        Docker + PostgreSQL setup        :done, p1a, 2026-02-17, 2d
        Data Profiling                   :done, p1b, after p1a, 1d
        Extract + Validate               :done, p1c, after p1b, 2d
        DQ Rejected Records              :done, p1d, after p1c, 1d
        UPSERT to Bronze                 :done, p1e, after p1d, 1d

    section Phase 2 — Transformation
        dbt project setup                :active, p2a, 2026-02-24, 1d
        Staging models + Name Resolution :active, p2b, after p2a, 2d
        Join Jobs + Revenue              :p2c, after p2b, 2d
        Data Lineage + Catalog           :p2d, after p2c, 2d

    section Phase 3 — Gold Layer
        dim_influencer (SCD2)            :p3a, 2026-03-03, 2d
        fact_daily_profit                :p3b, after p3a, 2d
        Revenue Leakage Flag             :p3c, after p3b, 1d
        DQ Metrics Table                 :p3d, after p3c, 1d

    section Phase 4 — Orchestration
        Mage.ai pipeline                 :p4a, 2026-03-10, 2d
        Schedule + Alert                 :p4b, after p4a, 2d
        PowerBI Dashboards               :p4c, after p4b, 3d
        Security Roles + Case Study      :p4d, after p4c, 2d
```

---

## Cấu trúc Thư mục

```
revenue-intelligence-platform/
│
├── README.md
├── docker-compose.yml
├── .env.example
│
├── ingestion/                  # Python extraction scripts
│   ├── extractors/
│   │   ├── base_extractor.py
│   │   ├── job_extractor.py
│   │   └── revenue_extractor.py
│   ├── validators/
│   │   └── great_expectations/
│   ├── loaders/
│   │   └── postgres_loader.py
│   └── profiler/
│       └── data_profiler.py
│
├── dbt_project/                # dbt transformation
│   ├── models/
│   │   ├── staging/            # Bronze → Silver
│   │   ├── intermediate/       # Business logic joins
│   │   └── marts/              # Gold Layer (Star Schema)
│   ├── tests/
│   ├── macros/
│   └── schema.yml              # Data Catalog
│
├── orchestration/              # Mage.ai pipeline definitions
│   └── pipelines/
│       └── daily_etl/
│
├── dashboards/                 # PowerBI files + screenshots
│
├── tests/                      # pytest suite
│
└── docs/                       # Documentation
    ├── banner.png
    ├── DE_blueprint_2026.md
    ├── architecture_design.md
    └── business_architecture.md
```

---

## Thiết kế cho Mở rộng

| Kịch bản | Giải pháp | Effort |
|:---|:---|:---|
| Thêm nguồn dữ liệu (CRM, API) | Thêm 1 Extractor module mới kế thừa `BaseExtractor` | Low |
| Chuyển lên Cloud (AWS/GCP) | Đổi connection string trong `.env` | Low |
| Real-time tracking | Thêm Kafka trước Ingestion Layer (Lambda Architecture) | Medium |
| Data Lakehouse | Chuyển Bronze sang Delta Lake / Apache Iceberg | Medium |

---

## Tài liệu Chi tiết

| Tài liệu | Nội dung |
|:---|:---|
| [DE Blueprint 2026](docs/DE_blueprint_2026.md) | Kế hoạch thực hiện: Phases, Tasks, Test Cases, Tech Stack, CV Keywords |
| [Architecture Design](docs/architecture_design.md) | Sơ đồ kỹ thuật: Component, Sequence, Activity, ERD, Deployment + ADR + SLA |
| [Business Architecture](docs/business_architecture.md) | Business Context: Cơ cấu công ty, phòng ban, luồng dữ liệu, pain points |

---

<p align="center">
  <sub>Built with ❤️ as a Data Engineering portfolio project</sub>
</p>
