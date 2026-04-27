# DEAH Requirements Agent — API Reference

**Base URL:** `http://35.209.107.68:5173/api/v1`

All endpoints accept and return `application/json` unless noted. The export endpoint returns a file attachment.

---

## Quick Summary

| Endpoint | Method | Description |
|---|---|---|
| `/files/upload` | POST | Upload a new document file |
| `/files/list` | GET | List stored files by project |
| `/files/projects` | GET | List all project names |
| `/files/find` | GET | Find a file record by storage path |
| `/files/register` | POST | Register an existing GCS file in the DB |
| `/files/{file_id}/parse` | POST | Extract tasks from a single file |
| `/files/parse-merged` | POST | Merge multiple files and extract as one |
| `/files/{file_id}/progress` | GET | Get live extraction progress |
| `/files/{file_id}/gap-progress` | GET | Get gap analysis progress |
| `/files/{file_id}/gaps/reanalyze` | POST | Re-run gap analysis for a file |
| `/files/storage` | DELETE | Delete a file from storage |
| `/files/{file_id}/status` | GET | Get file record status |
| `/files/{file_id}/coverage-gaps` | GET | Get coverage gap report for a file |
| `/tasks` | GET | List all extracted tasks (with filters) |
| `/tasks/{task_id}` | GET | Get a single task |
| `/tasks/{task_id}` | PATCH | Update a task |
| `/tasks/{task_id}` | DELETE | Soft-delete a task |
| `/tasks/{task_id}/gaps` | GET | Get gap analysis report for a task |
| `/tasks/{task_id}/gaps/apply` | POST | Accept an AI gap suggestion |
| `/tasks/export` | POST | Export tasks as JSON / CSV / Markdown |
| `/tasks/jira-push` | POST | Push tasks to Jira |
| `/tasks/jira-push-linked` | POST | Push story + linked tasks to Jira |

---

## File Endpoints

### 1. Upload File

**`POST /files/upload`**

Uploads a document to Google Cloud Storage and registers it in the database.
Accepted formats: `pdf`, `docx`, `txt`, `md`, `vtt`, `srt`. Maximum size: 50 MB.

**Request** — `multipart/form-data`

| Field | Type | Required | Description |
|---|---|---|---|
| `file` | file | Yes | Document file to upload |
| `project_name` | string | No | Project/product folder name (e.g. `"5G_PROJECT"`) |
| `user_name` | string | No | Uploader name |

**Response**

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique file record UUID |
| `filename` | string | Original filename |
| `file_path` | string | Full GCS storage path |
| `storage_location` | string | `"gcs"` or `"local"` |
| `upload_time` | string | ISO 8601 timestamp |
| `status` | string | `"uploaded"` |
| `file_size` | integer | Size in bytes |
| `mime_type` | string | Detected MIME type |

```json
{
  "id": "f3a1c2d4-8e5f-4a3b-b2c1-9d0e7f6a5b4c",
  "filename": "Webex_Call_Transcript_Sprint1.docx",
  "file_path": "requirements-pod/5G_PROJECT/2026-04-20/Webex_Call_Transcript_Sprint1_260420.docx",
  "storage_location": "gcs",
  "upload_time": "2026-04-20T10:30:00",
  "status": "uploaded",
  "file_size": 13131,
  "mime_type": "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
}
```

---

### 2. List Files

**`GET /files/list`**

Returns all stored files under a project prefix, newest first.

**Query Parameters**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `project_name` | string | No | Filter by project folder. Returns all files if omitted. |

```
GET /files/list?project_name=5G_PROJECT
```

**Response** — array of storage objects

```json
[
  {
    "name": "requirements-pod/5G_PROJECT/2026-04-20/Webex_Sprint1_260420.docx",
    "size": 13131,
    "updated": 1745145600
  }
]
```

---

### 3. List Projects

**`GET /files/projects`**

Returns all unique project folder names that exist in storage.

**Response** — array of strings

```json
["5G_PROJECT", "CUSTOMER_360", "NETWORK_OPS"]
```

---

### 4. Find File by Path

**`GET /files/find`**

Looks up a file's database record by its GCS storage path.

**Query Parameters**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `path` | string | Yes | Full GCS storage path |

```
GET /files/find?path=requirements-pod/5G_PROJECT/2026-04-20/Webex_Sprint1_260420.docx
```

**Response** — FileOut object (same shape as Upload response), or `404` if not found.

---

### 5. Register File

**`POST /files/register`**

Creates a database record for a file that already exists in GCS (find-or-create). Used when staging files from the Stored Files browser without re-uploading.

**Request Body**

| Field | Type | Required | Description |
|---|---|---|---|
| `file_path` | string | Yes | Full GCS storage path |
| `storage_location` | string | No | `"gcs"` (default) or `"local"` |
| `user_name` | string | No | Associated user |
| `file_size` | integer | No | File size in bytes |

```json
{
  "file_path": "requirements-pod/5G_PROJECT/2026-04-20/Webex_Sprint1_260420.docx",
  "storage_location": "gcs",
  "file_size": 13131
}
```

**Response** — FileOut object

---

### 6. Extract Tasks — Single File

**`POST /files/{file_id}/parse`**

Downloads the file from storage, runs the 9-stage AI extraction pipeline, and saves extracted tasks to the database. Any previously extracted non-pushed tasks for this file are cleared before saving new ones. Gap analysis runs automatically in the background after extraction completes.

**Path Parameter**

| Parameter | Description |
|---|---|
| `file_id` | File record UUID from upload or register |

**Query Parameters**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `llm_provider` | string | No | Override LLM backend: `"claude"`, `"claude-sdk"`, `"mock"` |

```
POST /files/f3a1c2d4-.../parse?llm_provider=claude-sdk
```

**Response** — array of TaskOut objects

```json
[
  {
    "id": "t1b2c3d4-1a2b-3c4d-5e6f-7a8b9c0d1e2f",
    "task_id": "SRC-042",
    "task_heading": "Implement scheduled incremental load pipeline",
    "description": "Build a pipeline to load incremental data from MySQL to BigQuery on a daily schedule.",
    "task_type": "story",
    "status": "extracted",
    "priority": "medium",
    "story_points": 13,
    "assignee": "Shailendra",
    "source_file_id": "f3a1c2d4-8e5f-4a3b-b2c1-9d0e7f6a5b4c",
    "confidence_score": 1.0,
    "created_at": "2026-04-20T10:35:00",
    "updated_at": "2026-04-20T10:35:00",
    "gap_report": null
  }
]
```

**Extraction Pipeline Stages**

The 9-stage pipeline runs synchronously; progress can be tracked via the `/progress` endpoint.

| Stage | Description |
|---|---|
| 1 — normalising | Strip null bytes, normalise line endings |
| 2 — chunking | Split text into overlapping chunks |
| 3 — extracting | Parallel LLM extraction per chunk |
| 4 — deduplicating | Local dedup per document (cosine similarity) |
| 5 — pooling | Build global task pool across all chunks |
| 6 — merging | Graph similarity merge (cross-chunk dedup) |
| 7 — temporal reasoning | Infer schedule intervals and dates |
| 8 — scoring | Assign confidence scores, sort descending |
| 9 — saving | Clear old tasks, write new tasks to DB |

---

### 7. Extract Tasks — Merged Files

**`POST /files/parse-merged`**

Downloads multiple files, merges their text into one document, and runs a single AI extraction pass. Recommended when multiple files belong to the same session (e.g. transcript + technical spec from the same sprint). Clears old non-pushed tasks for all provided file IDs before saving.

**Request Body**

| Field | Type | Required | Description |
|---|---|---|---|
| `file_ids` | array of strings | Yes | List of file record UUIDs to merge and extract |

**Query Parameters**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `llm_provider` | string | No | Override LLM backend: `"claude"`, `"claude-sdk"`, `"mock"` |

```json
{
  "file_ids": [
    "f3a1c2d4-8e5f-4a3b-b2c1-9d0e7f6a5b4c",
    "a9b8c7d6-5e4f-3a2b-1c0d-9e8f7a6b5c4d"
  ]
}
```

**Response** — same as single file parse (array of TaskOut objects)

---

### 8. Extraction Progress

**`GET /files/{file_id}/progress`**

Returns live extraction progress while a parse is in-flight. Recommended polling interval: ~800 ms. Returns an empty object `{}` when no extraction is running for the file.

**Response**

| Field | Type | Description |
|---|---|---|
| `stage` | string | Current pipeline stage (see stages table above) |
| `chunks_done` | integer | Number of chunks processed so far |
| `chunks_total` | integer | Total number of chunks |
| `pct` | integer | Completion percentage (0–100) |

```json
{
  "stage": "extracting",
  "chunks_done": 3,
  "chunks_total": 7,
  "pct": 54
}
```

---

### 9. Gap Analysis Progress

**`GET /files/{file_id}/gap-progress`**

Returns the status of background gap analysis that runs after extraction completes. Poll until `status` is `"done"` or `"error"`.

**Response**

| Field | Type | Description |
|---|---|---|
| `file_id` | string | File record UUID |
| `status` | string | `"pending"`, `"running"`, `"done"`, `"error"`, `"idle"` |
| `task_count` | integer | Total tasks being analysed |
| `done_count` | integer | Tasks with completed gap analysis |

```json
{
  "file_id": "f3a1c2d4-8e5f-4a3b-b2c1-9d0e7f6a5b4c",
  "status": "running",
  "task_count": 4,
  "done_count": 2
}
```

> When `status` is `"done"`, the entry is cleared after the first poll — the frontend should reload tasks immediately on seeing this value.

---

### 10. Re-run Gap Analysis

**`POST /files/{file_id}/gaps/reanalyze`**

Clears existing gap reports for all story tasks in a file and re-runs gap analysis. Returns immediately — track progress via `GET /files/{file_id}/gap-progress`.

**Response**

```json
{
  "file_id": "f3a1c2d4-8e5f-4a3b-b2c1-9d0e7f6a5b4c",
  "status": "started",
  "task_count": 4,
  "message": "Gap analysis started for 4 task(s)."
}
```

If analysis is already running: `{ "status": "already_running", "message": "Gap analysis is already in progress." }`

If no tasks exist for the file: `{ "status": "skipped", "message": "No tasks found for this file." }`

---

### 11. Delete File from Storage

**`DELETE /files/storage`**

Permanently deletes a file from GCS and removes its database record. Associated task records have their `source_file_id` set to `null` (they are not deleted).

**Query Parameters**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `path` | string | Yes | Full GCS storage path of the file to delete |

```
DELETE /files/storage?path=requirements-pod/5G_PROJECT/2026-04-20/Webex_Sprint1.docx
```

**Response** — `204 No Content`

---

### 12. File Status

**`GET /files/{file_id}/status`**

Returns the current status record for a file.

**Response** — FileOut object

| `status` value | Meaning |
|---|---|
| `uploaded` | File stored, not yet extracted |
| `parsed` | Extraction completed successfully |
| `error` | Extraction failed |

---

### 13. Coverage Gaps

**`GET /files/{file_id}/coverage-gaps`**

Returns the coverage gap report generated after extraction — topics or areas mentioned in the source document that were not captured as any task.

**Response**

```json
{
  "file_id": "f3a1c2d4-8e5f-4a3b-b2c1-9d0e7f6a5b4c",
  "analyzed_at": "2026-04-20T10:40:00",
  "gaps": [
    {
      "topic": "Error handling for API timeouts",
      "severity": "medium",
      "description": "Transcript mentions retry logic but no corresponding task was extracted."
    }
  ]
}
```

---

## Task Endpoints

### 14. List Tasks

**`GET /tasks`**

Returns all extracted tasks. Soft-deleted tasks are excluded by default unless `status=deleted` is passed.

**Query Parameters**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `source_file` | string | No | Filter by file record UUID |
| `user_name` | string | No | Filter by uploader / user name |
| `status` | string | No | Filter by task status: `extracted`, `modified`, `pushed`, `deleted` |

```
GET /tasks?source_file=f3a1c2d4-...&status=extracted
```

**Response** — array of TaskOut objects

---

### 15. Get Task

**`GET /tasks/{task_id}`**

Returns a single task by its sequential ID.

**Path Parameter**

| Parameter | Description |
|---|---|
| `task_id` | Sequential task ID in `SRC-NNN` format (e.g. `SRC-042`) |

**Response** — TaskOut object

```json
{
  "id": "t1b2c3d4-1a2b-3c4d-5e6f-7a8b9c0d1e2f",
  "task_id": "SRC-042",
  "task_heading": "Implement scheduled incremental load pipeline",
  "description": "Build a pipeline to load incremental MySQL data into BigQuery daily.",
  "task_type": "story",
  "status": "extracted",
  "priority": "medium",
  "story_points": 13,
  "assignee": "Shailendra",
  "reporter": null,
  "sprint": null,
  "fix_version": null,
  "start_date": null,
  "due_date": null,
  "acceptance_criteria": "[\"Pipeline runs daily at 2AM\", \"Incremental load handles deletes\"]",
  "schedule_interval": "daily",
  "confidence_score": 1.0,
  "jira_id": null,
  "jira_url": null,
  "gap_report": null,
  "source_file_id": "f3a1c2d4-8e5f-4a3b-b2c1-9d0e7f6a5b4c",
  "task_source": "Webex_Call_Transcript_Sprint1.docx",
  "created_at": "2026-04-20T10:35:00",
  "updated_at": "2026-04-20T10:35:00"
}
```

---

### 16. Update Task

**`PATCH /tasks/{task_id}`**

Updates editable fields on a task. Only fields included in the request body are modified. After saving, the gap report is automatically refreshed so resolved fields are cleared from the gap badge.

**Request Body** — all fields optional

| Field | Type | Description |
|---|---|---|
| `task_heading` | string | Task summary / title |
| `description` | string | Full task description |
| `task_type` | string | `"bug"`, `"story"`, `"task"`, `"subtask"` |
| `status` | string | `"extracted"`, `"modified"`, `"pushed"`, `"deleted"` |
| `priority` | string | `"critical"`, `"high"`, `"medium"`, `"low"` |
| `story_points` | integer | Effort estimate in story points |
| `assignee` | string | Developer implementing the task |
| `reporter` | string | Person who raised the requirement |
| `sprint` | string | Sprint name (e.g. `"Sprint 14"`) |
| `fix_version` | string | Target release version (e.g. `"v2.1.0"`) |
| `start_date` | string | ISO date string `"2026-04-20"` |
| `due_date` | string | ISO date string `"2026-05-01"` |
| `acceptance_criteria` | string | JSON-encoded array of criteria strings |
| `schedule_interval` | string | `"daily"`, `"hourly"`, `"weekly"`, `"on-demand"` |

```json
{
  "priority": "high",
  "story_points": 21,
  "sprint": "Sprint 14",
  "assignee": "Shailendra"
}
```

**Response** — updated TaskOut object

> Tasks with `status: "pushed"` or `status: "deleted"` return `410 Gone`.

---

### 17. Delete Task

**`DELETE /tasks/{task_id}`**

Soft-deletes a task by setting its status to `"deleted"`. The record is retained in the database but excluded from all list results unless explicitly filtered.

**Response** — TaskOut object with `"status": "deleted"`

---

### 18. Get Gap Report

**`GET /tasks/{task_id}/gaps`**

Returns the AI-generated gap analysis report for a task, identifying missing, incomplete, or assumed fields.

**Response**

```json
{
  "task_id": "SRC-042",
  "gap_report": {
    "field_gaps": [
      {
        "field": "schedule_interval",
        "severity": "high",
        "message": "Schedule interval is missing — required for pipeline tasks.",
        "suggestion": "daily",
        "can_apply": true,
        "resolved": false
      },
      {
        "field": "acceptance_criteria",
        "severity": "medium",
        "message": "No acceptance criteria defined.",
        "suggestion": "[\"Pipeline completes within 30 minutes\", \"Failed records are logged to Cloud Logging\"]",
        "can_apply": true,
        "resolved": false
      }
    ],
    "assumptions": [
      {
        "field": "assignee",
        "message": "Assignee inferred from conversation context.",
        "current_value": "Shailendra"
      }
    ]
  }
}
```

---

### 19. Apply Gap Suggestion

**`POST /tasks/{task_id}/gaps/apply`**

Applies an AI-suggested value for a specific missing field, updates the task, and refreshes the gap report.

Allowed fields: `acceptance_criteria`, `schedule_interval`, `story_points`, `priority`, `assignee`, `description`, `due_date`, `start_date`

**Request Body**

| Field | Type | Required | Description |
|---|---|---|---|
| `field` | string | Yes | Field name to update |
| `value` | any | Yes | Value to apply (string, integer, or JSON-encoded string for arrays) |

```json
{
  "field": "schedule_interval",
  "value": "daily"
}
```

**Response** — updated TaskOut object with refreshed `gap_report`

---

### 20. Export Tasks

**`POST /tasks/export`**

Downloads selected tasks as a file. Returns a file attachment, not JSON.

**Request Body**

| Field | Type | Required | Description |
|---|---|---|---|
| `task_ids` | array of strings | Yes | List of `SRC-NNN` task IDs to export |
| `format` | string | No | `"json"` (default), `"csv"`, or `"md"` |

```json
{
  "task_ids": ["SRC-042", "SRC-043", "SRC-044"],
  "format": "csv"
}
```

**Response** — file download

| Format | Content-Type | Filename |
|---|---|---|
| `json` | `application/json` | `tasks.json` |
| `csv` | `text/csv` | `tasks.csv` |
| `md` | `text/markdown` | `tasks.md` |

---

### 21. Push Tasks to Jira

**`POST /tasks/jira-push`**

Pushes selected tasks to Jira. Creates new issues for tasks without a `jira_id`; updates existing issues for tasks that already have one. Task status is set to `"pushed"` on success.

**Request Body**

| Field | Type | Required | Description |
|---|---|---|---|
| `task_ids` | array of strings | Yes | List of `SRC-NNN` task IDs to push |

```json
{
  "task_ids": ["SRC-042", "SRC-043"]
}
```

**Response**

```json
{
  "results": [
    {
      "task_id": "SRC-042",
      "success": true,
      "jira_id": "SCRUM-150",
      "jira_url": "https://your-org.atlassian.net/browse/SCRUM-150",
      "action": "created"
    },
    {
      "task_id": "SRC-043",
      "success": true,
      "jira_id": "SCRUM-148",
      "jira_url": "https://your-org.atlassian.net/browse/SCRUM-148",
      "action": "updated"
    }
  ]
}
```

| `action` value | Meaning |
|---|---|
| `created` | New Jira issue was created |
| `updated` | Existing Jira issue was updated |

---

### 22. Push Story + Linked Tasks to Jira

**`POST /tasks/jira-push-linked`**

Pushes a parent story to Jira first, then pushes all specified child tasks and creates issue links from each task back to the story. Use this when tasks should appear as linked work items under a parent story in Jira.

**Request Body**

| Field | Type | Required | Description |
|---|---|---|---|
| `story_task_id` | string | Yes | `SRC-NNN` ID of the parent story task |
| `task_ids` | array of strings | Yes | `SRC-NNN` IDs to push and link under the story |

```json
{
  "story_task_id": "SRC-042",
  "task_ids": ["SRC-043", "SRC-044", "SRC-045"]
}
```

**Response**

```json
{
  "story_jira_key": "SCRUM-150",
  "results": [
    {
      "task_id": "SRC-042",
      "role": "story",
      "success": true,
      "jira_id": "SCRUM-150",
      "jira_url": "https://your-org.atlassian.net/browse/SCRUM-150",
      "action": "created",
      "linked_to": null
    },
    {
      "task_id": "SRC-043",
      "role": "task",
      "success": true,
      "jira_id": "SCRUM-151",
      "jira_url": "https://your-org.atlassian.net/browse/SCRUM-151",
      "action": "created",
      "linked_to": "SCRUM-150",
      "link_warning": null
    },
    {
      "task_id": "SRC-044",
      "role": "task",
      "success": true,
      "jira_id": "SCRUM-152",
      "jira_url": "https://your-org.atlassian.net/browse/SCRUM-152",
      "action": "created",
      "linked_to": "SCRUM-150",
      "link_warning": null
    }
  ]
}
```

---

## Data Models

### FileOut

| Field | Type | Description |
|---|---|---|
| `id` | string | UUID primary key |
| `filename` | string | Original filename |
| `file_path` | string | Full GCS storage path |
| `storage_location` | string | `"gcs"` or `"local"` |
| `uploaded_by` | string\|null | Uploader name |
| `upload_time` | string | ISO 8601 timestamp |
| `status` | string | `"uploaded"`, `"parsed"`, `"error"` |
| `file_size` | integer\|null | Size in bytes |
| `mime_type` | string\|null | Detected MIME type |
| `coverage_gaps` | string\|null | JSON-encoded coverage gap report |

### TaskOut

| Field | Type | Description |
|---|---|---|
| `id` | string | UUID primary key |
| `task_id` | string | Sequential ID, `SRC-NNN` format |
| `task_heading` | string | Task title / summary |
| `description` | string\|null | Full task description |
| `task_type` | string | `"bug"`, `"story"`, `"task"`, `"subtask"` |
| `status` | string | `"extracted"`, `"modified"`, `"pushed"`, `"deleted"` |
| `priority` | string\|null | `"critical"`, `"high"`, `"medium"`, `"low"` |
| `story_points` | integer\|null | Effort estimate |
| `assignee` | string\|null | Assigned developer |
| `reporter` | string\|null | Requirement owner |
| `sprint` | string\|null | Sprint name |
| `fix_version` | string\|null | Target release version |
| `start_date` | string\|null | ISO date |
| `due_date` | string\|null | ISO date |
| `acceptance_criteria` | string\|null | JSON-encoded array of criteria |
| `schedule_interval` | string\|null | `"daily"`, `"hourly"`, `"weekly"`, `"on-demand"` |
| `confidence_score` | float\|null | AI extraction confidence (0.0–1.0) |
| `jira_id` | string\|null | Jira issue key (e.g. `"SCRUM-150"`) |
| `jira_url` | string\|null | Full Jira issue URL |
| `location` | string\|null | Source location / section in the document |
| `gap_report` | string\|null | JSON-encoded gap analysis result |
| `source_file_id` | string\|null | Parent file record UUID |
| `task_source` | string\|null | Source filename |
| `user_name` | string\|null | Uploader name |
| `created_at` | string | ISO 8601 creation timestamp |
| `updated_at` | string | ISO 8601 last-modified timestamp |

---

## Error Responses

All errors return a JSON body containing a `detail` object with a human-readable message and a machine-readable `code`.

| HTTP Status | Code | Trigger |
|---|---|---|
| `400` | — | Bad request — invalid input or missing required fields |
| `401` | `LLM_AUTH_ERROR` | Invalid or missing LLM API key |
| `402` | `LLM_QUOTA_ERROR` | LLM usage quota exceeded |
| `404` | `FILE_NOT_FOUND` | File record UUID does not exist |
| `404` | `TASK_NOT_FOUND` | Task ID does not exist |
| `410` | `TASK_DELETED` | Task is deleted and cannot be updated |
| `413` | `FILE_TOO_LARGE` | File exceeds 50 MB limit |
| `422` | `INVALID_EXTENSION` | Unsupported file format |
| `422` | `INVALID_FIELD` | Field not allowed via gap apply |
| `422` | `INVALID_FORMAT` | Unknown export format |
| `500` | `PARSE_ERROR` | Extraction pipeline failed |
| `500` | `STORAGE_ERROR` | GCS read/write/delete failed |

```json
{
  "detail": {
    "detail": "Source file 'f3a1c2d4-...' not found.",
    "code": "FILE_NOT_FOUND"
  }
}
```
