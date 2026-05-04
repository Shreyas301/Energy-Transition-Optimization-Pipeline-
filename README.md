# ⚡ Energy Transition Optimization Pipeline

> An automated, self-correcting data pipeline that ingests renewable energy production data from public APIs, validates it, and stores it in a structured PostgreSQL database — orchestrated by n8n with Python-powered gap detection and backfill logic.

![Python](https://img.shields.io/badge/Python-3.10%2B-blue)
![n8n](https://img.shields.io/badge/n8n-Workflow-orange)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-336791)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED)
![License](https://img.shields.io/badge/License-MIT-green)

---

## 📖 Overview

This project builds a production-grade ETL pipeline that monitors renewable energy production (solar, wind, hydro, nuclear, etc.) across European countries using the [Energy-Charts API](https://api.energy-charts.info/) by Fraunhofer ISE.

The pipeline runs hourly via **n8n** (a low-code workflow automation tool), stores data in **PostgreSQL**, and uses a separate **Python service** to detect and fill gaps in the time-series — providing self-correcting behavior even when the API or pipeline experiences failures.

### Why This Project?

Energy data is critical for understanding the renewable transition, but public APIs are unreliable: they timeout, return malformed data, and have outages. This pipeline demonstrates how to build a robust ingestion system that **assumes failure** and **corrects itself automatically**.

---

## 🏗️ Architecture

```
┌──────────────────┐
│ Energy-Charts    │
│ Public API       │
│ (Fraunhofer ISE) │
└────────┬─────────┘
         │ HTTP GET
         ▼
┌──────────────────┐         ┌──────────────────┐
│       n8n        │────────▶│   PostgreSQL 15  │
│  (Orchestrator)  │  INSERT │   (Data Store)   │
│  - Schedule      │         │  - energy_readings
│  - Retry logic   │         │  - pipeline_logs │
│  - Validation    │         └────────▲─────────┘
└──────────────────┘                  │
                                      │ Gap Detection
                                      │ + Backfill
                            ┌─────────┴─────────┐
                            │  Python Service   │
                            │  (backfill.py)    │
                            │  - psycopg2       │
                            │  - requests       │
                            │  - exponential    │
                            │    backoff retry  │
                            └───────────────────┘
```

---

## ✨ Key Features

- 🔄 **Automated hourly ingestion** via n8n scheduled workflows
- 🛡️ **Self-correcting logic** at four layers:
  - HTTP retry with exponential backoff
  - Null-value filtering for shut-down energy sources
  - Auto-correction of physically impossible negative renewable values
  - Idempotent inserts via PostgreSQL `ON CONFLICT` clauses
- 🔍 **Gap detection** using SQL window functions (`generate_series` + `LEFT JOIN`)
- 📊 **Audit logging** of every pipeline run in `pipeline_logs`
- 🐳 **Fully containerized** with Docker Compose — one command to run everything
- 🔐 **Credentials managed via `.env`** — no secrets in code

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|------------|
| **Orchestration** | n8n (workflow automation) |
| **Data Source** | Energy-Charts public API (Fraunhofer ISE) |
| **Database** | PostgreSQL 15 |
| **Backfill Service** | Python 3.10+ (psycopg2, requests, python-dotenv) |
| **Infrastructure** | Docker, Docker Compose |
| **DB Client** | DBeaver (for manual queries) |

---

## 📁 Repository Structure

```
energy-pipeline/
├── docker-compose.yml          # Infrastructure (PostgreSQL + n8n)
├── Dockerfile.n8n              # Custom n8n image (optional, for Python support)
├── .env.example                # Template for environment variables
├── .gitignore                  # Excludes secrets and venv
├── README.md                   # This file
│
├── sql/
│   └── schema.sql              # Database table definitions
│
├── n8n-workflows/
│   ├── energy-data-pipeline.json   # Main hourly ingestion workflow
│   └── daily-backfill.json         # Daily backfill orchestrator
│
└── python-scripts/
    ├── backfill.py             # Self-correcting gap-filler script
    ├── requirements.txt        # Python dependencies
    ├── .env                    # Local credentials (NOT committed)
    └── backfill.log            # Generated at runtime
```

---

## 🚀 Quick Start

### Prerequisites

Before you begin, install:

- **Docker Desktop** ([download](https://www.docker.com/products/docker-desktop/))
- **Python 3.10+** ([download](https://www.python.org/downloads/))
- **DBeaver Community** ([download](https://dbeaver.io/download/)) — optional but recommended
- **Git** ([download](https://git-scm.com/))

### 1. Clone the Repository

```bash
git clone https://github.com/YOUR-USERNAME/energy-pipeline.git
cd energy-pipeline
```

### 2. Configure Environment Variables

Copy the example file and edit it with your own values:

```bash
cp .env.example .env
```

Open `.env` and update the password (and any other values you want to change):

```env
# PostgreSQL
POSTGRES_USER=energy_admin
POSTGRES_PASSWORD=your_secure_password_here
POSTGRES_DB=energy_db

# n8n
N8N_USER=admin
N8N_PASSWORD=your_n8n_password_here

# API
API_BASE_URL=https://api.energy-charts.info/public_power
DEFAULT_COUNTRY=de
```

Also create `python-scripts/.env`:

```env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=energy_db
DB_USER=energy_admin
DB_PASSWORD=your_secure_password_here
API_BASE_URL=https://api.energy-charts.info/public_power
DEFAULT_COUNTRY=de
```

### 3. Start the Infrastructure

```bash
docker compose up -d
```

This launches:
- PostgreSQL on `localhost:5432`
- n8n on `localhost:5678`

Verify both are running:

```bash
docker ps
```

### 4. Initialize the Database

Connect to PostgreSQL using DBeaver (or any SQL client) with these credentials:

| Field | Value |
|-------|-------|
| Host | `localhost` |
| Port | `5432` |
| Database | `energy_db` |
| User | `energy_admin` |
| Password | (from your `.env`) |

Then run the SQL in `sql/schema.sql`:

```sql
CREATE TABLE energy_readings (
    id SERIAL PRIMARY KEY,
    country_code VARCHAR(10) NOT NULL,
    energy_source VARCHAR(50) NOT NULL,
    production_mwh NUMERIC(12, 2),
    timestamp_utc TIMESTAMP NOT NULL,
    fetched_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    data_quality VARCHAR(20) DEFAULT 'valid',
    UNIQUE(country_code, energy_source, timestamp_utc)
);

CREATE TABLE pipeline_logs (
    id SERIAL PRIMARY KEY,
    run_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(20),
    records_processed INTEGER,
    error_message TEXT
);
```

### 5. Set Up the n8n Workflow

1. Open n8n at [http://localhost:5678](http://localhost:5678)
2. Create your owner account on first launch
3. Click **"Import from File"** in the workflow menu
4. Import `n8n-workflows/energy-data-pipeline.json`
5. Update the PostgreSQL credential in the imported workflow:
   - Host: `postgres` (the Docker container name, NOT `localhost`)
   - Database: `energy_db`
   - User: `energy_admin`
   - Password: (from your `.env`)
6. Click the **"Active"** toggle to enable the schedule

### 6. Set Up the Python Backfill Service

```bash
cd python-scripts

# Create and activate a virtual environment
python -m venv venv

# Windows:
venv\Scripts\activate

# Mac/Linux:
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### 7. Run the Backfill (Optional First Test)

```bash
python backfill.py
```

You should see logs showing the script connecting to the database, scanning for gaps, and either fixing them or reporting "No gaps found."

---

## 🧪 Testing the Self-Correcting Logic

To prove the pipeline heals itself, simulate a data loss and watch it recover:

```sql
-- In DBeaver, delete an hour of data
DELETE FROM energy_readings 
WHERE timestamp_utc >= NOW() - INTERVAL '2 days'
  AND timestamp_utc < NOW() - INTERVAL '2 days' + INTERVAL '1 hour';
```

Then run:

```bash
python python-scripts/backfill.py
```

The script will detect the gap, refetch the data from the API, and insert it back. Verify with:

```sql
SELECT * FROM pipeline_logs ORDER BY run_timestamp DESC LIMIT 5;
```

---

## 📊 Querying the Data

### Latest readings
```sql
SELECT * FROM energy_readings 
ORDER BY timestamp_utc DESC 
LIMIT 20;
```

### Today's energy mix for Germany
```sql
SELECT 
    energy_source, 
    ROUND(SUM(production_mwh)::numeric, 2) AS total_mwh
FROM energy_readings
WHERE country_code = 'DE'
  AND timestamp_utc >= CURRENT_DATE
GROUP BY energy_source
ORDER BY total_mwh DESC;
```

### Pipeline health summary
```sql
SELECT 
    status, 
    COUNT(*) AS runs, 
    SUM(records_processed) AS total_records
FROM pipeline_logs
WHERE run_timestamp >= NOW() - INTERVAL '7 days'
GROUP BY status;
```

### Data quality breakdown
```sql
SELECT 
    data_quality, 
    COUNT(*) AS row_count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) AS pct
FROM energy_readings
GROUP BY data_quality;
```

---

## 🔧 Configuration Reference

### Environment Variables (root `.env`)

| Variable | Description | Default |
|----------|-------------|---------|
| `POSTGRES_USER` | PostgreSQL admin username | `energy_admin` |
| `POSTGRES_PASSWORD` | PostgreSQL password | *(required)* |
| `POSTGRES_DB` | Database name | `energy_db` |
| `N8N_USER` | n8n admin username | `admin` |
| `N8N_PASSWORD` | n8n admin password | *(required)* |

### Environment Variables (`python-scripts/.env`)

| Variable | Description | Default |
|----------|-------------|---------|
| `DB_HOST` | DB host (use `localhost` from host, `postgres` from Docker) | `localhost` |
| `DB_PORT` | DB port | `5432` |
| `DB_NAME` | Database name | `energy_db` |
| `DB_USER` | DB username | `energy_admin` |
| `DB_PASSWORD` | DB password | *(required)* |
| `API_BASE_URL` | Energy-Charts API endpoint | `https://api.energy-charts.info/public_power` |
| `DEFAULT_COUNTRY` | Default country code | `de` |

---

## 🐛 Troubleshooting

| Problem | Solution |
|---------|----------|
| `docker compose up` fails | Ensure Docker Desktop is running |
| n8n can't connect to PostgreSQL | Use `postgres` (not `localhost`) as the host inside n8n |
| Python script: `psycopg2.OperationalError` | Check that PostgreSQL container is running (`docker ps`) |
| n8n workflow errors on Code node | Switch language to JavaScript if Python is unavailable |
| `relation "energy_readings" does not exist` | Re-run `sql/schema.sql` in your SQL client |
| API returns empty data | Try a date range from the last 1–7 days; future dates return nothing |

---

## 🗺️ Roadmap

- [x] Hourly automated ingestion
- [x] Self-correcting backfill in Python
- [x] Idempotent database inserts
- [x] Audit logging
- [ ] Slack/email alerting on pipeline failures
- [ ] Grafana dashboard for visualization
- [ ] Multi-country parallel fetching
- [ ] Cloud deployment (AWS/GCP)
- [ ] Anomaly detection with statistical thresholds

---

## 📚 What I Learned Building This

- **Idempotency matters.** Without `ON CONFLICT DO NOTHING`, re-running the pipeline corrupted the database. With it, the pipeline can be safely retried any number of times.
- **Self-correction beats perfect prevention.** Rather than trying to prevent every API failure, designing the system to detect and recover from failures was simpler and more reliable.
- **The "host" gotcha in Docker.** Services inside Docker reach each other by container name (`postgres`), but the host machine reaches them via `localhost`. This caused several hours of debugging until I understood the network model.
- **SQL `generate_series` is magical.** Detecting gaps in time-series data with three lines of SQL is far cleaner than equivalent Python.
- **Logs are infrastructure.** The `pipeline_logs` table turned out to be more useful than I expected — every debugging session started there.

---

## 📝 License

MIT License — feel free to fork, modify, and use this as a learning resource.

---

## 🙏 Acknowledgments

- **[Fraunhofer ISE](https://www.energy-charts.info/)** for providing the free, well-documented Energy-Charts API
- **[n8n](https://n8n.io/)** for an excellent open-source automation platform
- **[PostgreSQL](https://www.postgresql.org/)** for being PostgreSQL

---

## 👤 Author

**Shreyas Sandbhor**

- GitHub: [@your-username](https://github.com/your-username)
- LinkedIn: [your-profile](https://linkedin.com/in/your-profile)
- Email: shreyassandbhor301@gmail.com

---

*If you found this project useful, please ⭐ the repository!*
