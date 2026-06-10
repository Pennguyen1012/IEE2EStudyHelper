# Product Requirements Document (PRD)

**Project Name:** SAP IEE2E Learning & RAG Evaluation Platform  
**Target Audience:** Internal Team (12 Members)  
**Current Status:** Draft / Phase 1

---

## 1. Executive Summary

### 1.1 Project Overview

The SAP IEE2E Learning Platform is an internal web application designed to facilitate study, consolidate knowledge, and provide automated evaluation for an enclosed team of 12 individuals (instructors and learners) preparing for the SAP IEE2E certification.

### 1.2 Core Objectives

- Centralize documentation and study notes categorized by Units.
- Provide a robust, Duolingo/Quizlet-style assessment system featuring both Multiple Choice Questions (MCQ) and Short Answer Questions (SAQ).
- Automate the grading of SAQs using a Retrieval-Augmented Generation (RAG) pipeline powered by the Gemini API.

### 1.3 Technical & Financial Constraints

- **Budget:** Strictly 0 VND/month.
- **Infrastructure:** 100% reliance on the Cloudflare Free Tier ecosystem (Pages, Workers, D1, R2, Vectorize).
- **AI Integration:** Gemini API (Free Tier), heavily optimized via backend queuing to never exceed the 15 Requests Per Minute (RPM) hard limit.

---

## 2. User Roles & Authentication

The system supports three distinct access levels. Users must explicitly select their role via a toggle switch on the login interface before authenticating.

### 2.1 Role Definitions

- **Operator (Project Owner):** Supreme privileges. Manages system configurations, database migrations, and top-level user account provisioning.
- **Admin (Instructor/Manager):** Responsible for content creation. Can upload source materials, construct quizzes, define sample answers, and review learner progress.
- **Learner (Student):** The end-user who consumes study materials, creates personal notes, and participates in assessments.

### 2.2 Authentication Flow

- **Mechanism:** Standard ID and Password login. Accounts are pre-provisioned by the Operator/Admin.
- **Security Protocol:** Upon the very first successful login, the system will enforce a mandatory password reset. Users cannot proceed to the dashboard without establishing a new, secure password.

---

## 3. Functional Requirements: Document & Notes Management

### 3.1 File Support & Categorization

- **Supported Formats:** `.md`, `.docx`, `.txt`.
- **Structure:** All uploaded documents must be assigned to a specific `Unit`. A single `Unit` serves as a folder/category that can house multiple document files.

### 3.2 CRUD Permissions & Versioning

- **Learners:** Have the authorization to Create, Read, Update, and Delete _only_ their own personal notes.
- **Version Control:** The system must implement robust versioning for all notes. Every update creates a new version footprint in the database (or R2), allowing users to restore previous iterations in case of accidental deletion or unwanted modifications.

### 3.3 Collaboration & Feedback (Admin Review Mode)

- **Visibility:** Admins hold read access to all Learner-generated notes.
- **Annotation UI:** Admins can interact with Learner notes using a "Suggest/Review" mechanism similar to Google Docs. Admins can highlight specific text blocks and attach textual comments via a dedicated Comment Box.
- **Notification System:** Whenever an Admin submits a comment on a note, the backend will trigger an automated email notification to the Learner's configured Gmail address (utilizing a free SMTP relay like Resend or SendGrid).

### 3.4 Vectorization Pipeline (Background Process)

- Whenever a file is uploaded or updated, a background Cloudflare Worker will parse the text, perform chunking, and push the vectorized embeddings (using Gemini's embedding model) into Cloudflare Vectorize. These vectors are tagged with the specific Unit/Topic ID for targeted RAG retrieval later.

---

## 4. Functional Requirements: Quiz System

### 4.1 Quiz Creation (Admin UI)

Admins require a streamlined, Quizlet-style interface to assemble assessments. A single quiz session is divided into two distinct sections:

- **Multiple Choice Questions (MCQ) - Max 40 questions:**
  - _Inputs:_ Question Title, Options, Checkbox for "Multiple correct answers allowed", and the exact Correct Answer(s).
  - _Metadata:_ Unit/Topic Label (This tag is strictly hidden from the Learner UI and is utilized solely by the backend RAG mechanism).
- **Short Answer Questions (SAQ) - Max 20 questions:**
  - _Inputs:_ Question Title, Unit/Topic Label (hidden from Learner), and the precise Sample Answer (the gold standard for AI grading).

### 4.2 Quiz Execution (Learner UI)

- **Sectional Flow:** The quiz is presented in two distinct phases. The Learner must complete the MCQ section first. Upon clicking "Move on to next section", the system locks/saves the MCQs and presents the SAQs. Learners can navigate back and forth between sections before final submission.
- **Autosave Mechanism:** To prevent data loss during long sessions, the Frontend will implement a silent autosave feature. Every 2 to 3 seconds, the current state of both MCQ and SAQ answers will be pushed to a temporary state in the backend (or LocalStorage synchronized with the DB).
- **Final Submission:** A global "Submit" button finalizes the session, pushing both MCQ and SAQ data for definitive grading.

---

## 5. Functional Requirements: AI Grading System (RAG)

The core technical complexity lies in the automated grading of the SAQs using the Gemini API.

### 5.1 Grading Logic & Prompt Engineering

- **Binary Assessment:** The system grades strictly on a Yes/No (Correct/Incorrect) basis.
- **Strict RAG Constraint:** The AI prompt must heavily penalize hallucinations. The model is explicitly instructed to evaluate the Learner's answer strictly against the Admin's _Sample Answer_ and the retrieved _Knowledge Base Context_ (from Cloudflare Vectorize). It must **not** rely on its internal pre-trained knowledge or perform external web searches to validate answers.

### 5.2 JSON Output Schema

To ensure predictable backend parsing, the Gemini API will be forced (via Structured Outputs) to return an exact JSON array for each SAQ:

```json
{
  "id": "string (Question ID)",
  "isCorrect": "boolean",
  "feedback": "string (Short, 1-2 sentence explanation)"
}
```

### 5.3 Conflict Handling Mechanism

Given the strict instructions, there may be edge cases where a Learner provides an objectively true answer (in the real world), but it contradicts the Admin's provided context. The AI must flag this instead of blindly marking it wrong or right.

- If the AI detects that external truth conflicts with the internal Knowledge Base/Sample Answer, it will append two additional fields to the JSON response:

```json
{
  "id": "string",
  "isCorrect": "boolean",
  "feedback": "string",
  "isConflict": true,
  "whyConflict": "string (Explanation of the discrepancy between user knowledge and system knowledge)"
}
```

- The backend will save these conflict flags in the D1 database. This ensures that in future updates, Operators can build queries to surface controversial questions for manual Admin review.

## 6. Non-Functional Requirements & Architecture

### 6.1 Technology Stack

    Frontend: React (deployed via Cloudflare Pages). Focus on HTML/CSS/JS stability.

    Backend: Cloudflare Workers (Node.js/TypeScript).

    Relational Database: Cloudflare D1 (Users, Quizzes, Quiz Logs, Note Metadata, Conflict Flags).

    Object Storage: Cloudflare R2 (Raw .md, .docx, .txt files).

    Vector Database: Cloudflare Vectorize (Embedded chunks of study materials).

    AI Engine: Gemini 2.5 Flash API.

### 6.2 Rate Limit Management

Because the Gemini Free Tier restricts usage to 15 Requests Per Minute, concurrent submissions by the 12-person team could trigger HTTP 429 Too Many Requests errors.

    Frontend Mitigation: Upon clicking "Submit", the UI must immediately disable the button and display a loading state (e.g., "AI is evaluating...").

    Backend Mitigation (Exponential Backoff): Cloudflare Workers must wrap the Gemini API calls in a robust retry queue. If a 429 error is intercepted, the Worker will not return an error to the user. Instead, it will pause (sleep) for 3-5 seconds and retry the request automatically up to 3 times before failing gracefully.

### 6.3 Performance Expectations

    Frontend interactions (switching sections, autosaving) must be instantaneous (< 200ms).

    AI SAQ grading response time should generally conclude within 10-15 seconds, factoring in retrieval, API latency, and potential backoff delays.

## 7. Out of Scope (Phase 2 Considerations)

To ensure rapid development and deployment of the core learning loop, the following features are explicitly excluded from the current phase:

    Reporting and Dashboards: Global progress charts, comparative analytics among team members, and detailed individual performance matrixes (identifying weak topics or tracking weekly reading completions) are deferred. All data (scores, conflicts, timestamps) will be securely logged in Cloudflare D1 during Phase 1 to ensure immediate availability for Dashboard implementation in Phase 2.
