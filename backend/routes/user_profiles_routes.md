# User Profile

This module defines the API endpoints for managing user profiles and related resume components, including education history, work history, projects, and skills. All operations are scoped to the authenticated user and are backed by multiple database tables.

## Overview

* **Blueprint name**: `user_profiles`
* **Base path**: Mounted under `/api/user-profiles`
* **Authentication**: Required for all modifying and user-specific data fetch routes
* **Models involved**:

  * `UserProfileDB`
  * `EducationHistory`
  * `WorkHistory`
  * `Project`
  * `Skill`

---

## Route Endpoints

### Profile

#### `GET /`

Fetches all user profiles (admin-level visibility).

#### `GET /me`

Fetches the currently authenticated user's profile, including:

* Basic profile information
* Associated `education_history` and `work_history` entries

#### `PUT /me`

Updates the authenticated user's profile fields. The `email` field is immutable.

**Updatable Fields**:
`first_name`, `last_name`, `phone`, `location`, `state`, `bio`, `linkedin`, `github`, `race`, `ethnicity`, `gender`, `disability_status`, `veteran_status`

#### `POST /newUser`

Creates a new user profile. Requires an `email` and accepts all standard profile fields.

---

### Education History

#### `POST /education`

Adds a new education entry to the authenticated user's profile.

#### `PUT /education/<int:edu_id>`

Updates an existing education entry. Ownership is validated.

#### `DELETE /education/<int:edu_id>`

Deletes the specified education entry. Only the profile owner may delete it.

---

### Work History

#### `POST /work`

Adds a new work history entry to the authenticated user's profile.

#### `PUT /work/<int:work_id>`

Updates an existing work entry. Ownership is validated.

#### `DELETE /work/<int:work_id>`

Deletes the specified work entry. Only the profile owner may delete it.

---

### Projects

#### `POST /projects`

Adds a new project to the authenticated user's profile.

#### `PUT /projects/<int:proj_id>`

Updates an existing project entry. Only the profile owner may update it.

#### `DELETE /projects/<int:proj_id>`

Deletes the specified project entry.

---

### Skills

#### `POST /skills`

Adds a new skill to the authenticated user's profile.

#### `PUT /skills/<int:skill_id>`

Updates an existing skill entry.

#### `DELETE /skills/<int:skill_id>`

Deletes the specified skill entry.

---

## Behavior Summary

* **Authorization**: All `POST`, `PUT`, and `DELETE` endpoints are protected using `@jwt_required()` and enforce ownership checks
* **Error Handling**: Returns appropriate status codes for missing resources, unauthorized access, or validation errors
* **Date Handling**: Dates are expected in ISO format (YYYY-MM-DD) and serialized the same way in responses

---