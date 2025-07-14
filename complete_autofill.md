## Chrome Extension: Autofill and Job Scraping Flow

This page outlines the logic and flow behind the Chrome extension's `content.js` file, which powers both the autofill functionality and job scraping capability within supported job application pages.

### Overview

The content script is responsible for:

* **Autofill**: Detects input fields, sends classification data to the backend, receives mapped values, and injects them back into the page.
* **Job Scraping**: Gathers job metadata (title, company, description, etc.) based on platform-specific selectors and forwards the structured job data to the background script.

These flows are triggered via messages sent to the content script from either the extension popup or background context.

---
### Background Script: `background.js`

The `background.js` file serves as the message router and compute manager for the Chrome extension. It handles requests that cannot or should not be executed directly from the content script, such as embedding computation, network fetches, and persistent job saving.

This file is always running in the background and facilitates communication between the content script, popup interface, and any offscreen documents.

---

### Key Responsibilities

#### 1. TensorFlow\.js Embedding via Offscreen Document

To compute semantic embeddings (e.g., for field classification or value matching), the background script creates and manages an **offscreen document**:

* `createOffscreenDocument()` sets up an invisible background page (`offscreen.html`) with access to TensorFlow\.js.
* `handleEmbedding(text)` sends a message to this offscreen document to compute the embedding and listens for the response.

This offloads heavy operations from the content script and complies with Chrome extension security restrictions.

#### 2. Embedding Request Handling

When a message of type `COMPUTE_EMBEDDING` is received, the background script:

* Ensures the offscreen document is created
* Passes the text to the offscreen script
* Waits for and returns the embedding via a Promise resolution

This setup allows multiple concurrent embedding requests while maintaining a consistent interface.

#### 3. Job Save Requests

The background script listens for `SAVE_JOB` messages and performs a POST request to the backend API:

```
POST /api/jobs/addJob
```

It sends the `jobData` payload and returns the API’s success status, HTTP status code, and any error or response data. This decouples the saving logic from the content script and ensures CORS-compliant communication using extension permissions.

---

## Autofill Workflow

The autofill process is initiated when a `RUN_AUTOFILL` message is received.

#### 1. Field Extraction

The script scans the current page using `extractFields()`:

* Supports recursive scanning into same-origin iframes and shadow DOMs.
* Captures each field’s metadata:
  `selector`, `label`, `placeholder`, `type`, `required`, `name`, `id`, `pattern`, `maxlength`, and `options` (for selects).

#### 2. Field Classification

The metadata is sent to the backend at `/api/autofill/classify`.
This API returns a list of matched fields, each linked to a canonical profile field (e.g., `email`, `first_name`).

> **Reference:**
> [Backend Autofill Flow](./backend/routes/autofill_routes.md)

#### 3. Field Value Retrieval

The matched selectors are passed to `/api/autofill/fill`, which returns:

* Values to autofill
* Confidence level (`high`, `medium`)
* Optional warnings (e.g., conflicting values)

#### 4. Injection into Page

Each field is located on the page using `deepQuerySelector()` and updated with the fetched value.

Special handling includes:

* `select` dropdowns (including Select2-type dropdowns)
* `checkbox` and `radio` elements
* `contentEditable` elements

Visual indicators are applied:

* Green outline for high confidence
* Blue outline for medium confidence
* Orange outline for warnings

Events (`input`, `change`, `blur`) are dispatched to ensure frontend frameworks detect the changes.

#### 5. Feedback Buttons

For each filled field, 👍/👎 feedback buttons are inserted. Clicking a button:

* Sends a feedback payload to `/api/autofill/feedback`
* Stores a local copy using `chrome.storage.local`
* Displays visual confirmation and disables further input

#### 6. User Notification

After all fields are processed, a summary message is displayed on the page indicating the number of fields filled and their confidence levels.

---

## Job Scraping Workflow

Triggered by a `SAVE_JOB` message, the job scraping logic gathers structured job data using platform-specific selectors.

### Platform Detection

The script supports:

* LinkedIn
* Otta
* Jobright
* Wellfound

Platform is inferred from `window.location.hostname`.

### Data Extracted

* `company`
* `job_title`
* `location`
* `country`
* `job_link`
* `job_description`

  * Jobright-specific logic merges "Responsibilities" and "Qualifications"
* `posting_status` (e.g., "Saved")
* `job_platform`
* `date_applied`

The extracted job data is forwarded to the background script, which persists it to the backend or local storage.

---

## Utility Functions

| Function                | Purpose                                                      |
| ----------------------- | ------------------------------------------------------------ |
| `deepQuerySelector()`   | Recursively finds elements in DOM, iframes, and shadow roots |
| `extractFields()`       | Collects metadata from all visible form elements             |
| `uniqueCssSelector()`   | Builds a stable CSS selector for a given element             |
| `getNearestLabelText()` | Finds a human-readable label for input fields                |
| `notifyPage()`          | Displays toast-style notifications                           |
| `addFeedbackButtons()`  | Adds and manages inline thumbs up/down UI for feedback       |

---

### Interaction Summary

| Message Type          | Purpose                                | Triggered By            |
| --------------------- | -------------------------------------- | ----------------------- |
| `COMPUTE_EMBEDDING`   | Compute semantic embedding using TF.js | Content script          |
| `SAVE_JOB`            | Save structured job data to backend    | Content script          |

---
