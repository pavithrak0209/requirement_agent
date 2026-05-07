# Hi, I'm Pavithra 👋

**AI & Data Engineer** — building end-to-end intelligent systems, from document ingestion pipelines to multi-agent LLM orchestration.

Currently working on **DEAH** (Data Engineering Agent Hub): a platform where AI agents extract structured knowledge from unstructured documents and sync directly to Jira.

- 🔭 &nbsp; Building [`requirement_agent`](https://github.com/pavithrak0209/requirement_agent) — TaskFlow AI Agent: upload docs → 9-stage AI extraction → Jira
- 🧠 &nbsp; `claude-sonnet-4-6` via `anthropic` SDK + `claude-agent-sdk`; async LLM orchestration with `asyncio.Semaphore` + exponential backoff
- 🗄️ &nbsp; Production DB: **MySQL** via `PyMySQL` + `cloud-sql-python-connector` on Cloud SQL | SQLite for dev
- ☁️ &nbsp; **Google Cloud Storage**, Cloud SQL, GCP VM deployments
- 🛠️ &nbsp; Python 3.11 · FastAPI · SQLAlchemy 2.x · Alembic · Pydantic v2 · React + Vite · Docker
- 🔗 &nbsp; **Jira Cloud REST API v3** · `httpx` async · nginx reverse proxy

---

## 🛠️ Tech Stack

### AI / LLM
![Claude Sonnet 4.6](https://img.shields.io/badge/Claude_Sonnet_4.6-D97757?style=flat-square&logo=anthropic&logoColor=white)
![Anthropic SDK](https://img.shields.io/badge/anthropic_SDK-191919?style=flat-square&logo=anthropic&logoColor=white)
![Claude Agent SDK](https://img.shields.io/badge/claude--agent--sdk-6E56CF?style=flat-square&logoColor=white)

### Backend
![Python](https://img.shields.io/badge/Python_3.11+-3776AB?style=flat-square&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)
![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy_2.x-D71F00?style=flat-square&logo=sqlalchemy&logoColor=white)
![Alembic](https://img.shields.io/badge/Alembic-6E56CF?style=flat-square&logoColor=white)
![Pydantic v2](https://img.shields.io/badge/Pydantic_v2-E92063?style=flat-square&logo=pydantic&logoColor=white)
![httpx](https://img.shields.io/badge/httpx_async-009688?style=flat-square&logoColor=white)
![Uvicorn](https://img.shields.io/badge/Uvicorn-499848?style=flat-square&logoColor=white)

### Database
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=flat-square&logo=mysql&logoColor=white)
![PyMySQL](https://img.shields.io/badge/PyMySQL-4479A1?style=flat-square&logo=mysql&logoColor=white)
![Cloud SQL](https://img.shields.io/badge/Cloud_SQL_(MySQL)-4285F4?style=flat-square&logo=googlecloud&logoColor=white)
![SQLite](https://img.shields.io/badge/SQLite_(dev)-003B57?style=flat-square&logo=sqlite&logoColor=white)

### Storage & Cloud
![GCS](https://img.shields.io/badge/Google_Cloud_Storage-4285F4?style=flat-square&logo=googlecloud&logoColor=white)
![GCP VM](https://img.shields.io/badge/GCP_VM-4285F4?style=flat-square&logo=googlecloud&logoColor=white)
![cloud-sql-python-connector](https://img.shields.io/badge/cloud--sql--python--connector-4285F4?style=flat-square&logo=googlecloud&logoColor=white)

### Document Parsing
![PyMuPDF](https://img.shields.io/badge/PyMuPDF_(PDF)-FF0000?style=flat-square&logoColor=white)
![python-docx](https://img.shields.io/badge/python--docx_(DOCX)-2B579A?style=flat-square&logo=microsoftword&logoColor=white)

### Frontend & Infra
![React + Vite](https://img.shields.io/badge/React_+_Vite-61DAFB?style=flat-square&logo=react&logoColor=black)
![TailwindCSS](https://img.shields.io/badge/TailwindCSS-06B6D4?style=flat-square&logo=tailwindcss&logoColor=white)
![nginx](https://img.shields.io/badge/nginx-009639?style=flat-square&logo=nginx&logoColor=white)
![Docker](https://img.shields.io/badge/Docker_+_Compose-2496ED?style=flat-square&logo=docker&logoColor=white)
![Jira](https://img.shields.io/badge/Jira_Cloud_API_v3-0052CC?style=flat-square&logo=jira&logoColor=white)

### Testing
![pytest](https://img.shields.io/badge/pytest_+_pytest--asyncio-0A9EDC?style=flat-square&logo=pytest&logoColor=white)
![respx](https://img.shields.io/badge/respx_(httpx_mocking)-6E56CF?style=flat-square&logoColor=white)

---

## 🚀 Featured Project — TaskFlow AI Agent (DEAH · Requirements POD)

> **[github.com/pavithrak0209/requirement_agent](https://github.com/pavithrak0209/requirement_agent)**
>
> Upload a requirements document (PDF, DOCX, TXT, MD, VTT, SRT — up to 50 MB) → a 9-stage AI pipeline powered by `claude-sonnet-4-6` ingests, deduplicates, temporally reasons, and confidence-scores structured tasks → review and inline-edit in the React UI → push Jira tickets with full field mapping in one click.

### 🔬 9-Stage Extraction Pipeline

Located in `core/requirements_pod/services/extraction/`. Entry point: `parse_file()`. Chunk failures log and return `[]` — partial results are always preserved.

| Stage | Name | Detail |
|-------|------|--------|
| 1 | **Input Normalisation** | Strip null bytes, normalise line endings → plain UTF-8 |
| 2 | **Token-aware Chunker** | 2000-token windows, 200-token overlap, no mid-sentence splits (`chunker.py`) |
| 3 | **Parallel LLM Extraction** | `asyncio.Semaphore` (max 5 concurrent), exponential backoff retry (3 attempts) (`llm.py`) |
| 4 | **Local Deduplication** | Jaccard similarity per document, threshold 0.75 (`dedup.py`) |
| 5 | **Global Task Pool** | Flatten all per-doc deduped tasks (`dedup.py`) |
| 6 | **Graph Similarity Merge** | Union-Find with path compression, threshold 0.55 (`merge.py`) |
| 7 | **Temporal Reasoning** | Override-marker & initial-marker detection across document order (`temporal.py`) |
| 8 | **Confidence Scoring** | Extraction score + cluster density + cross-source bonus + AC richness (`scoring.py`) |
| 9 | **Output Normalisation** | Maps to `TaskOut` Pydantic schema → MySQL / SQLite (`output.py`) |

### 🏗️ Architecture

| Layer | Detail |
|-------|--------|
| **LLM abstraction** | `BaseLLMProvider` ABC → `ClaudeProvider` (prod, `claude-agent-sdk`) / `MockProvider` (dev, reads fixture JSON — no API key needed); per-request override via `?llm_provider=claude\|mock` |
| **Storage abstraction** | `BaseStorageProvider` ABC → `GCSProvider` (prod, `asyncio.to_thread()` for blocking I/O) / `LocalProvider` (dev, `./local_storage/`) |
| **Database** | MySQL on Cloud SQL via `cloud-sql-python-connector[pymysql]` (prod) — swap to SQLite with one `DATABASE_URL` change; ORM: `SourceFile` + `Task`; soft-delete via `status=deleted`; `task_id` slug (`SRC-001`) generated via `MAX(task_id)` + retry on `IntegrityError` |
| **Migrations** | Alembic: `001_initial` + `002_add_task_fields` (priority, reporter, sprint, fix_version, story_points, acceptance_criteria) |
| **API** | FastAPI REST `/api/v1` — live SSE extraction progress `{stage, chunks_done, chunks_total, pct}`; JSON structured logging with `request_id` on every line |
| **Jira** | `JiraService.push_task()` + `update_existing_task()` → Jira Cloud REST v3; sends summary, description + AC (ADF), issuetype, priority, `customfield_10016` (story points), fixVersions, sprint as label `sprint:<name>`; writes back `jira_id` + `jira_url`; result includes `action: "created"\|"updated"` |
| **Frontend** | React + Vite SPA — Upload tab (drag-drop + live progress bar) / Browse Files tab (GCS browser, date filter, batch-parse up to 3 files) / Tasks tab (inline edit, collapsible filters, bulk export JSON/CSV/MD, Jira push confirm dialog showing created vs updated count) |
| **Infra** | Docker + `docker-compose`; API `:8000`, UI `:5173` via nginx; `pydantic-settings` — env vars only, no committed secrets |

### 📋 Extracted Task Fields

| Field | Source | Notes |
|-------|--------|-------|
| `task_heading` | AI | Maps to Jira summary (≤80 chars, imperative verb) |
| `description` | AI | Full detail |
| `task_type` | AI | `bug` / `story` / `task` / `subtask` |
| `priority` | AI | `critical` / `high` / `medium` / `low` |
| `story_points` | AI | Fibonacci: 1, 2, 3, 5, 8, 13, 21 |
| `acceptance_criteria` | AI | JSON-encoded `list[str]`, testable criteria |
| `reporter` | AI | Extracted when mentioned in the document |
| `sprint` | AI / manual | Sprint name when mentioned |
| `fix_version` | AI / manual | Release version when mentioned |
| `user_name` | Manual | Assignee — blank by default |
| `jira_id` | Jira | Set after push; `jira_url` also written back |

### 🔌 Key API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/files/upload` | Upload document (multipart, up to 50 MB) |
| `GET` | `/api/v1/files/list?user_name=` | List GCS objects for user |
| `POST` | `/api/v1/files/register` | Find-or-create DB record for GCS file (idempotent) |
| `GET` | `/api/v1/files/find?path=` | Look up DB file record by storage path |
| `DELETE` | `/api/v1/files/storage?path=` | Delete file from storage and DB |
| `POST` | `/api/v1/files/{id}/parse?llm_provider=` | Trigger extraction; override LLM provider per-request |
| `GET` | `/api/v1/files/{id}/progress` | Live SSE progress `{stage, chunks_done, chunks_total, pct}` |
| `GET` | `/api/v1/files/{id}/status` | Poll parse status |
| `GET` | `/api/v1/tasks` | List tasks (`?source_file=&user_name=&status=`) |
| `PATCH` | `/api/v1/tasks/{id}` | Update any task field |
| `DELETE` | `/api/v1/tasks/{id}` | Soft-delete (status → deleted) |
| `POST` | `/api/v1/tasks/export` | Export selected tasks as JSON / CSV / MD |
| `POST` | `/api/v1/tasks/jira-push` | Push to Jira with full field mapping |

---

## 📊 GitHub Stats

![Pavithra's GitHub stats](https://github-readme-stats.vercel.app/api?username=pavithrak0209&show_icons=true&hide_border=true&title_color=6E56CF&icon_color=6E56CF&count_private=true&include_all_commits=true&theme=default)
&nbsp;
![Top Languages](https://github-readme-stats.vercel.app/api/top-langs/?username=pavithrak0209&layout=compact&hide_border=true&title_color=6E56CF&langs_count=6&theme=default)

---

## 📬 Connect

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/k-pavithra/)
&nbsp;
[![Gmail](https://img.shields.io/badge/Gmail-EA4335?style=flat-square&logo=gmail&logoColor=white)](mailto:pavithrakannan0209@gmail.com)
&nbsp;
[![Phone](https://img.shields.io/badge/+1_214_960_8865-25D366?style=flat-square&logo=whatsapp&logoColor=white)](tel:+12149608865)

---

<div align="center">
  <img src="https://komarev.com/ghpvc/?username=pavithrak0209&style=flat-square&color=6E56CF&label=profile+views" />
</div>
