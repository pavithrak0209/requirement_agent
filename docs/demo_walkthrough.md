# TaskFlow AI — Requirements Agent
## Demo Walkthrough

---

### What Is This?

TaskFlow AI is a Requirements Agent that reads any document — a meeting transcript, a technical spec, a Webex call summary — and automatically extracts all the requirements, tasks, bugs, and action items. It structures them, lets you review and edit, and pushes them directly to Jira.

**The problem it solves:** After every sprint call or design meeting, someone has to manually go through hours of discussion, pick out every requirement, and create Jira tickets one by one. This agent does that in seconds.

---

### The Demo Flow

---

#### 1. Open the App

When you open the app you see two tabs — **Files** and **Tasks**.

In the top right corner there is an LLM selector. Right now it is set to **Claude SDK**, which means it is using Anthropic's Claude model for real AI extraction. There is also a **Mock** mode for testing without making any API calls.

---

#### 2. Upload a Document

Click on the Files tab. You will see:
- A **Project / Product** dropdown — select your project (e.g. `5G_PROJECT`)
- A **drag and drop area** to upload your document
- A **Stored Files** panel on the right showing all previously uploaded files

The agent accepts:
- **PDF** — design docs, spec sheets
- **DOCX** — Word documents
- **TXT / MD** — plain text, markdown notes
- **VTT / SRT** — Webex or Teams transcript caption files

Drop your file in. The moment you upload, the file is saved to **Google Cloud Storage**, organized by project and date, and extraction starts automatically.

---

#### 3. Watch the Extraction Progress

A live progress bar appears showing exactly what stage the agent is on:

| Stage | What You See |
|-------|-------------|
| Normalising | Reading and decoding the document |
| Chunking | Splitting into sections |
| Extracting | Claude AI reading each section |
| Deduplicating | Removing repeated items |
| Merging | Combining similar tasks |
| Scoring | Rating each task for completeness |
| Saving | Storing results to database |

The whole process typically takes **20–40 seconds** for a standard transcript.

---

#### 4. Review the Tasks

Once done, you are automatically taken to the **Tasks** tab. Every item the AI found is listed here as a structured task card with:

- **Task ID** — e.g. `SRC-001`, auto-generated
- **Heading** — concise title extracted from the document
- **Type** — Bug, Story, Task, or Sub-task
- **Priority** — Critical, High, Medium, or Low
- **Story Points** — estimated in Fibonacci scale (1, 2, 3, 5, 8, 13, 21)
- **Acceptance Criteria** — measurable done conditions extracted from the discussion
- **Sprint / Fix Version** — if mentioned in the document
- **Confidence Score** — color coded:
  - Green — AI is very confident
  - Blue — good confidence
  - Amber — review recommended
  - Red — needs human attention

You can **filter** tasks by source file, assignee, or status. You can **sort** by any column.

---

#### 5. Edit a Task

Click on any task to open the edit panel. You can change:
- Task type and priority
- Description and acceptance criteria
- Story points
- Assignee (user name)
- Sprint name and fix version

Changes save instantly. The task status updates from **Extracted** to **Modified** so you can track what has been reviewed.

---

#### 6. Run Gap Analysis

Before pushing to Jira, click **Gap Report**. The agent checks every task for missing or incomplete fields — no description, empty acceptance criteria, missing story points. It gives you a coverage report so you know exactly what to fill in before the tickets land in Jira.

---

#### 7. Push to Jira

Select the tasks you want to push and click **Push to Jira**.

The agent:
- Creates **new** Jira tickets for tasks that have never been pushed
- **Updates** existing tickets if they were pushed before (no duplicates)
- Populates: summary, description, acceptance criteria, type, priority, story points, fix version, sprint label

A confirmation dialog shows you exactly how many were created vs updated.

After pushing, each task displays its **Jira ticket ID** and a **direct link** to Jira.

---

#### 8. Export

You can also export tasks as:
- **CSV** — for spreadsheets
- **JSON** — for integrations
- **Markdown** — for sharing with stakeholders

Files are auto-named with a timestamp suffix.

---

### Summary

| Step | Action | Time |
|------|--------|------|
| Upload | Drop a Webex transcript or spec doc | ~5 seconds |
| Extract | Claude AI reads and structures everything | ~30 seconds |
| Review | Check tasks, edit if needed | ~5 minutes |
| Push | Send to Jira with one click | ~5 seconds |

**What used to take an hour now takes under 10 minutes.**
