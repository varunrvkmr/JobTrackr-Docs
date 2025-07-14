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

## Notes

* Session data is not persisted between server restarts. Consider using Redis or a database-backed cache for production deployment.
* The OpenAI model used is `gpt-4.1-mini`.
* All user and job data are fetched securely using the authenticated user's `user_profile_id`.

## Future Work
* Consider using Redis or a database-backed cache for production deployment.
