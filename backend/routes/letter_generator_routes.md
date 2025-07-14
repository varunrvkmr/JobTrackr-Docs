# `letter_generator.py`

This module defines the API endpoints for generating cover letters, comparing user profiles against job descriptions, and supporting application preparation workflows. It integrates with the OpenAI API to provide AI-powered functionality and uses user-specific data from the database to tailor outputs.

## Overview

* **Blueprint name**: `letter_generator`
* **Base path**: `/api/letter-generator`
* **Authentication**: All data-fetching and comparison endpoints are secured via JWT authentication
* **Integrations**: Uses OpenAI's GPT model for text generation and comparison logic

---

## Route Endpoints

### `GET /<int:job_id>`

**Description**: Retrieves the details of a specific job posting, restricted to the authenticated user.

**Returns**: JSON with all relevant job fields including `company`, `job_title`, `posting_status`, `job_link`, `location`, `job_description`, `date_applied`, `job_platform`, and `user_notes`.

---

### `POST /compare-skills/<int:job_id>`

**Description**: Compares the user's profile (education, work experience, projects, and skills) against the job description for the specified job ID.

**Behavior**:

* Fetches the authenticated user's profile and resume records from the database
* Constructs structured representations of both the job and the user profile
* Uses OpenAI's function-calling API to identify:

  * Matched and missing skills
  * Matched and missing experience
  * Match percentage
  * Recommendations for improvement

**Returns**: A structured comparison result or error message if the operation fails.

---

### `POST /generate`

**Description**: Generates a tailored cover letter based on the provided job details and resume.

**Request Body**:

```json
{
  "job": { ... },
  "resume": "Stringified resume content"
}
```

**Behavior**:

* Optionally fetches the job description from the job link using web scraping
* Constructs a prompt for OpenAI using job details, job description, and resume
* Returns the generated cover letter

**Returns**: A `cover_letter` string or an error message.

---

## Supporting Functions

### `fetch_job_description(job_link)`

Scrapes the job description from a job posting webpage, typically used to enhance AI-generated outputs.

### `_compare_with_chatgpt(job, user)`

Internal utility that uses OpenAI's function-calling API to perform structured comparisons between job requirements and a candidate profile.

---

## Dependencies

* **OpenAI API**: Used to generate natural language output for comparisons, answers, and cover letters
* **BeautifulSoup + requests**: Used for HTML parsing and content extraction from job pages
* **User data models**: Includes `UserProfileDB`, `EducationHistory`, `WorkHistory`, `Project`, and `Skill`
