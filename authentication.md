# Authentication

The authentication system in this application supports both traditional email/password login and federated identity through social logins using Okta. It uses a hybrid approach combining JWT tokens and HTTP-only cookies to manage secure sessions.

## Overview

* **JWT + Cookies**: Authentication is based on short-lived access tokens and long-lived refresh tokens, issued as JWTs and stored securely in HTTP-only cookies.
* **Social Login**: Users can log in using Gmail, Apple, or GitHub accounts through Okta (Auth0 integration), with fallback handling for various user info schemas.
* **User Management**: Registration and login routes create or retrieve `UserAuth` and `UserProfileDB` entries, mapping external identity providers to internal user records.

## Cookie-Based JWT Token Workflow

### Login (`POST /api/auth/login`)

Users provide their email and password. On successful login:

* A **15-minute access token** is issued and set as `access_token_cookie`
* A **30-day refresh token** is issued and set as `refresh_token_cookie` (scoped to `/api/auth/refresh`)

### Refresh Access Token (`POST /api/auth/refresh`)

Clients can obtain a new access token by presenting a valid `refresh_token_cookie`.

### Logout (`POST /api/auth/logout`)

Deletes both cookies, effectively logging the user out.

### Token-Protected Routes

Routes like `/check`, `/user/<id>`, and `/protected` require a valid access token via `@jwt_required()`.

## Registration (`POST /api/auth/register`)

This endpoint registers a user by creating:

* A `UserAuth` record (with hashed password)
* A `UserProfileDB` record linked by `user_auth_id`

It expects the following fields:

* Required: `username`, `email`, `password`, `first_name`, `last_name`
* Optional: `phone`, `location`

Duplicate emails are rejected.

## Social Login via Okta

Okta (via Auth0) enables users to sign in using Gmail, Apple, or GitHub.

### Initiate Login (`GET /api/auth/okta/login`)

Redirects the user to Okta's login screen using the `authorize_redirect` method with requested scopes: `openid profile email`.

### Handle Callback (`GET /api/auth/okta/callback`)

Handles the OAuth2 callback:

1. Exchanges the authorization code for tokens
2. Retrieves user information (email, name, sub)
3. If the user does not exist, creates both `UserAuth` and `UserProfileDB` entries
4. Issues access and refresh tokens and sets them in cookies
5. Redirects the user to the frontend

### Logout (`GET /api/auth/okta/logout`)

Clears the session and redirects the user to the configured `returnTo` URL via Okta logout.

## Token Storage and Security

* **Access tokens** are short-lived and scoped to all routes (`path="/"`)
* **Refresh tokens** are long-lived and scoped to the refresh endpoint
* **Cookies are HTTP-only**, which helps mitigate XSS attacks
* `secure=False` is used in development; in production, this should be set to `True` to ensure HTTPS-only cookie transmission

## Notes

* Email is the primary user identifier across both internal and federated systems.
* The application includes fallback logic for non-standard social login responses (e.g., handling missing `email` or `sub` fields).
* Tokens are created using `flask_jwt_extended` and stored using standard `set_cookie` or `set_access_cookies`.

---