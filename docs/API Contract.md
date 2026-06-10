# API Contract & Communication Protocol

**Base URL:** `/api/v1`
**Authorization:** All endpoints (except `/auth/login`) require the `Authorization: Bearer <JWT_TOKEN>` header.

---

## 1. Authentication (`/auth`)

### 1.1. Login

- **Endpoint:** `POST /auth/login`
- **Description:** Authenticates user and returns a JWT.
- **Request Payload:**
  ```json
  {
    "username": "learner_01",
    "password": "hashed_password_or_plain"
  }
  ```
- **Response (200 OK):**
  ```json
  {
    "token": "eyJhbGciOiJIUzI1...",
    "user": {
      "id": "usr_123",
      "role": "learner",
      "isFirstLogin": true
    }
  }
  ```

### 1.2. Change Password

- **Endpoint:** `POST /auth/change-password`
- **Description:** Mandatory step if `isFirstLogin` is true.
- **Request Payload:**
  ```json
  {
    "oldPassword": "...",
    "newPassword": "..."
  }
  ```
- **Response (200 OK):** `{ "message": "Password updated successfully." }`

---

## 2. Notes & Document Management (`/notes`)

### 2.1. Get Notes List (Paginated)

- **Endpoint:** `GET /notes`
- **Query Parameters:** `?unit_id=unit_456&page=1&limit=20`
- **Description:** Retrieves a paginated list of notes for a specific unit.
- **Response (200 OK):**
  ```json
  {
    "data": [
      {
        "id": "note_abc",
        "title": "SAP Architecture Overview",
        "author_id": "usr_789",
        "latest_version": 2,
        "updated_at": "2026-06-10T10:00:00Z"
      }
    ],
    "pagination": {
      "currentPage": 1,
      "totalPages": 3,
      "totalItems": 45
    }
  }
  ```

### 2.2. Request Upload URL (Presigned URL Flow - Step 1)

- **Endpoint:** `POST /notes/upload-url`
- **Description:** Generates a direct R2 upload link. Frontend will PUT the raw file to this URL.
- **Request Payload:**
  ```json
  {
    "fileName": "sap_notes.docx",
    "contentType": "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
  }
  ```
- **Response (200 OK):**
  ```json
  {
    "uploadUrl": "https://pub-r2-link.../notes/temp_123?X-Amz-Signature=...",
    "storageKey": "notes/temp_123.docx"
  }
  ```

### 2.3. Confirm Upload & Trigger RAG (Presigned URL Flow - Step 2)

- **Endpoint:** `POST /notes`
- **Description:** Called after the file is successfully uploaded to R2. Creates DB records and triggers the background background `waitUntil` Vectorization task.
- **Request Payload:**
  ```json
  {
    "title": "My SAP Notes",
    "unit_id": "unit_456",
    "topic_id": "topic_789",
    "storageKey": "notes/temp_123.docx"
  }
  ```
- **Response (201 Created):** `{ "message": "Note created. AI vectorization running in background." }`

---

## 3. Quiz & Execution (`/quiz`)

### 3.1. Start Quiz Session

- **Endpoint:** `POST /quiz/start`
- **Description:** Initializes a session and fetches questions. _Crucial: Correct answers and sample answers are stripped from the response._
- **Request Payload:** `{ "unit_id": "unit_456" }`
- **Response (200 OK):**
  ```json
  {
    "sessionId": "sess_abc123",
    "mcq": [
      {
        "id": "q_001",
        "content": "What does SAP stand for?",
        "options": ["A", "B", "C", "D"],
        "isMultipleChoice": false
      }
    ],
    "saq": [
      {
        "id": "q_002",
        "content": "Explain the role of IEE2E briefly."
      }
    ]
  }
  ```

### 3.2. Autosave Session State

- **Endpoint:** `POST /quiz/autosave`
- **Description:** Writes current draft answers to Cloudflare KV. Called every 3 seconds.
- **Request Payload:**
  ```json
  {
    "sessionId": "sess_abc123",
    "mcqAnswers": { "q_001": [2] },
    "saqAnswers": { "q_002": "IEE2E acts as a bridge..." }
  }
  ```
- **Response (200 OK):** `{ "message": "Autosaved" }`

### 3.3. Final Submit & AI Grading

- **Endpoint:** `POST /quiz/submit`
- **Description:** Locks the session, fetches KV data, runs deterministic logic for MCQs, and executes Gemini RAG Backoff Queue for SAQs.
- **Request Payload:** `{ "sessionId": "sess_abc123" }`
- **Response (200 OK):**
  ```json
  {
    "status": "graded",
    "totalScore": "85/100",
    "details": {
      "mcq": [{ "id": "q_001", "isCorrect": true }],
      "saq": [
        {
          "id": "q_002",
          "isCorrect": false,
          "feedback": "Your answer missed the core component of data integration.",
          "isConflict": false
        }
      ]
    }
  }
  ```

---

## 4. Standard HTTP Error Codes

- **`400 Bad Request`**: Missing parameters or malformed JSON payload.
- **`401 Unauthorized`**: JWT token is missing, expired, or invalid.
- **`403 Forbidden`**: User lacks permissions (e.g., a Learner trying to access Admin endpoints).
- **`404 Not Found`**: Resource (Note, Quiz, User) does not exist.
- **`429 Too Many Requests`**: Client-side rate limiting (Frontend is spamming requests). _Note: Gemini's 429 error is handled internally via Worker backoff and will NOT be exposed to the frontend unless all 3 retries fail, in which case a `503 Service Unavailable` is returned._
- **`500 Internal Server Error`**: Uncaught backend crashes.
