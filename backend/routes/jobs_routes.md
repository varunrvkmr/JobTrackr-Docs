# `jobs_routes.py`

This module defines the API endpoints related to job postings within the application. It uses a Flask `Blueprint` to group route handlers that manage CRUD operations for job data. All endpoints are secured via JWT authentication and scoped by the current user's identity.

## Overview

* **Blueprint name**: `job_bp`
* **Base path**: Typically mounted under `/api/jobs` by the main application
* **Authentication**: All routes require a valid JWT token and operate on the authenticated user's data
* **Model used**: `JobPosting` (SQLAlchemy ORM model)

---

## Route Endpoints

### `GET /getJobs`

**Description**: Fetches all job postings for the authenticated user.

**Returns**: A list of job entries, each with fields including `id`, `company`, `job_title`, `job_description`, `job_link`, `location`, `country`, and `posting_status`.

### `POST /addJob`

**Description**: Adds a new job posting.

**Request Body**:

* Required: `company`, `job_title`, `posting_status`
* Optional: `location`, `country`, `job_link`, `job_description`, `job_platform`, `date_applied` (ISO format: `YYYY-MM-DD`)

**Behavior**:

* Validates the input
* Parses `date_applied` if provided
* Defaults missing values
* Commits a new `JobPosting` to the database

**Returns**: A success message and the newly created job object on success. Returns error messages for missing fields or invalid formats.

### `DELETE /deleteJob/<int:job_id>`

**Description**: Deletes a job posting by its ID.

**Behavior**:

* Validates the job exists and belongs to the authenticated user
* Deletes the job if authorized

**Returns**: Success message or appropriate error if the job is not found or access is denied.

### `GET /getJob/<int:job_id>`

**Description**: Fetches a single job posting by ID.

**Returns**: A JSON object containing all relevant job fields if found and authorized. Otherwise returns 404 or 403 as appropriate.

### `PUT /updateJobStatus/<int:job_id>`

**Description**: Updates the `posting_status` field of a job posting.

**Request Body**:

```json
{
  "status": "Applied"
}
```

**Returns**: Updated status or an error if the job does not exist, the user is unauthorized, or input is missing or invalid.

### `PUT /updateJob/<int:job_id>`

**Description**: Updates multiple fields of a job posting.

**Request Body**: Accepts any combination of the following fields:

* `company`
* `job_title`
* `job_description`
* `job_link`
* `location`
* `country`
* `posting_status`
* `job_platform`
* `user_notes`
* `date_applied` (must be ISO format)

**Behavior**:

* Validates user ownership
* Parses and applies only provided fields
* Commits changes to the database

**Returns**: Success message and the updated job data, or an appropriate error message.

---

## Error Handling

Each route includes robust error handling. Common failure cases include:

* Malformed or missing request data
* Invalid JWT identity
* Job not found
* Unauthorized access to jobs not owned by the user
* Database commit failures or integrity errors

Detailed logs are written using `current_app.logger` to aid in debugging during development.