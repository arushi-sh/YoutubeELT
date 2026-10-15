# 📺 YouTube ELT Pipeline

An end-to-end **ELT data pipeline** that pulls video analytics for a YouTube channel from the **YouTube Data API v3**, loads them into a **PostgreSQL** data warehouse, transforms them, and validates the data with **Soda** — all orchestrated by **Apache Airflow** and shipped with a **GitHub Actions CI/CD** pipeline.

*Example analysis performed on the MrBeast YouTube channel.*

![Python](https://img.shields.io/badge/Python-3776AB?style=flat-square&logo=python&logoColor=white)
![Apache Airflow](https://img.shields.io/badge/Airflow-017CEE?style=flat-square&logo=apacheairflow&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat-square&logo=redis&logoColor=white)
![Soda](https://img.shields.io/badge/Soda-Data%20Quality-6E44FF?style=flat-square)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-2088FF?style=flat-square&logo=githubactions&logoColor=white)
![pytest](https://img.shields.io/badge/pytest-0A9EDC?style=flat-square&logo=pytest&logoColor=white)

---

## 🏗️ Architecture

```
 YouTube Data API v3
         │
         ▼
┌─────────────────────┐   triggers   ┌─────────────────────┐   triggers   ┌─────────────────────┐
│ DAG 1: produce_json │ ───────────▶ │ DAG 2: update_db    │ ───────────▶ │ DAG 3: data_quality │
│ • get uploads list  │              │ • staging schema    │              │ • Soda scan staging │
│ • fetch video IDs   │              │   (raw load)        │              │ • Soda scan core    │
│ • fetch stats       │              │ • core schema       │              │                     │
│ • save JSON         │              │   (transformed)     │              │                     │
└─────────────────────┘              └─────────────────────┘              └─────────────────────┘
   daily @ 14:00 IST                        PostgreSQL data warehouse
```

Airflow runs with the **CeleryExecutor** (Redis broker, Postgres metadata DB) in Docker Compose.

---

## 🔄 Pipeline

### 1. Extract — `produce_json`
- Looks up the channel's **uploads playlist** from its handle
- Pages through the playlist (50 per page) to collect every **video ID**
- Fetches `snippet`, `contentDetails` and `statistics` for the videos **in batches**
- Saves the raw result as `data/YT_data_<date>.json`
- Scheduled daily at **14:00 IST**, then triggers the next DAG

### 2. Load & Transform — `update_db`
- **Staging schema** — raw data is upserted: new videos inserted, existing ones updated, removed ones deleted
- **Core schema** — cleaned data:
  - ISO-8601 durations (e.g. `PT10M30S`) parsed into a proper `TIME`
  - each video tagged as **`Shorts`** (≤ 60s) or **`Normal`**
- Triggers the data-quality DAG when done

### 3. Data quality — `data_quality`
**Soda** checks run on both the staging and core layers:
- no missing `Video_ID`
- no duplicate `Video_ID`
- `Likes_Count` never greater than `Video_Views`
- `Comments_Count` never greater than `Video_Views`

---

## ✅ Testing & CI/CD

**Tests (pytest)**
- Unit tests — Airflow variables, Postgres connection config, and a **DAG integrity** check (all DAGs import without errors)
- Integration tests — real YouTube API response and a real Postgres connection

**GitHub Actions** (`.github/workflows/ci-cd_yt-elt.yaml`) runs on pushes to `main` / `feature/*` and on PRs:
1. **Build & push** the custom Airflow image to Docker Hub (only when `Dockerfile` / `requirements.txt` change)
2. Start the full stack with `docker compose up`
3. Run **unit + integration tests** inside the Airflow worker
4. Run **end-to-end tests** by triggering all three DAGs
5. Tear the stack down

All credentials come from GitHub **secrets** and **variables** — nothing is hard-coded.

---

## 📁 Project structure

```
YoutubeELT/
├── dags/
│   ├── main.py                 # the 3 DAGs and how they chain
│   ├── api/video_stats.py      # YouTube API extraction tasks
│   ├── datawarehouse/          # staging/core load, upserts, transformations
│   └── dataquality/soda.py     # Soda scan task
├── include/soda/               # Soda data source config + checks
├── tests/                      # pytest unit & integration tests
├── docker/postgres/            # init script for multiple databases
├── docker-compose.yaml         # Airflow (Celery) + Postgres + Redis
├── dockerfile                  # custom Airflow image
└── .github/workflows/          # CI/CD pipeline
```

---

## 🚀 Running locally

1. Create a `.env` file in the project root with your values (see the variables used in `docker-compose.yaml` and the CI workflow) — at minimum your YouTube `API_KEY`, `CHANNEL_HANDLE`, database credentials and Airflow `FERNET_KEY`.
2. Start everything:

   ```bash
   docker compose up -d
   ```

3. Open Airflow at **http://localhost:8080**, enable the `produce_json` DAG and trigger it — the other two DAGs run automatically after it.

Run the tests inside the worker:

```bash
docker exec -it airflow-worker pytest tests/ -v
```
