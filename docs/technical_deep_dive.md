# TaskFlow AI — Requirements Agent
## Technical Deep Dive

> This document covers both the plain English explanation of how the system works and the technical implementation details. Each section includes a simple explanation followed by the technical specifics.

---

### What Is This System?

**In simple terms:**
Think of this agent as a very smart assistant that reads your meeting transcripts or documents, picks out every task and requirement that was discussed, organises them neatly, and creates Jira tickets for you — all automatically. What a human would take an hour to do manually, this does in under 30 seconds.

**Technically:**
TaskFlow AI is a full-stack web application. The user uploads a document through a React frontend. The backend (FastAPI) runs a 9-stage AI pipeline powered by Anthropic's Claude model to extract structured tasks. Those tasks are stored in a MySQL database and can be pushed to Jira via the REST API.

---

### System Architecture Overview

**In simple terms:**
The system has three main parts — a website the user interacts with, a backend server that does all the thinking, and a database that stores everything.

```
Browser (React + Vite)          ← What the user sees and clicks
        │
        │  HTTP / REST
        ▼
FastAPI Backend  (Python, async) ← The brain — processes documents, talks to Claude AI
        │
        ├── Extraction Pipeline  (9 stages, asyncio)
        │         └── Anthropic Claude SDK  ← The AI that reads and understands the document
        │
        ├── Database  (MySQL on Google Cloud SQL)  ← Stores all files and tasks
        │
        ├── Storage   (Google Cloud Storage)  ← Stores the uploaded documents
        │
        └── Jira      (Jira Cloud REST API v3)  ← Creates/updates Jira tickets
```

---

### Tech Stack

**In simple terms:**
These are the tools and technologies used to build the system.

| Layer | Technology | What It Does in Plain English |
|-------|-----------|-------------------------------|
| Frontend | React 18, Vite, TailwindCSS | The website/UI the user sees |
| Backend | FastAPI, Python 3.11, asyncio | The server that handles all logic |
| AI Model | Anthropic Claude (claude-sonnet) via SDK | The AI brain that reads documents |
| Database | MySQL on Cloud SQL | Stores all tasks and file records |
| DB Migrations | Alembic | Manages database structure changes safely |
| Storage | Google Cloud Storage (GCS) | Cloud file storage for uploaded documents |
| Jira Integration | Jira Cloud REST API v3 | Connects to Jira to create/update tickets |
| Testing | pytest, pytest-asyncio, respx | Automated tests to verify everything works |

---

### Repository Structure

**In simple terms:**
This is how the code is organised in folders. Think of it like a filing cabinet — each drawer has a specific purpose.

```
DEAH/
├── core/
│   ├── common/                     # Shared tools reused across all agents
│   │   ├── llm/                    # Everything related to talking to the AI model
│   │   ├── storage/                # Everything related to saving/reading files
│   │   ├── scrum/                  # Everything related to Jira
│   │   └── database/               # Database connection setup
│   └── requirements_pod/           # The Requirements Agent (main agent)
│       ├── main.py                 # Starting point — launches the backend server
│       ├── config.py               # All configuration settings (API keys, URLs, etc.)
│       ├── api/v1/                 # The API endpoints the frontend calls
│       ├── db/                     # Database models and queries
│       ├── schemas/                # Data structures / shapes of data
│       ├── services/
│       │   └── extraction/         # The 9-stage AI pipeline (the core logic)
│       └── migrations/             # Scripts to update the database structure
└── webapp/
    └── requirements_pod/           # The React website (frontend)
        └── src/
            ├── App.jsx             # Main page of the website
            ├── api/client.js       # How the frontend talks to the backend
            └── components/         # Individual UI components (buttons, tables, modals)
```

---

### Backend — FastAPI Entry Point

**In simple terms:**
This is the main "door" into the backend system. When the server starts up, it prepares the database and storage so everything is ready before any user request comes in. Every request gets a unique ID so we can trace it if something goes wrong.

**Technically:**
`core/requirements_pod/main.py` — FastAPI app with async lifespan context.
- On startup: initialises DB tables, creates local storage directory
- JSON structured logging with per-request UUIDs (for tracing)
- CORS configured via `CORS_ORIGINS` setting (default `*`)
- Three API routers: `/api/v1/health`, `/api/v1/files`, `/api/v1/tasks`

---

### The 9-Stage Extraction Pipeline

**In simple terms:**
When you upload a document, the system does not just hand it straight to the AI. It processes it through 9 carefully designed steps — like an assembly line — to make sure the output is accurate, complete, and free of duplicates.

**Technically:**
Entry point: `parse_file()` in `core/requirements_pod/services/extraction/pipeline.py`. The pipeline runs fully async. Progress is tracked in an in-memory dict keyed by `file_id` and streamed to the frontend via a polling endpoint.

---

#### Stage 1 — Normalise
**File:** `pipeline.py` → `_decode_file()`

**In simple terms:**
The first step is just reading the file. Since documents come in many formats (PDF, Word, transcript files), the system converts everything into plain text so the AI can read it consistently.

**Technically:**
Decodes raw bytes into plain text based on file extension:
- `.pdf` → PyMuPDF (`fitz`) extracts text page by page
- `.docx` → `python-docx` joins all paragraph text
- `.vtt` / `.srt` → strips timecodes and speaker labels, extracts clean dialogue
- `.txt` / `.md` → UTF-8 decode

Raises `RuntimeError` for empty or unreadable files.

---

#### Stage 2 — Token-Aware Chunking
**File:** `chunker.py`

**In simple terms:**
The AI can only read a certain amount of text at once — like how a person can only focus on a few pages at a time. So the system cuts the document into smaller overlapping sections. The overlap (200 tokens) is important — it ensures that if a requirement is mentioned at the end of one section and the beginning of the next, it is not missed.

**Technically:**
Splits the normalised text into overlapping windows:
- **Window size:** 2000 tokens (configurable via `LLM_CHUNK_SIZE` in config)
- **Overlap:** 200 tokens (configurable via `LLM_CHUNK_OVERLAP`)

```python
def chunk_text(text: str, max_tokens: int = 2000, overlap: int = 200) -> list[str]:
```

---

#### Stage 3 — Parallel LLM Extraction
**File:** `llm.py`

**In simple terms:**
Now the AI reads each section simultaneously — not one by one, but all at the same time (in parallel). This is what makes extraction fast. For each section, Claude is given a specific instruction: "Find every requirement, task, bug, or action item in this text and return them in a structured format." If Claude fails or is too busy, the system automatically retries with a wait in between.

**Technically:**
Each chunk is sent to Claude in parallel using `asyncio.gather` with a `Semaphore` to cap concurrent requests.

The system prompt instructs Claude to return structured JSON with fields: `task_heading`, `description`, `task_type`, `priority`, `story_points`, `acceptance_criteria`, `reporter`, `sprint`, `fix_version`.

Retry logic uses **exponential backoff** — retried up to 3 times before being skipped (logged, not fatal).

```python
async def extract_all(chunks, provider, semaphore) -> list[list[RawTask]]:
```

---

#### Stage 4 — Local Dedup (Jaccard Similarity)
**File:** `dedup.py`

**In simple terms:**
Because sections overlap, the AI might extract the same task twice from two nearby sections. This step compares tasks from the same section and removes near-identical ones, keeping only the most complete version. It measures how similar two task titles are by comparing the words they share.

**Technically:**
Duplicates within each chunk are removed using **Jaccard similarity** on task heading tokens:

```
similarity = |set_A ∩ set_B| / |set_A ∪ set_B|
```

If two tasks from the same chunk exceed the similarity threshold, the one with fewer fields populated is dropped.

---

#### Stage 5 — Global Task Pool
**File:** `dedup.py` → `build_global_pool()`

**In simple terms:**
After cleaning up duplicates within each section, all the remaining tasks from all sections are gathered into one big combined list — ready to be further merged across sections.

**Technically:**
All per-chunk deduped tasks are flattened into a single global task pool ready for cross-chunk merging.

---

#### Stage 6 — Graph Merge (Union-Find)
**File:** `merge.py`

**In simple terms:**
Even after the per-section cleanup, the same requirement might still appear in section 3 and section 7 worded slightly differently. This step treats tasks like a network — if two tasks are similar enough, they are connected. Then all connected tasks are merged into one final task, taking the best and most complete details from each version.

**Technically:**
Uses a **Union-Find (Disjoint Set Union)** data structure:
- Builds a similarity graph: edge between two tasks if Jaccard similarity > threshold
- Union-Find groups connected tasks into components
- Each component is merged into one canonical task, taking the most complete field values

---

#### Stage 7 — Temporal Reasoning
**File:** `temporal.py`

**In simple terms:**
In real meetings, people change their minds. Someone might say "Let's make this high priority" and then ten minutes later say "Actually, let's bump that to critical." This step is smart enough to understand that the second statement is an update to the first — not a new separate requirement. So instead of creating two tickets, it updates the original one.

**Technically:**
- Detects **override markers** — phrases like "actually", "change that to", "update the previous", "forget what I said"
- When detected, links the new task to the original and applies the override
- Detects **initial markers** — phrases like "initially", "as discussed earlier" — and marks those tasks as superseded

```python
class TemporalTask:
    task: RawTask
    overrides: str | None    # ID of task this overrides
    marker: str | None       # detected trigger phrase
```

---

#### Stage 8 — Confidence Scoring
**File:** `scoring.py`

**In simple terms:**
Not every task extracted will be perfectly complete. Some tasks might be missing a description or acceptance criteria because the document did not mention them. This step gives each task a score from 0 to 1 — how confident the system is that the task is well-defined and ready for Jira. Low scores are flagged in the UI so humans know which ones need attention.

**Technically:**
Each task is scored 0.0 – 1.0 based on field completeness:

| Field | Weight |
|-------|--------|
| task_heading | 0.20 |
| description | 0.25 |
| acceptance_criteria | 0.25 |
| task_type | 0.10 |
| priority | 0.10 |
| story_points | 0.10 |

Score displayed color coded in the UI:
- **Green** ≥ 0.9 — ready to push
- **Blue** ≥ 0.7 — mostly complete
- **Amber** ≥ 0.4 — review recommended
- **Red** < 0.4 — needs human attention

---

#### Stage 9 — Normalise to TaskOut and Save
**File:** `output.py`, `repository.py`

**In simple terms:**
The final step takes all the processed tasks and saves them to the database in a clean, consistent format. Each task gets a unique ID like `SRC-001`. The system handles the case where multiple documents are being processed at the same time — it makes sure no two tasks accidentally get the same ID.

**Technically:**
Converts pipeline output to `TaskOut` Pydantic schema, then persists to DB via the repository layer. `task_id` is generated using `MAX(task_id) + 1` with retry on `IntegrityError` (up to 5 attempts) to handle concurrent uploads safely.

---

### LLM Abstraction Layer

`core/common/llm/`

**In simple terms:**
The system is built so that the AI model can be swapped out without rewriting the whole pipeline. Today it uses Claude. Tomorrow it could use a different model — only one setting needs to change. There is also a "Mock" mode that returns fake data for testing, so developers can test everything without needing an API key or spending money on AI calls.

**Technically:**
```
BaseLLMProvider (ABC)
    ├── extract_tasks(text, prompt) -> list[dict]
    └── call_raw(system, user) -> str

ClaudeProvider     →  Anthropic SDK (real extraction)
MockProvider       →  reads tests/fixtures/mock_llm_response.json
```

The LLM provider is selected via `LLM_PROVIDER` in `config.env`, or overridden per-request via `?llm_provider=claude|mock` query param.

---

### Storage Abstraction Layer

`core/common/storage/`

**In simple terms:**
Uploaded documents need to be stored somewhere. In production, files go to Google Cloud Storage (a secure cloud file system). During local development, files are just saved on the developer's own machine. The code is written so that switching between the two requires no changes — just a config setting.

**Technically:**
```
BaseStorageProvider (ABC)
    ├── upload(file, path) -> str
    ├── download(path) -> bytes
    └── list(prefix) -> list[str]

LocalProvider   →  ./local_storage/  (dev)
GCSProvider     →  Google Cloud Storage (prod)
```

All blocking I/O is wrapped in `asyncio.to_thread()` so it does not block the async event loop.

GCS path format: `{GCS_PREFIX}/{user}/{YYYY-MM-DD}/{filename}_{yymmddHHMMSS}.{ext}`

---

### Database Layer

`core/requirements_pod/db/`

**In simple terms:**
The database is like a structured spreadsheet that stores two types of records — the files that were uploaded, and the tasks that were extracted from them. Every file and task gets a unique ID. When a task is "deleted" by the user, it is not actually removed — it is just marked as deleted so the history is preserved.

**Technically:**
**Database:** MySQL on **Google Cloud SQL** (`verizon-data:us-central1:mysql-druid-metadatastore`), connected via the **Cloud SQL Python Connector** using PyMySQL. Falls back to direct TCP on the VM if the connector is unavailable.

Two ORM models (SQLAlchemy `DeclarativeBase`):

**SourceFile** — table: `req_agent_input`

Stores information about each uploaded document.

| Column | What It Stores |
|--------|---------------|
| `id` | Unique identifier (UUID) |
| `filename` | Original file name |
| `file_path` | Where the file is stored (GCS path) |
| `storage_location` | gcs or local |
| `uploaded_by` | Who uploaded it |
| `upload_time` | When it was uploaded |
| `status` | uploaded / parsed / error |
| `file_size` | Size of the file |
| `mime_type` | File type (pdf, docx, etc.) |
| `coverage_gaps` | JSON gap analysis result |

**Task** — table: `req_agent_tasks`

Stores each extracted task/requirement.

| Column | What It Stores |
|--------|---------------|
| `id` | Unique identifier (UUID) |
| `task_id` | Human-readable ID e.g. `SRC-001` |
| `task_heading` | Title of the task |
| `description` | Full description |
| `task_type` | bug / story / task / subtask |
| `priority` | critical / high / medium / low |
| `story_points` | Effort estimate (Fibonacci) |
| `acceptance_criteria` | Done conditions (JSON list) |
| `reporter` | Person who raised it |
| `assignee` | Person assigned to do it |
| `sprint` | Sprint name |
| `fix_version` | Release version |
| `start_date` / `due_date` | Dates if mentioned |
| `schedule_interval` | hourly / daily / weekly / on-demand |
| `confidence_score` | AI confidence 0.0–1.0 |
| `jira_id` | Set after Jira push |
| `jira_url` | Link to the Jira ticket |
| `status` | extracted / modified / pushed / deleted |
| `source_file_id` | Links back to the uploaded file |
| `raw_llm_json` | Raw AI output (for debugging) |
| `gap_report` | Gap analysis JSON |

Soft delete: setting `status = deleted` keeps the record in the DB for audit purposes.

Migrations managed by **Alembic** — database schema changes are versioned and applied safely without data loss.

---

### Jira Integration

`core/common/scrum/jira.py`

**In simple terms:**
When you click "Push to Jira", the system calls Jira's API directly. It checks whether the task already exists in Jira (was it pushed before?) and either creates a new ticket or updates the existing one. All the details — title, description, acceptance criteria, priority, story points — are sent across in Jira's required format. After a successful push, the Jira ticket ID and link are saved back against the task so you can always find it.

**Technically:**
Calls **Jira Cloud REST API v3** — `POST /rest/api/3/issue` (create) or `PUT /rest/api/3/issue/{id}` (update).

Payload built by `_build_fields()`:
- `summary` ← `task_heading`
- `description` ← ADF (Atlassian Document Format) with description + acceptance criteria as a bulleted list
- `issuetype` ← mapped from `task_type`
- `priority` ← mapped directly
- `customfield_10016` ← story points
- `fixVersions` ← fix version name
- Sprint sent as label: `sprint:<name>` (Jira sprint field requires board ID, label is safer)

After push: `jira_id` and `jira_url` written back to the DB record. Action returned: `"created"` or `"updated"`.

---

### Key API Endpoints

**In simple terms:**
These are the communication channels between the frontend (website) and the backend (server). Every button click on the UI triggers one of these calls behind the scenes.

| Method | Endpoint | What It Does (Plain English) |
|--------|---------|------------------------------|
| POST | `/api/v1/files/upload` | Upload a document |
| POST | `/api/v1/files/{id}/parse` | Start the AI extraction on an uploaded file |
| GET | `/api/v1/files/{id}/progress` | Check how far along the extraction is (live progress bar) |
| GET | `/api/v1/files/{id}/status` | Check if extraction is done |
| POST | `/api/v1/files/register` | Register a GCS file in the database (idempotent) |
| GET | `/api/v1/tasks` | Get the list of extracted tasks |
| PATCH | `/api/v1/tasks/{id}` | Edit/update a task |
| DELETE | `/api/v1/tasks/{id}` | Soft-delete a task |
| POST | `/api/v1/tasks/export` | Download tasks as CSV, JSON, or Markdown |
| POST | `/api/v1/tasks/jira-push` | Push selected tasks to Jira |

---

### Frontend Architecture

`webapp/requirements_pod/src/`

**In simple terms:**
The website is built as a single page application — meaning the page never fully reloads. Switching between tabs, uploading files, editing tasks all happen instantly without page refreshes. Each screen is broken into small reusable components.

**Technically:**

| Component | What It Does |
|-----------|-------------|
| `App.jsx` | Root component — manages which tab is active, the LLM selector, and state shared across the app |
| `api/client.js` | Central place where all backend calls are made — consistent error handling across the whole app |
| `FileManager.jsx` | Combines the upload area and the stored files browser into one screen |
| `FileUpload.jsx` | Drag and drop upload — shows the live progress bar while extraction runs |
| `GCSBrowser.jsx` | Lists files stored in GCS for the current user; supports selecting up to 5 for batch parsing |
| `TaskTable.jsx` | The main task list — sort, filter, inline edit, bulk export, Jira push with confirm dialog |
| `TaskCard.jsx` | Full edit modal when you click into a single task |
| `GapReportModal.jsx` | Shows the gap analysis results — which tasks are missing fields |

Vite dev server proxies `/api` → `http://localhost:8000` so there are no browser security (CORS) issues in development.

---

### Testing Strategy

**In simple terms:**
Every important part of the system has automated tests. These tests run without needing a real database, a real AI key, or a real Jira account — everything is simulated. This means any developer can run the full test suite on their laptop in seconds and catch problems before they reach production.

**Technically:**
- **Unit tests** per pipeline module: `test_chunker.py`, `test_dedup.py`, `test_merge.py`, `test_temporal.py`, `test_scoring.py`, `test_pipeline.py`
- **Integration tests**: `tests/test_files.py`, `tests/test_tasks.py`, `tests/test_extraction.py`, `tests/test_jira.py`
- In-memory SQLite via `conftest.py` — no MySQL/Cloud SQL setup needed for tests
- `MockProvider` auto-activated in tests — no `ANTHROPIC_API_KEY` needed
- HTTP mocking via `respx` for Jira API calls

Run all tests:
```bash
pytest
```

Run extraction pipeline tests only:
```bash
pytest core/requirements_pod/services/extraction/tests/
```
