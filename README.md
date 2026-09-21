# NY Taxi Data Pipeline — Dockerized PostgreSQL Ingestion

## 1. Project Title & Overview

This project is a minimal, fully containerized **data engineering pipeline** that downloads NYC Yellow Taxi trip-record CSV data, cleans and chunks it with `pandas`, and loads it into a **PostgreSQL** database running in Docker. A **pgAdmin** container is provided alongside Postgres for visual database management, and the whole stack (database + admin UI) is orchestrated with **Docker Compose**.

**Data flow at a glance:**

```
NYC TLC CSV (gzip, HTTPS)
        │
        ▼
Python ingestion container (ingest_data.py)
   pandas chunked read → dtype/date cleanup → SQLAlchemy/psycopg
        │  SQL INSERT over Docker network (host: pgdatabase)
        ▼
PostgreSQL container (pgdatabase)  ──►  pgAdmin container (pgadmin)
   db: ny_taxi                          Web UI: http://localhost:8085
   volume: ny_taxi_postgres_data        volume: pgadmin_data
```

## 2. Architecture & Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Ingestion | Python 3.13, `uv`, `pandas`, `SQLAlchemy`, `psycopg[binary,pool]`, `click`, `tqdm` | Download, transform, and load CSV data in 100k-row chunks |
| Storage | PostgreSQL 18 (`postgres:18` image) | Persists `ny_taxi` database and taxi trip tables |
| Administration | pgAdmin 4 (`dpage/pgadmin4` image) | Browser-based GUI for querying/managing Postgres |
| Containerization | Docker, Docker Compose | Builds reproducible images and orchestrates the stack on a shared network |

**Containers created:** 3 total — `pgdatabase` and `pgadmin` run as persistent Compose services; the ingestion container (built from `pipeline/Dockerfile`, tagged `taxi_ingest:v001`) is run as a one-off (`--rm`) job against the same Docker network.

| Container | Image | Ports (host:container) | Volume | Role |
|---|---|---|---|---|
| `pgdatabase` | `postgres:18` | `5432:5432` | `ny_taxi_postgres_data:/var/lib/postgresql` | Data storage |
| `pgadmin` | `dpage/pgadmin4` | `8085:80` | `pgadmin_data:/var/lib/pgadmin` | DB management UI |
| `taxi_ingest` (ephemeral) | `taxi_ingest:v001` (local build) | none | none | One-off data load job |

Containers discover each other by **service/container name** over the Docker network's built-in DNS — no manual IP configuration is needed. Only pgAdmin's web port is exposed to the host browser; container-to-container traffic (ingestion → Postgres, pgAdmin → Postgres) stays entirely inside the Docker network on port 5432.

## 3. Prerequisites & Environment Setup

- Docker Desktop / Docker Engine + Docker Compose v2 (`docker compose version`)
- [`uv`](https://docs.astral.sh/uv/) for local Python dependency management (`pip install uv`)
- Python 3.13 (managed automatically by `uv`)

**Environment variables** (defaults used throughout this guide — override for production):

| Variable | Default | Used by |
|---|---|---|
| `POSTGRES_USER` | `root` | `pgdatabase` |
| `POSTGRES_PASSWORD` | `root` | `pgdatabase` |
| `POSTGRES_DB` | `ny_taxi` | `pgdatabase` |
| `PGADMIN_DEFAULT_EMAIL` | `admin@admin.com` | `pgadmin` |
| `PGADMIN_DEFAULT_PASSWORD` | `root` | `pgadmin` |

> For any non-local/shared environment, move these out of the compose file into a `.env` file (git-ignored) and never commit real credentials.

## 4. Step-by-Step Deployment & Pipeline Execution

**1. Start the database and pgAdmin:**
```bash
docker compose up -d
docker compose ps
```

**2. Build the ingestion image:**
```bash
cd pipeline
docker build -t taxi_ingest:v001 .
cd ..
```

**3. Find the Compose network name:**
```bash
docker network ls
# look for "<project_dir>_default", e.g. pipeline_default
```

**4. Run the ingestion job on that network:**
```bash
docker run -it --rm \
  --network=<project_dir>_default \
  taxi_ingest:v001 \
    --pg-user=root \
    --pg-pass=root \
    --pg-host=pgdatabase \
    --pg-port=5432 \
    --pg-db=ny_taxi \
    --target-table=yellow_taxi_trips
```

The container exits automatically (`--rm`) once ingestion completes; progress is logged via `tqdm` chunk-by-chunk.

## 5. Connecting pgAdmin to PostgreSQL

1. Open `http://localhost:8085` in a browser.
2. Log in with `PGADMIN_DEFAULT_EMAIL` / `PGADMIN_DEFAULT_PASSWORD` (defaults: `admin@admin.com` / `root`).
3. Right-click **Servers** → **Register** → **Server…**
4. **General** tab — Name: `Local Docker` (any label).
5. **Connection** tab:
   - Host name/address: `pgdatabase` (the Compose service name — **not** `localhost`)
   - Port: `5432`
   - Username: `root`
   - Password: `root`
6. Save. The `ny_taxi` database and its tables should appear in the left-hand tree.

## 6. Data Flow & Verification

**Via pgAdmin:** Query Tool on the `ny_taxi` database:
```sql
SELECT COUNT(*) FROM yellow_taxi_trips;
SELECT * FROM yellow_taxi_trips LIMIT 10;
```

**Via `pgcli` from the host:**
```bash
uv run pgcli -h localhost -p 5432 -u root -d ny_taxi
```
```sql
\dt
SELECT COUNT(*) FROM yellow_taxi_trips;
```

A successful load returns a non-zero row count (~1.37M rows for the January 2021 Yellow Taxi CSV).

## 7. Troubleshooting & Teardown

| Symptom | Likely Cause | Fix |
|---|---|---|
| `could not translate host name "pgdatabase"` | Ingestion container not on the same Docker network as `pgdatabase` | Confirm `--network=<project>_default` matches `docker network ls` output |
| pgAdmin can't connect to server | Used `localhost` instead of the container/service name | Use `pgdatabase` as the Host in the server registration |
| Port `5432` / `8085` already in use | Another local Postgres/pgAdmin instance running | Stop the conflicting process, or change the host-side port in `docker-compose.yaml` |
| Data missing after restart | Volumes were removed | Named volumes (`ny_taxi_postgres_data`, `pgadmin_data`) persist across `docker compose down`; only `down -v` removes them |

**Stop services (keep data):**
```bash
docker compose down
```

**Stop services and wipe all data volumes:**
```bash
docker compose down -v
```

**View logs:**
```bash
docker compose logs -f pgdatabase
docker compose logs -f pgadmin
```
