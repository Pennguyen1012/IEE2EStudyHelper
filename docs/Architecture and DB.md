# Technical Architecture & Database Schema

**Project:** SAP IEE2E Learning & RAG Evaluation Platform  
**Environment:** Cloudflare Ecosystem (Free Tier) + Gemini API

---

## 1. System Flow

### 1.1. Authentication Flow

1. User selects Role -> Enters ID/Password on the React Frontend.
2. Cloudflare Workers (CF Workers) receive the request and query the `Users` table in Cloudflare D1.
3. Returns a JWT Token to be stored in an HttpOnly Cookie or LocalStorage.
4. If `is_first_login = true`, the Frontend forces a redirect to the password reset page.

### 1.2. Document & RAG Pipeline (Background Processing)

1. User (Admin/Learner) uploads/edits `.md`, `.docx`, or `.txt` files.
2. Frontend sends the file payload to CF Workers.
3. CF Workers save the raw file directly to **Cloudflare R2** (with a new version tag) and insert metadata into the `Note_Versions` table (D1).
4. CF Workers immediately return a `200 OK` status to the Frontend so the user doesn't have to wait.
5. **Background Task:** CF Workers use `ctx.waitUntil()` to execute a background process:
   - Extract text from the file.
   - Perform text chunking.
   - Call Gemini API to generate Vector Embeddings.
   - Push Vectors to **Cloudflare Vectorize** alongside `unit_id` and `topic_id` metadata.

### 1.3. Quiz Execution & Autosave Flow

1. User starts a Quiz -> CF Workers create a record in `Quiz_Sessions` (D1) with status `in_progress`.
2. While the user is taking the quiz, the Frontend triggers an Autosave request every 3 seconds.
3. CF Workers receive the data and write it to **Cloudflare KV** (High-speed cache, avoids consuming D1 write quotas). Key structure: `autosave:{session_id}`.
4. If the user clicks "Move on" or navigates between MCQ and SAQ sections -> Frontend fetches data from CF KV to re-render the quiz state seamlessly.

### 1.4. Submission & AI Grading Flow

1. User clicks "Submit". Frontend locks the button and displays a loading spinner.
2. CF Workers retrieve the final, complete dataset from CF KV, write it permanently to the `Quiz_Answers` table (D1), and delete the KV record.
3. The MCQ grading function executes immediately and calculates the score.
4. The SAQ grading function executes next. For each SAQ:
   - Fetch context from Cloudflare Vectorize based on `unit_id` or `topic_id`.
   - Package Context + Sample Answer + User Answer and call the Gemini API.
   - **Queue/Backoff Logic:** If Gemini returns a `429 Too Many Requests` error, the Worker automatically triggers a `setTimeout` (e.g., 4 seconds) and retries (maximum of 3 attempts).
5. The Worker saves the JSON result returned by the AI (including the Conflict flag, if any) into D1 and sends the final aggregated results back to the Frontend.

---

## 2. Tech Stack Details

- **Frontend:** React (SPA) hosted on **Cloudflare Pages**.
- **Backend / API:** **Cloudflare Workers** (Node.js/TypeScript).
- **Relational Database:** **Cloudflare D1** (SQLite edge DB).
- **High-Speed Cache:** **Cloudflare KV** (Used for Autosave states).
- **Object Storage:** **Cloudflare R2** (Stores raw source files, supports versioning).
- **Vector Database:** **Cloudflare Vectorize** (Stores embedded text chunks for RAG).
- **AI Model:** **Gemini 2.5 Flash API** (Enforcing Structured Output JSON mode).
- **Email Service:** Free SMTP relay (SendGrid / Resend) invoked via REST API from the Worker.

---

## 3. Database Schema (Cloudflare D1 - SQLite)

### 3.1. User & Access

```sql
CREATE TABLE Users (
    id TEXT PRIMARY KEY,
    username TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    role TEXT NOT NULL, -- 'operator', 'admin', 'learner'
    email TEXT, -- Used for email notifications
    is_first_login BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3.2. Knowledge Categorization

```sql
CREATE TABLE Units (
    id TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    description TEXT
);

CREATE TABLE Topics (
    id TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    description TEXT
);
```

### 3.3. Document Management & Versioning

```sql
CREATE TABLE Notes (
    id TEXT PRIMARY KEY,
    author_id TEXT NOT NULL, -- References Users(id)
    unit_id TEXT, -- References Units(id)
    topic_id TEXT, -- References Topics(id)
    title TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE Note_Versions (
    id TEXT PRIMARY KEY,
    note_id TEXT NOT NULL, -- References Notes(id)
    version_number INTEGER NOT NULL,
    storage_key TEXT NOT NULL, -- R2 file path (e.g., notes/123/v2.md)
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE Note_Comments (
    id TEXT PRIMARY KEY,
    note_version_id TEXT NOT NULL, -- Pinpoints the exact version being commented on
    admin_id TEXT NOT NULL,
    highlight_text TEXT,
    comment_body TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3.4. Document Management & Versioning

```sql
CREATE TABLE Questions (
    id TEXT PRIMARY KEY,
    type TEXT NOT NULL, -- 'mcq' or 'saq'
    unit_id TEXT, -- Nullable, but at least one (unit_id or topic_id) must be populated
    topic_id TEXT, -- Nullable
    content TEXT NOT NULL,
    options_json TEXT, -- JSON Array storing answer choices (e.g., ["A", "B", "C"])
    is_multiple_choice BOOLEAN DEFAULT FALSE, -- For MCQs allowing multiple correct options
    correct_answer_json TEXT, -- JSON Array storing correct answer(s) (e.g., [0, 2])
    sample_answer TEXT -- Target answer for SAQ RAG evaluation
);

CREATE TABLE Quiz_Sessions (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    status TEXT DEFAULT 'in_progress', -- 'in_progress', 'submitted', 'graded'
    started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    submitted_at TIMESTAMP
);

CREATE TABLE Quiz_Answers (
    id TEXT PRIMARY KEY,
    session_id TEXT NOT NULL,
    question_id TEXT NOT NULL,
    user_answer TEXT, -- The option selected or text typed by the user
    is_correct BOOLEAN,
    feedback TEXT,
    is_conflict BOOLEAN DEFAULT FALSE,
    conflict_reason TEXT
);
```

---

## 4. Storage Structure

### 4.1. Cloudflare R2 (Object Storage)

Naming Convention to strictly enforce Versioning:

- Pattern: `notes/{note_id}/v{version_number}.{extension}`
- Example: `notes/note_abc123/v1.md, notes/note_abc123/v2.md`
  (_When fetching a note to read, the Worker queries `Note_Versions` for the latest `storage_key`, then retrieves the corresponding object from R2_)

### 4.2. Cloudflare KV (Temporary Storage)

Utilized for continuous Autosaving without draining the primary database write quota.

- Key Pattern: `autosave:session:{session_id}`
- Value Format:

```json
{
  "mcq_answers": { "q1_id": [0], "q2_id": [1, 2] },
  "saq_answers": { "q3_id": "User's draft response..." },
  "last_saved": "2026-06-10T09:00:00Z"
}
```

### 4.3. Cloudflare Vectorize (Vector Database)

- **Dimension:** Dependent on the `text-embedding-004` model output (typically 768 dimensions).
- **Metadata attached to each Vector:** Crucial for limiting the RAG search scope and improving accuracy.

```json
{
  "unit_id": "unit_456",
  "topic_id": "topic_789",
  "note_id": "note_abc123",
  "chunk_index": 4
}
```
