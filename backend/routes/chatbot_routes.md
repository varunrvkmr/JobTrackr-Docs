# Chat

This module defines the API endpoints for the AI-powered chat feature. The chatbot uses OpenAI's language models to answer user questions about tailoring their resumes to specific job descriptions. It maintains conversational context across sessions and uses dynamic resume/job inputs generated from the database.

## Overview

* **Blueprint name**: `chat`
* **Base path**: `/api/chat`
* **Authentication**: All routes require JWT authentication
* **Chat context**:

  * Sessions are maintained in-memory (not persisted across restarts)
  * Each session includes the user's resume, job description, and message history
* **Resume data** is assembled from:

  * `WorkHistory`
  * `EducationHistory`
  * `Project`
  * `Skill`

---

## Chat Session Routes

### `POST /start`

**Description**: Initializes a new chat session using the authenticated user's resume and a selected job description.

**Request Body**:

```json
{
  "job_id": 123,
  "question": "How can I tailor my resume for this role?"
}
```

**Behavior**:

* Builds a condensed resume from database entries
* Extracts and optimizes the job description
* Creates a new session with a system prompt tailored to the context
* Sends the user's question to OpenAI and returns the AI's response

**Returns**: A new session ID, the initial AI response, and session metadata

---

### `POST /<session_id>/message`

**Description**: Sends a new message within an existing session.

**Request Body**:

```json
{
  "message": "What specific experience should I highlight?"
}
```

**Behavior**:

* Appends the message to the session history
* Sends the entire context (limited to recent messages) to OpenAI
* Returns the assistant's reply

**Returns**: AI-generated response and message count

---

### `GET /<session_id>/history`

**Description**: Retrieves the message history for a session, excluding system prompts.

**Returns**: A list of messages including roles (`user`, `assistant`) and timestamps

---

### `GET /<session_id>/context`

**Description**: Returns the session's initial context, which includes the resume and job description used during session creation.

---

### `GET /sessions`

**Description**: Returns a list of all active sessions with metadata. Intended for debugging or administrative use.

---

### `GET /health`

**Description**: Health check endpoint to verify that the chatbot route is live and responsive.

---

## Internal Logic and Helpers

* **Resume Construction**:

  * Functions like `build_optimized_resume_from_db()` and its enhanced variant create text summaries using user data, token limits, and section prioritization.
* **Job Description Optimization**:

  * `smart_truncate_job_description()` and `optimize_job_description()` clean and truncate job descriptions based on token budgets.
* **Token Management**:

  * Uses `tiktoken` for accurate token counting
  * Sections are truncated or reformatted when necessary to meet context limits
* **Session Management**:

  * Each session is stored in an in-memory dictionary
  * Sessions contain all messages and the original context

---


## Endpoints and Their DB Interactions

### 1. `/api/chat/start` — `start_chat()`

* **Purpose**: Start or resume a chat session for a specific job.
* **DB interactions**:

  * Gets `UserProfileDB` using JWT identity.
  * Retrieves the relevant `JobPosting` by `job_id`.
  * Uses `ChatSession.get_or_create_for_job(...)` to fetch or create a `ChatSession`.
  * If new, it builds:

    * `optimized_resume` (from Work, Education, Projects, Skills tables)
    * `optimized_job_desc` (from JobPosting)
  * Adds a `ChatMessage` for the user's question.
  * Uses RAG to generate a response, then adds a second `ChatMessage` for the assistant.
  * Creates or updates a `ChatSessionMetrics` entry.
  * **Commits** all inserts/updates to the DB.

### 2. `/api/chat/<session_id>/message` — `send_message()`

* **Purpose**: Add a new message to an existing session.
* **DB interactions**:

  * Retrieves the existing `ChatSession`.
  * Adds the new `ChatMessage` (user).
  * Calls RAG, then saves the assistant’s `ChatMessage`.
  * Updates `ChatSession.updated_at` and `ChatSessionMetrics`.

### 3. `/api/chat/<session_id>/history` — `get_chat_history()`

* **Purpose**: Retrieve full chat history for a session.
* **DB interactions**:

  * Queries `ChatSession`.
  * Retrieves related `ChatMessage`s (excluding deleted/system ones).

### 4. `/api/chat/<session_id>/context` — `get_session_context()`

* **Purpose**: Return the `initial_context` (resume + job description).
* **DB interactions**:

  * Queries `ChatSession` and returns `initial_context`.

### 5. `/api/chat/sessions` — `list_sessions()`

* **Purpose**: List all active chat sessions for the current user.
* **DB interactions**:

  * Gets `UserProfileDB` from JWT.
  * Queries `ChatSession` by `user_profile_id`.

### 6. `/api/chat/<session_id>/deactivate` — `deactivate_session()`

* **Purpose**: Soft delete a session.
* **DB interactions**:

  * Retrieves and updates `ChatSession.is_active = False`.

---

### Summary of DB Operations:

* **Reads**: `UserProfileDB`, `JobPosting`, `ChatSession`, `ChatMessage`, `ChatSessionMetrics`, `WorkHistory`, `EducationHistory`, `Project`, `Skill`.
* **Writes/Inserts**:

  * `ChatSession` (if new)
  * `ChatMessage` (user and assistant)
  * `ChatSessionMetrics` (new or updated)
* **Updates**:

  * `ChatSession.updated_at`, `initial_context`, `is_active`
  * `ChatSessionMetrics.total_messages`, `total_tokens_used`

<img src="../assets/chatbot-db.svg" alt="Chat DB Schema" width="2000">
---
