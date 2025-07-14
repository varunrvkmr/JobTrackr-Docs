# Autofill

This module defines the backend routes responsible for the autofill functionality used by the Chrome extension. It classifies input fields on job application forms and fills them with corresponding user profile data using embeddings, pattern matching, and TensorFlow-based models.

## Overview

* **Blueprint name**: `autofill`
* **Base path**: `/api/autofill`
* **Primary functions**:

  * Field classification using semantic similarity and context-aware thresholds
  * Autofill using user profile, education, and work history data
* **Storage**:

  * Canonical embeddings are loaded from a JSON file
  * CSV logs are maintained for classification and filling activity

---

## Route Endpoints

### `POST /classify`

Classifies HTML form fields by mapping them to canonical user profile fields using embedding similarity and contextual cues.

* Embeds field text using the Universal Sentence Encoder (USE)
* Applies dynamic thresholds based on field type, required status, label length, and pattern matches
* Returns accepted matches including selector, canonical field name, score, threshold, and confidence signals

---

### `POST /fill`

Fills form fields using the classified matches by extracting values from the user's profile, work history, or education history.

* Uses type-specific handlers to validate and format data
* Supports text, email, select, date, checkbox, radio, and other standard HTML input types
* Returns a list of filled values with selectors and confidence levels

---

### `POST /debug/threshold`

Debugging endpoint to view the calculated threshold and context features for a given form field. Used during development to refine classification behavior.

---

### `POST /feedback`

Logs feedback from the extension about whether a classified field was correct or incorrect. This is useful for improving future accuracy and auditability.

---

## Canonical Field Embeddings

The `classify` route relies on comparing incoming field representations with a set of canonical profile fields using vector embeddings. These are precomputed and loaded from a local JSON file.

* Embedding similarity is computed using cosine similarity
* Contextual features influence the acceptance threshold dynamically
* Logs are written to `logs/model_calls.csv` for monitoring and improvement

---

## Embedding Service: `embed_routes.py`

Autofill classification relies on the `get_embed_model()` function defined in the `embed_routes.py` module.

### What it does:

* Loads the **Universal Sentence Encoder (USE)** from a locally stored TensorFlow Hub format
* Embeds field labels, placeholders, and other textual cues into dense vectors
* Lazily initializes the model per worker at runtime
* Exposes two routes:

  * `GET /api/embed/health` to verify model availability
  * `POST /api/embed` to embed a list of input texts and return their embeddings

### Usage:

The `get_embed_model()` function is used internally by `/classify` to embed incoming form field descriptors in a batched, cached manner.

---

## Field Matching and Filling Logic

The autofill system uses a modular approach to handle different input types:

* `text`, `textarea`, `email`, `url` → `handle_text`
* `select`, `dropdown` → `semantic_select`
* `radio`, `checkbox` → `handle_radio`, `handle_checkbox`
* `date`, `number`, `tel` → Specialized formatters
* `file` → Not supported for filling

Each handler validates values against constraints like `pattern` and `maxlength`, and may fall back to lower-confidence methods if primary strategies fail.

---

## Notes

* Profile data is drawn from:

  * `UserProfileDB` (e.g., `first_name`)
  * First `WorkHistory` entry (e.g., `work_title`)
  * First `EducationHistory` entry (e.g., `education_degree`)
* The matching process is designed to be resilient across varying field naming conventions in real-world job forms

## Future Work
* In production, consider replacing in-memory caches and log files with persistent storage and analytics pipelines

---