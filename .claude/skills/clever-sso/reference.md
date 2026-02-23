# Clever SSO Technical Reference

This file contains detailed technical reference for the Clever SSO integration skill. The SKILL.md file contains the main instructions; this file provides deeper context for edge cases and advanced scenarios.

## Complete OAuth 2.0 Flow Diagram

```
User clicks "Log in with Clever" or Clever Portal icon
    |
    v
GET https://clever.com/oauth/authorize
    ?response_type=code
    &client_id={CLIENT_ID}
    &redirect_uri={REDIRECT_URI}
    [&district_id={DISTRICT_ID}]       (optional: skip district picker)
    [&confirmed=false]                  (optional: force identity confirm)
    [&state={CSRF_TOKEN}]              (optional: CSRF protection)
    |
    v
Clever authenticates user (or uses existing session)
    |
    v
Clever redirects to: {REDIRECT_URI}?code={AUTH_CODE}
    |
    v
Server exchanges code for token:
POST https://clever.com/oauth/tokens
    Headers:
      Authorization: Basic base64({CLIENT_ID}:{CLIENT_SECRET})
      Content-Type: application/json
    Body:
      {"code": "{AUTH_CODE}", "grant_type": "authorization_code", "redirect_uri": "{REDIRECT_URI}"}
    |
    v
Response: {"access_token": "il..."}
    |
    v
GET https://api.clever.com/v3.0/me
    Headers: Authorization: Bearer {ACCESS_TOKEN}
    Response: {"type": "student", "data": {"id": "...", "district": "..."}}
    |
    v
GET https://api.clever.com/v3.0/users/{id}
    Headers: Authorization: Bearer {ACCESS_TOKEN}
    Response: {"data": {"id": "...", "name": {"first": "...", "last": "..."}, "email": "...", "roles": [...]}}
    |
    v
Match or create user in your database, create session
```

## OIDC Flow (Alternative to plain OAuth)

OIDC adds an ID token (JWT) to the token response, eliminating the need for the /me API call.

```
Same authorization redirect as OAuth
    |
    v
Token exchange response includes BOTH:
  - access_token (opaque string, same as OAuth)
  - id_token (JWT, decodable)
    |
    v
Decode the id_token JWT to extract:
  - sub / multi_role_user_id  -> Clever user ID
  - user_type                 -> student/teacher/staff/district_admin
  - district                  -> district ID
  - email                     -> may be null
  - given_name / family_name  -> user's name
  - iss                       -> always "https://clever.com"
  - aud                       -> your Client ID
  - exp                       -> 1 hour from issuance
```

OIDC Discovery: `https://clever.com/.well-known/openid-configuration`

## API Response Formats

### GET /v3.0/me (Student)
```json
{
  "type": "student",
  "data": {
    "id": "5a4fcb9a1b2c3d4e5f6a7b8c",
    "district": "4fd43cc56d11340000000005"
  },
  "links": [
    {"rel": "self", "uri": "/me"},
    {"rel": "canonical", "uri": "/v3.0/users/5a4fcb9a1b2c3d4e5f6a7b8c"}
  ]
}
```

### GET /v3.0/me (Teacher)
```json
{
  "type": "teacher",
  "data": {
    "id": "5a4fcb9a1b2c3d4e5f6a7b8d",
    "district": "4fd43cc56d11340000000005"
  },
  "links": [
    {"rel": "self", "uri": "/me"},
    {"rel": "canonical", "uri": "/v3.0/users/5a4fcb9a1b2c3d4e5f6a7b8d"}
  ]
}
```

### GET /v3.0/users/{id}
```json
{
  "data": {
    "id": "5a4fcb9a1b2c3d4e5f6a7b8c",
    "district": "4fd43cc56d11340000000005",
    "email": "student@example.com",
    "name": {
      "first": "Jane",
      "middle": "M",
      "last": "Doe"
    },
    "roles": {
      "student": {
        "legacy_id": "legacy123",
        "sis_id": "SIS456",
        "school": "school789",
        "grade": "5",
        "schools": ["school789"]
      }
    },
    "created": "2023-01-15T10:30:00.000Z",
    "last_modified": "2023-06-20T14:22:00.000Z"
  },
  "links": [
    {"rel": "self", "uri": "/v3.0/users/5a4fcb9a1b2c3d4e5f6a7b8c"}
  ]
}
```

### GET /v3.0/districts/{id}
```json
{
  "data": {
    "id": "4fd43cc56d11340000000005",
    "name": "Springfield School District",
    "mdr_number": "123456",
    "nces_id": "0123456"
  }
}
```

## Multi-Role User Handling

A single person can hold multiple roles in Clever (e.g., a teacher who is also a school admin). The API handles this as follows:

- The `/me` endpoint returns the user's primary type and a single `id`
- The `/users/{id}` endpoint returns a `roles` object with entries for each role
- Each role has its own `legacy_id` (used in API versions before v3.0)
- The top-level `id` is the multi-role user ID (consistent across roles in v3.0+)

```json
{
  "data": {
    "id": "unified_id_here",
    "roles": {
      "teacher": {
        "legacy_id": "teacher_legacy_id",
        "school": "school_id",
        "schools": ["school_id"]
      },
      "school_admin": {
        "legacy_id": "admin_legacy_id",
        "school": "school_id",
        "schools": ["school_id"]
      }
    }
  }
}
```

Always use the top-level `id`, never the `legacy_id`, for user matching.

## Instant Login Links

For districts that want to embed direct links to your app:

```
https://clever.com/oauth/instant-login?client_id={CLIENT_ID}&district_id={DISTRICT_ID}
```

This skips the Clever Portal and goes directly to authentication for that district.

Note: Instant Login does NOT work with Library integrations.

## Error Responses

### Token Exchange Errors
```json
{"error": "invalid_grant", "error_description": "code has been used or has expired"}
```
Cause: Authorization code was already exchanged or took too long (>60 seconds).

```json
{"error": "invalid_client"}
```
Cause: Wrong Client ID or Client Secret in the Basic auth header.

```json
{"error": "redirect_uri_mismatch"}
```
Cause: The redirect_uri in the token request doesn't match what's configured in the dashboard.

### Callback Errors
If the user's district hasn't authorized your app, Clever redirects to:
```
{REDIRECT_URI}?error=unauthorized-user
```

Handle this gracefully: show a message like "Your school hasn't set up access to [App Name] yet."

## SSO Button Assets

Official Clever branded buttons (use SVG versions):

| Style | URL |
|---|---|
| Log in with Clever (white bg) | `https://files.readme.io/99d7b57-white-liwc.svg` |
| Log in with Clever (blue bg) | `https://files.readme.io/d1a4142-blue-liwc.svg` |
| Sign up with Clever (white bg) | `https://files.readme.io/42e3523-white-suwc.svg` |
| Sign up with Clever (blue bg) | `https://files.readme.io/00498c0-blue-suwc.svg` |

## Clever Developer Dashboard URLs

| Page | URL |
|---|---|
| Dashboard home | https://apps.clever.com |
| Sign up | https://apps.clever.com/signup |
| Clever Academy | https://clever.com/academy |
| API status | https://status.clever.com |
| Support | https://support.clever.com |
| Dev docs | https://dev.clever.com |

## District SSO Scopes

These scopes are automatically granted for District SSO integrations:
- `read:district_admins_basic` — access district admin profiles
- `read:school_admins_basic` — access school admin profiles
- `read:students_basic` — access student profiles
- `read:teachers_basic` — access teacher profiles
- `read:user_id` — access Clever user IDs

## Token Characteristics

| Property | Access Token | ID Token (OIDC) |
|---|---|---|
| Format | Opaque string (prefix: `il`) | JWT |
| Decodable | No | Yes |
| Expiry | 24 hours | 1 hour |
| Refreshable | No | No |
| Scope | User's own data only | N/A (contains claims) |
| Endpoints accessible | /me, /users/{id}, /districts/{id} | N/A |

## Framework-Specific Notes

### Next.js / NextAuth
- Add a custom Clever provider to NextAuth configuration
- Use the `credentials` or custom OAuth provider pattern
- Handle the callback in `/api/auth/callback/clever`

### Express / Passport
- Create a `passport-clever` strategy or use `passport-oauth2` with Clever endpoints
- Register the strategy with Clever's authorization and token URLs

### Django
- Use `social-auth-app-django` with a custom Clever backend
- Or implement manually with `requests` library in a view

### Rails / Devise
- Use OmniAuth with a custom Clever strategy
- Or use the `omniauth-oauth2` gem configured for Clever endpoints

### Flask
- Use Flask-Dance or Authlib with Clever OAuth configuration
- Or implement manually with the `requests` library

### Laravel
- Use Laravel Socialite with a custom Clever provider
- Or implement manually in a controller
