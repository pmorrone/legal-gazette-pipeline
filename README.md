# Legal Gazette Pipeline

A small, end-to-end data engineering project over **Argentina's Boletín Oficial** (the official gazette where every national decree, resolution and regulation is published).

The goal is to turn daily, unstructured legal publications into tested, documented, analytics-ready tables, using a modern data stack.

## Architecture

```
boletinoficial.gob.ar
        │  scraping (Python, requests + BeautifulSoup)
        ▼
  Bronze  ── raw HTML + extracted metadata, Parquet on MinIO (S3-compatible)
        │
        ▼
  Silver / Gold ── dbt models on DuckDB, with tests and docs      (step 3)
        │
        ▼
  Dashboard ── Streamlit                                          (step 4)

  Orchestration: Dagster (step 4) · Packaging: Docker Compose · CI: GitHub Actions
```

## Status

- [x] **Step 1** – Source analysis and scope
- [x] **Step 2** – Ingestion into the bronze layer *(in progress)*
- [ ] **Step 3** – dbt models (staging → marts), tests, docs
- [ ] **Step 4** – Dagster orchestration, Streamlit dashboard, optional LLM enrichment

## Data source notes

The Boletín Oficial has no public API, so ingestion scrapes the website:

| Page | URL pattern |
|---|---|
| Daily section listing | `/seccion/primera/YYYYMMDD` |
| Single publication | `/detalleAviso/primera/<id>/YYYYMMDD` |

Each publication page contains the full text in HTML plus the issuing body (organismo), norm type and number (e.g. *Decreto 875/2026*), the GDE identifier (e.g. `DECTO-2026-875-APN-PTE`) and the publication date. References to other norms appear only as plain text.

Scope of the MVP: **Primera Sección** (legislation), last three months plus daily increments. The scraper waits between requests and retries with backoff to keep the load on the site low.

The site's `robots.txt` only disallows the **Segunda Sección** (companies and court notices, which contain personal data). The scraper refuses to request it.

## Bronze layout

```
s3://lake/bronze/boletin/seccion=primera/fecha=YYYY-MM-DD/avisos.parquet
```

One file per edition day. Re-running a day overwrites its partition, so loads are idempotent. The raw HTML is kept, so the parsing rules can change later without scraping again.

## Running it

Requirements: Docker and Python 3.11+.

```bash
cp .env.example .env
docker compose up -d                 # MinIO on :9000, console on http://localhost:9001

python -m venv .venv && source .venv/bin/activate
pip install -e '.[dev]'

gazette-ingest --date 2026-11-19                   # one day
gazette-ingest --from 2026-08-20 --to 2026-11-19   # backfill
gazette-ingest --date 2026-11-19 --local-dir data  # write locally, no MinIO

pytest                                                  # unit tests (no network)
```
