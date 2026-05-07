<div align="center">

[![Typing SVG](https://readme-typing-svg.demolab.com?font=Fira+Code&size=22&pause=1000&color=6E56CF&center=true&vCenter=true&width=650&lines=Hi%2C+I'm+Pavithra+%F0%9F%91%8B;AI+%26+Data+Engineer;Building+LLM-powered+agents+%26+pipelines)](https://git.io/typing-svg)

</div>

---

### About me

I'm an **AI & Data Engineer** building end-to-end intelligent systems — from document ingestion pipelines to multi-agent LLM orchestration. Currently working on **DEAH** (Data Engineering Agent Hub), a platform where AI agents extract structured knowledge from unstructured documents and sync directly to Jira.

- 🔭 &nbsp; Building [`requirement_agent`](https://github.com/pavithrak0209/requirement_agent) — TaskFlow AI Agent: upload docs → 9-stage AI extraction pipeline → Jira
- 🧠 &nbsp; Working with `claude-sonnet-4-6` via `anthropic` SDK + `claude-agent-sdk`, async LLM orchestration with `asyncio.Semaphore` + exponential backoff retry
- 🗄️ &nbsp; Production DB: **MySQL** via `PyMySQL` + `cloud-sql-python-connector` on Cloud SQL | SQLite for local dev
- ☁️ &nbsp; Cloud: **Google Cloud Storage** (GCS), Cloud SQL, GCP VM deployments; GCS path: `{GCS_PREFIX}/{user}/{YYYY-MM-DD}/{filename}_{yymmddHHMMSS}.ext`
- 🛠️ &nbsp; Stack: Python 3.11 · FastAPI · SQLAlchemy 2.x · Alembic · Pydantic v2 · React + Vite · Docker
- 🔗 &nbsp; Integrations: **Jira Cloud REST API v3** · `httpx` async client · nginx reverse proxy

---

### Tech stack

**AI / LLM**

![Claude](https://img.shields.io/badge/Claude_Sonnet_4.6-D97757?style=flat-square&logo=anthropic&logoColor=white)
![Anthropic SDK](https://img.shields.io/badge/anthropic_SDK-191919?style=flat-square&logo=anthropic&logoColor=white)
![Claude Agent SDK](https://img.shields.io/badge/claude--agent--sdk-6E56CF?style=flat-square&logoColor=white)

**Backend**

![Python](https://img.shields.io/badge/Python_3.11+-3776AB?style=flat-square&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?style=flat-square&logo=fastapi&logoColor=white)
![SQLAlchemy](https://img.shields.io/badge/SQLAlchemy_2.x-D71F00?style=flat-square&logo=sqlalchemy&logoColor=white)
![Alembic](https://img.shields.io/badge/Alembic-6E56CF?style=flat-square&logoColor=white)
![Pydantic](https://img.shields.io/badge/Pydantic_v2-E92063?style=flat-square&logo=pydantic&logoColor=white)
![httpx](https://img.shields.io/badge/httpx_async-009688?style=flat-square&logoColor=white)
![uvicorn](https://img.shields.io/badge/Uvicorn-499848?style=flat-square&logoColor=white)

**Database**

![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=flat-square&logo=mysql&logoColor=white)
![PyMySQL](https://img.shields.io/badge/PyMySQL-4479A1?style=flat-square&logo=mysql&logoColor=white)
![Cloud SQL](https://img.shields.io/badge/Cloud_SQL_(MySQL)-4285F4?style=flat-square&logo=googlecloud&logoColor=white)
![SQLite](https://img.shields.io/badge/SQLite_(dev)-003B57?style=flat-square&logo=sqlite&logoColor=white)

**Storage & Cloud**

![GCS](https://img.shields.io/badge/Google_Cloud_Storage-4285F4?style=flat-square&logo=googlecloud&logoColor=white)
![GCP](https://img.shields.io/badge/GCP_VM-4285F4?style=flat-square&logo=googlecloud&logoColor=white)
![Cloud SQL Connector](https://img.shields.io/badge/cloud--sql--python--connector-4285F4?style=flat-square&logo=googlecloud&logoColor=white)

**Document Parsing**

![PyMuPDF](https://img.shields.io/badge/PyMuPDF_(PDF)-FF0000?style=flat-square&logoColor=white)
![python-docx](https://img.shields.io/badge/python--docx_(DOCX)-2B579A?style=flat-square&logo=microsoftword&logoColor=white)

**Frontend & Infra**

![React](https://img.shields.io/badge/React_+_Vite-61DAFB?style=flat-square&logo=react&logoColor=black)
![TailwindCSS](https://img.shields.io/badge/TailwindCSS-06B6D4?style=flat-square&logo=tailwindcss&logoColor=white)
![nginx](https://img.shields.io/badge/nginx-009639?style=flat-square&logo=nginx&logoColor=white)
![Docker](https://img.shields.io/badge/Docker_+_Compose-2496ED?style=flat-square&logo=docker&logoColor=white)
![Jira](https://img.shields.io/badge/Jira_Cloud_API_v3-0052CC?style=flat-square&logo=jira&logoColor=white)

**Testing**

![pytest](https://img.shields.io/badge/pytest_+_pytest--asyncio-0A9EDC?style=flat-square&logo=pytest&logoColor=white)
![respx](https://img.shields.io/badge/respx_(httpx_mocking)-6E56CF?style=flat-square&logoColor=white)

---

### Featured project — TaskFlow AI Agent (DEAH · Requirements POD)

<a href="https://github.com/pavithrak0209/requirement_agent">
  <img align="center" src="https://github-readme-stats.vercel.app/api/pin/?username=pavithrak0209&repo=requirement_agent&theme=default&hide_border=true&title_color=6E56CF&icon_color=6E56CF" />
</a>

<br/><br/>

**What it does:** Upload a requirements document (PDF, DOCX, TXT, MD, VTT, SRT, up to 50 MB) → a 9-stage AI pipeline powered by `claude-sonnet-4-6` ingests, deduplicates, temporally reasons, and confidence-scores structured tasks → review, inline-edit, and bulk-manage in the React UI → push Jira tickets with full field mapping in one click.

#### 🔬 9-stage extraction pipeline (`core/requirements_pod/services/extraction/`)

```
Stage 1 │ Input Normalisation        strip null bytes, normalise line endings → plain UTF-8
Stage 2 │ Token-aware Chunker        2000-token windows, 200-token overlap, no mid-sentence splits
Stage 3 │ Parallel LLM Extraction    asyncio.Semaphore (max 5), exponential backoff retry (3 attempts)
Stage 4 │ Local Dedup                Jaccard similarity per document (threshold 0.75)
Stage 5 │ Global Task Pool           flatten all per-doc deduped tasks
Stage 6 │ Graph Similarity Merge     Union-Find with path compression (threshold 0.55)
Stage 7 │ Temporal Reasoning         override-marker & initial-marker detection across doc order
Stage 8 │ Confidence Scoring         extraction score + cluster density + cross-source bonus + AC richness
Stage 9 │ Output Normalisation       → TaskOut Pydantic schema → MySQL / SQLite
```

Entry point: `parse_file()` — raises `RuntimeError` for empty/unreadable files; chunk failures log and return `[]` (partial results always preserved).

#### 🏗️ Architecture highlights

| Layer | Detail |
|---|---|
| **LLM** | `BaseLLMProvider` ABC → `ClaudeProvider` (prod, `claude-agent-sdk`) / `MockProvider` (dev, reads fixture JSON — no API key needed); per-request override via `?llm_provider=claude\|mock` |
| **Storage** | `BaseStorageProvider` ABC → `GCSProvider` (prod, blocking I/O via `asyncio.to_thread()`) / `LocalProvider` (dev, `./local_storage/`) |
| **Database** | MySQL on Cloud SQL via `cloud-sql-python-connector[pymysql]` (prod) — swap to SQLite with one `DATABASE_URL` change; ORM: `SourceFile` + `Task` models; soft-delete via `status=deleted`; `task_id` slug (`SRC-001`) generated via `MAX(task_id)` + retry on `IntegrityError` |
| **Migrations** | Alembic: `001_initial` + `002_add_task_fields` (priority, reporter, sprint, fix_version, story_points, acceptance_criteria) |
| **API** | FastAPI REST `/api/v1` — live SSE extraction progress `{stage, chunks_done, chunks_total, pct}`; JSON structured logging with `request_id` on every line |
| **Jira** | `JiraService.push_task()` + `update_existing_task()` → Jira Cloud REST v3; sends summary, description+AC (ADF), issuetype, priority, `customfield_10016` (story points), fixVersions, sprint as label `sprint:<name>`; writes back `jira_id` + `jira_url`; result includes `action: "created"\|"updated"` |
| **Frontend** | React + Vite SPA — Upload tab (drag-drop + live progress bar) / Browse Files tab (GCS browser, date filter, batch-parse up to 3 files) / Tasks tab (inline edit, collapsible filters, bulk export JSON/CSV/MD, Jira push confirm dialog) |
| **Infra** | Docker + `docker-compose`; API `:8000`, UI `:5173` via nginx; `pydantic-settings` — env vars only, no committed secrets |

#### 📋 Extracted task fields

`task_heading` · `description` · `task_type` (bug/story/task/subtask) · `priority` (critical/high/medium/low) · `story_points` (Fibonacci 1–21) · `acceptance_criteria` (JSON list) · `reporter` · `sprint` · `fix_version` · `user_name` (assignee) · `jira_id`

---

### GitHub stats

<div align="center">

<img height="160" src="https://github-readme-stats.vercel.app/api?username=pavithrak0209&show_icons=true&hide_border=true&title_color=6E56CF&icon_color=6E56CF&count_private=true&include_all_commits=true" />
&nbsp;&nbsp;
<img height="160" src="https://github-readme-stats.vercel.app/api/top-langs/?username=pavithrak0209&layout=compact&hide_border=true&title_color=6E56CF&langs_count=6" />

</div>

---

### Connect

<a href="https://www.linkedin.com/in/k-pavithra/">
  <img src="https://img.shields.io/badge/LinkedIn-0A66C2?style=flat-square&logo=linkedin&logoColor=white" />
</a>
&nbsp;
<a href="mailto:pavithrakannan0209@gmail.com">
  <img src="https://img.shields.io/badge/Gmail-EA4335?style=flat-square&logo=gmail&logoColor=white" />
</a>
&nbsp;
<a href="tel:+12149608865">
  <img src="https://img.shields.io/badge/+1_214_960_8865-25D366?style=flat-square&logo=whatsapp&logoColor=white" />
</a>

---

<div align="center">
  <img src="https://komarev.com/ghpvc/?username=pavithrak0209&style=flat-square&color=6E56CF&label=profile+views" />
</div>
