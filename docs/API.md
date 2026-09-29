# API Design

This document describes the REST API for Toremon App: how the frontend and backend talk to
each other, how authentication works, and what each endpoint does.

It is written to be readable by someone learning application architecture, so it explains
the reasoning behind the design as well as the design itself.

Related documents:

- [REQUIREMENTS.md](./REQUIREMENTS.md) — the product rules this API enforces.
- [DATABASE.md](./DATABASE.md) — the tables and fields behind these resources.
- [TECH_STACK.md](./TECH_STACK.md) — the technology choices.

---

## 1. API Overview

### Architecture

```
React frontend
      │
      │  REST API  (HTTP + JSON)
      ▼
Laravel backend
      │
      ▼
   MySQL
```

The frontend communicates with Laravel **only** through REST API endpoints. It never
communicates directly with MySQL. The browser has no database connection, no credentials,
and no way to reach the database except by asking the API.

This matters because it gives the system exactly one door. Every read and every write passes
through Laravel, so there is a single place where rules are enforced.

### What Laravel is responsible for

| Responsibility | Meaning |
| --- | --- |
| Authentication | Deciding **who** the caller is |
| Authorization | Deciding **what** that caller is allowed to touch |
| Validation | Rejecting malformed or invalid input |
| Business rules | Enforcing limits such as 5 active task lists and 100 tasks per list |
| Database access | Reading from and writing to MySQL |

### API prefix

Every endpoint in this document is served under the `/api` prefix.

```
/api/auth/login
/api/task-lists
/api/task-lists/{taskList}/tasks
```

---

## 2. Authentication

### Authentication vs authorization

These two words sound similar and are easy to confuse, so it is worth separating them
clearly.

| Aspect | Authentication | Authorization |
| --- | --- | --- |
| Question it answers | Who are you? | Are you allowed to do this? |
| Happens | First | After authentication |
| Example failure | You are not logged in | You are logged in, but this task list is not yours |
| Status code | `401 Unauthorized` | `403 Forbidden` |

Authentication proves identity. Authorization checks permission. A request can pass
authentication and still fail authorization.

### The token mechanism

Toremon App uses an **access token + refresh token** pair. Both are in **JWT** format.

| Token | Lifetime | Stored where | Sent how |
| --- | --- | --- | --- |
| Access token | 15 minutes (short-lived) | Frontend memory / Zustand | `Authorization: Bearer <access_token>` header |
| Refresh token | 7 days (long-lived) | HttpOnly, Secure, SameSite cookie | Automatically by the browser |

**Why two tokens?** A single long-lived token would be convenient but dangerous: if it
leaked, an attacker could use it for as long as it stayed valid. A single short-lived token
would be safe but annoying, because the user would have to log in every 15 minutes. Splitting
the two roles gives both properties. The token sent on every request is short-lived and
therefore low-value if stolen, while the long-lived token is kept somewhere JavaScript cannot
read.

**The refresh token is not directly accessible to JavaScript.** It lives in a cookie marked:

- `HttpOnly` — JavaScript cannot read it via `document.cookie`.
- `Secure` — it is only sent over HTTPS.
- `SameSite` — it is not sent along with cross-site requests.

**Refresh token rotation is not implemented in the initial version.** Rotation means issuing
a brand-new refresh token every time one is used and invalidating the old one. It is a
worthwhile hardening step, but it is deliberately out of scope for now. The same refresh
token stays valid for its full 7 days.

---

### Authentication endpoints

#### POST /api/auth/register

| Property | Value |
| --- | --- |
| **Purpose** | Create a new user account and sign that user in |
| **Method** | `POST` |
| **Auth required** | No — this endpoint is public |

Required fields: `name`, `email`, `password`. The input is validated, the email must be
unique, and the password is **securely hashed before being stored**. A plain-text password is
never written to the database.

After a successful registration the user is **automatically authenticated**, so there is no
need to call the login endpoint afterwards.

Request body:

```json
{
  "name": "Ada Lovelace",
  "email": "ada@example.com",
  "password": "her-secret-password"
}
```

Successful response — `201 Created`:

```json
{
  "access_token": "<jwt-access-token>",
  "user": {
    "id": 123,
    "name": "Ada Lovelace",
    "email": "ada@example.com"
  }
}
```

The refresh token is **not** in the response body. It is set as an HttpOnly cookie on the
same response.

Important error cases:

| Status | When |
| --- | --- |
| `422 Unprocessable Entity` | A required field is missing or invalid |
| `422 Unprocessable Entity` | The email is already registered |

#### POST /api/auth/login

| Property | Value |
| --- | --- |
| **Purpose** | Authenticate an existing user |
| **Method** | `POST` |
| **Auth required** | No — this endpoint is public |

The user provides an email and password. Laravel verifies the credentials, returns an access
token, and sets the refresh token as an HttpOnly cookie.

Request body:

```json
{
  "email": "ada@example.com",
  "password": "her-secret-password"
}
```

Successful response — `200 OK`:

```json
{
  "access_token": "<jwt-access-token>",
  "user": {
    "id": 123,
    "name": "Ada Lovelace",
    "email": "ada@example.com"
  }
}
```

Important error cases:

| Status | When |
| --- | --- |
| `422 Unprocessable Entity` | Email or password missing or malformed |
| `401 Unauthorized` | The credentials do not match a user |

#### POST /api/auth/refresh

| Property | Value |
| --- | --- |
| **Purpose** | Exchange a valid refresh token for a new access token |
| **Method** | `POST` |
| **Auth required** | No access token required — the refresh-token cookie is used instead |

This endpoint **does not require the access token**, which is the whole point: it is called
precisely when the access token has expired.

The browser sends the refresh-token cookie automatically, so the frontend does not need to
read it, attach it, or even know its value. Laravel validates the refresh token and, if it is
valid, returns a new access token.

Request body: *(none)*

Successful response — `200 OK`:

```json
{
  "access_token": "<new-jwt-access-token>"
}
```

Important error cases:

| Status | When |
| --- | --- |
| `401 Unauthorized` | The refresh cookie is missing, invalid, or expired |

When this happens, the session is over and the user must log in again.

#### POST /api/auth/logout

| Property | Value |
| --- | --- |
| **Purpose** | End the session |
| **Method** | `POST` |
| **Auth required** | The refresh-token cookie is sent automatically by the browser |

Logging out has two halves:

- The backend **clears/removes the refresh-token cookie** from the browser.
- The frontend **removes the access token** from memory / Zustand.

Both halves matter. Clearing only the frontend would leave the refresh cookie in the browser;
clearing only the cookie would leave a usable access token in memory until it expires.

Request body: *(none)*

Successful response — `204 No Content`

#### Known limitation of logout

Logout removes the cookie from the browser. It does **not** revoke the token itself.

**Refresh-token rotation and server-side revocation are not implemented in the initial
version.** There is no stored list of issued or revoked refresh tokens, so the backend has no
way to reject a refresh token it has already signed — it only checks that the signature and
expiry are valid.

The consequence is worth stating plainly: **if a refresh token were stolen before logout, it
could technically remain valid until its 7-day expiration**, even after the user has logged
out. Logging out protects the browser that logged out, not a copy taken elsewhere.

This is a known security limitation of a deliberately simplified learning implementation, not
an oversight. Addressing it would require server-side token tracking or rotation, which is out
of scope at this stage.

---

## 3. Token Flow

### Registering or logging in

```
Frontend                         Laravel
   │                                │
   │  POST /api/auth/register       │
   │       or /api/auth/login       │
   │───────────────────────────────▶│
   │                                │  validates credentials
   │                                │
   │  200/201                       │
   │  { access_token }              │
   │  Set-Cookie: refresh_token     │
   │     (HttpOnly, Secure,         │
   │      SameSite)                 │
   │◀───────────────────────────────│
   │                                │
   │  stores access token           │
   │  in memory / Zustand           │
```

### A normal API request

Every request to a protected endpoint carries the access token in a header:

```http
GET /api/task-lists HTTP/1.1
Authorization: Bearer <access_token>
Content-Type: application/json
```

### When the access token expires

Access tokens last 15 minutes, so this is a routine event, not an error condition.

```
Frontend                         Laravel
   │                                │
   │  GET /api/task-lists           │
   │  Authorization: Bearer <old>   │
   │───────────────────────────────▶│
   │           401 Unauthorized     │
   │◀───────────────────────────────│
   │                                │
   │  POST /api/auth/refresh        │
   │  (browser attaches the         │
   │   refresh cookie by itself)    │
   │───────────────────────────────▶│
   │                                │  validates refresh token
   │           200 { access_token } │
   │◀───────────────────────────────│
   │                                │
   │  stores new access token       │
   │  in memory, retries the        │
   │  original request              │
   │───────────────────────────────▶│
```

If the refresh call also fails with `401`, the refresh token is gone or expired, and the user
is sent back to the login page.

### Why the access token is not stored in localStorage

The access token is held **in memory only** — in a Zustand store, which lives in the
JavaScript heap and disappears when the tab is closed or reloaded.

`localStorage` is readable by any JavaScript running on the page. If an attacker manages to
execute a script in the app (a cross-site scripting bug, a compromised dependency), anything
in `localStorage` can be read and exfiltrated. A token held in memory is harder to reach and
disappears on reload, and the refresh token is out of reach entirely because `HttpOnly`
cookies are invisible to JavaScript.

The trade-off is that a page reload loses the access token — which is exactly what the
refresh endpoint is for.

---

## 4. JWT

### JWT is a format, not a lifecycle

These two ideas are easy to mix up:

- **JWT** describes **what the token looks like** — a signed token format that carries a set
  of claims, which the server can verify using the signature.
- **Access token** and **refresh token** describe **what a token is for** and how long it
  lives.

Both of Toremon's tokens are JWTs. The difference between them is their purpose and lifetime,
not their format.

### What a verified token does and does not tell the backend

Verifying the access token's signature tells Laravel **who the caller is**: the `sub` claim
identifies the authenticated user. That part needs no database lookup.

It does **not** tell Laravel whether that user may touch a particular task list or task.
Ownership is a fact about the data, not something carried in the token — and the token is
issued at login, long before the user requests any specific resource.

So **resource authorization still requires checking ownership**. Laravel may need to query the
database to confirm that the requested task list or task belongs to the user identified by
`sub`. Authentication comes from the token; authorization comes from the data.

### Simplified payload

```json
{
  "sub": 123,
  "type": "access",
  "iat": 1234567890,
  "exp": 1234568790
}
```

| Claim | Meaning |
| --- | --- |
| `sub` | Subject — the authenticated user's ID |
| `type` | Which kind of token this is: `access` or `refresh` |
| `iat` | Issued-at time |
| `exp` | Expiration time |

The `type` claim is what stops a refresh token from being used as an access token, or the
reverse. Each endpoint checks that it received the kind of token it expects.

### Never put secrets in a payload

A JWT payload is **signed, not encrypted**. The signature proves the payload was not
tampered with, but anyone holding the token can decode and read its contents.

Therefore **passwords and sensitive credentials must never be placed in a JWT payload.** The
payload should carry only what the server needs to identify the caller and the token, as
shown above.

---

## 5. Authorization

Toremon uses **resource-based authorization**: permission depends on who owns the specific
record being requested.

The backend identifies the authenticated user from the access token (the `sub` claim), then
verifies that the requested resource belongs to that user.

```
User A  ──▶  Task List 10   (owned by User A)   ──▶  allowed
User B  ──▶  Task List 10   (owned by User A)   ──▶  denied (403)
```

The same check applies to tasks, one step removed: a task belongs to a task list, and that
task list belongs to a user.

### Three rules that make this safe

**1. Never trust a `user_id` sent by the frontend as proof of ownership.** Anything in a
request body or query string is controlled by the caller and can be changed freely. If the
backend trusted a `user_id` field, any user could claim to be any other user. Ownership is
always derived from the verified access token, never from the request payload.

**2. Backend authorization is the security boundary.** It is the only check that actually
prevents access. Everything in front of it is convenience.

**3. Hiding UI elements is not authorization.** A hidden button is still reachable — anyone
can call the API directly with `curl`, browser devtools, or a script. Hiding controls is good
UX because it stops users from attempting actions that would fail, but it protects nothing.

---

## 6. Task List API

All task list endpoints **require authentication**, and a user can only access their own task
lists. A request for someone else's task list is rejected with `403 Forbidden`.

### Rules enforced by these endpoints

- A user can have a maximum of **5 active task lists**.
- If the user already has 5 active task lists, creating another is rejected.
- Deleting a task list decreases the active count, so the user can then create another one.
- There is **no daily creation limit**.
- The task list name is required and cannot be empty.
- The task list name has a maximum length of **100 characters**.
- A task list may contain zero tasks.
- Deleting a task list also deletes all of its tasks.

### GET /api/task-lists

| Property | Value |
| --- | --- |
| **Purpose** | List the authenticated user's task lists |
| **Auth required** | Yes |

Returns only the current user's task lists. Successful response — `200 OK`:

```json
{
  "data": [
    {
      "id": 10,
      "name": "Groceries",
      "created_at": "2026-09-28T10:15:00Z",
      "updated_at": "2026-09-28T10:15:00Z"
    }
  ]
}
```

A user with no task lists receives an empty `data` array, not an error.

### POST /api/task-lists

| Property | Value |
| --- | --- |
| **Purpose** | Create a task list |
| **Auth required** | Yes |

Request body:

```json
{
  "name": "Groceries"
}
```

Successful response — `201 Created`:

```json
{
  "data": {
    "id": 10,
    "name": "Groceries",
    "created_at": "2026-09-28T10:15:00Z",
    "updated_at": "2026-09-28T10:15:00Z"
  }
}
```

Important error cases:

| Status | When |
| --- | --- |
| `401 Unauthorized` | No valid access token |
| `422 Unprocessable Entity` | `name` missing, empty, or longer than 100 characters |
| `422 Unprocessable Entity` | The user already has 5 active task lists |

### GET /api/task-lists/{taskList}

| Property | Value |
| --- | --- |
| **Purpose** | Retrieve a single task list |
| **Auth required** | Yes |

Successful response — `200 OK`, same shape as a single item above.

Important error cases:

| Status | When |
| --- | --- |
| `401 Unauthorized` | No valid access token |
| `403 Forbidden` | The task list belongs to another user |
| `404 Not Found` | No task list with that ID exists |

### PATCH /api/task-lists/{taskList}

| Property | Value |
| --- | --- |
| **Purpose** | Rename a task list |
| **Auth required** | Yes |

Request body:

```json
{
  "name": "Weekly groceries"
}
```

Successful response — `200 OK`, returning the updated task list.

Important error cases:

| Status | When |
| --- | --- |
| `401 Unauthorized` | No valid access token |
| `403 Forbidden` | The task list belongs to another user |
| `404 Not Found` | No task list with that ID exists |
| `422 Unprocessable Entity` | `name` missing, empty, or longer than 100 characters |

### DELETE /api/task-lists/{taskList}

| Property | Value |
| --- | --- |
| **Purpose** | Delete a task list and all of its tasks |
| **Auth required** | Yes |

Successful response — `204 No Content`.

Deleting a task list frees a slot against the 5-task-list limit, so the user can immediately
create a new one.

Important error cases:

| Status | When |
| --- | --- |
| `401 Unauthorized` | No valid access token |
| `403 Forbidden` | The task list belongs to another user |
| `404 Not Found` | No task list with that ID exists |

---

## 7. Task API

All task endpoints **require authentication**, and users can only access tasks that belong to
their own task lists.

Notice the URL shapes. Creating and listing tasks happens **inside a task list**
(`/api/task-lists/{taskList}/tasks`), because that is the context a task needs. Updating and
deleting an existing task addresses it directly (`/api/tasks/{task}`), because a task ID is
already unique on its own.

### Rules enforced by these endpoints

- A task list can contain a maximum of **100 tasks**.
- The task title is required and cannot be empty.
- The task title has a maximum length of **255 characters**.
- Tasks can be renamed and deleted.
- Tasks can be marked completed, and completed tasks can be marked incomplete again.
- Completion is a single boolean field, `completed`. There are **no** additional statuses
  such as `not_started`, `in_progress`, or `done`.

### GET /api/task-lists/{taskList}/tasks

| Property | Value |
| --- | --- |
| **Purpose** | List the tasks in a task list |
| **Auth required** | Yes |

Successful response — `200 OK`:

```json
{
  "data": [
    {
      "id": 55,
      "task_list_id": 10,
      "title": "Buy milk",
      "completed": false,
      "created_at": "2026-09-28T10:20:00Z",
      "updated_at": "2026-09-28T10:20:00Z"
    }
  ]
}
```

An empty task list returns an empty `data` array.

Important error cases:

| Status | When |
| --- | --- |
| `401 Unauthorized` | No valid access token |
| `403 Forbidden` | The task list belongs to another user |
| `404 Not Found` | No task list with that ID exists |

### POST /api/task-lists/{taskList}/tasks

| Property | Value |
| --- | --- |
| **Purpose** | Create a task inside a task list |
| **Auth required** | Yes |

Request body:

```json
{
  "title": "Buy milk"
}
```

Successful response — `201 Created`, returning the created task.

Important error cases:

| Status | When |
| --- | --- |
| `401 Unauthorized` | No valid access token |
| `403 Forbidden` | The task list belongs to another user |
| `404 Not Found` | No task list with that ID exists |
| `422 Unprocessable Entity` | `title` missing, empty, or longer than 255 characters |
| `422 Unprocessable Entity` | The task list already contains 100 tasks |

### PATCH /api/tasks/{task}

| Property | Value |
| --- | --- |
| **Purpose** | Rename a task, or change its completed state |
| **Auth required** | Yes |

This single endpoint covers both edits. Rename a task:

```json
{
  "title": "Buy oat milk"
}
```

Mark it completed:

```json
{
  "completed": true
}
```

Mark it incomplete again:

```json
{
  "completed": false
}
```

Because `completed` is a plain boolean, moving a task back to incomplete is the same kind of
operation as completing it — there is no separate "reopen" endpoint and no status workflow to
step through.

Successful response — `200 OK`, returning the updated task.

Important error cases:

| Status | When |
| --- | --- |
| `401 Unauthorized` | No valid access token |
| `403 Forbidden` | The task belongs to another user's task list |
| `404 Not Found` | No task with that ID exists |
| `422 Unprocessable Entity` | `title` empty or longer than 255 characters, or `completed` is not a boolean |

### DELETE /api/tasks/{task}

| Property | Value |
| --- | --- |
| **Purpose** | Delete a task |
| **Auth required** | Yes |

Successful response — `204 No Content`.

Important error cases:

| Status | When |
| --- | --- |
| `401 Unauthorized` | No valid access token |
| `403 Forbidden` | The task belongs to another user's task list |
| `404 Not Found` | No task with that ID exists |

---

## 8. Progress

Task list progress is **derived data**. It is calculated from the tasks in a task list and is
**not stored in the database**.

### Formula

```
progress = (completed tasks / total tasks) × 100
```

Example: a task list with 3 completed tasks out of 5 total tasks is at **60%**.

### Zero tasks

If a task list contains **zero tasks**, no progress percentage or progress bar is displayed.
There is nothing to divide by, and an empty task list is not "0% done" in any meaningful
sense — it simply has no progress to show.

### Not a database field

There is **no `progress` column** in the database, and no cached or denormalized progress
value. See [DATABASE.md](./DATABASE.md). Progress is computed from the underlying tasks
whenever it is needed.

Storing it would mean keeping it correct after every task creation, deletion, and completion
toggle — and any missed update would leave the stored number silently wrong. Calculating it
on demand cannot drift.

### Where progress is calculated — not yet decided

Progress is derived from task data and is not stored in the database. **Where that
calculation happens has not been decided yet.**

Two options are open: the API could compute progress and include it in a response, or the
frontend could compute it from the task data it already receives. Both are consistent with
the rule above, because neither stores the value.

This document therefore does **not** define any progress-related response fields. That
decision will be made later and documented then.

---

## 9. HTTP Methods and Status Codes

### Methods

| Method | Meaning | Example |
| --- | --- | --- |
| `GET` | Retrieve data. Does not change anything | `GET /api/task-lists` |
| `POST` | Create something new | `POST /api/task-lists` |
| `PATCH` | Update part of an existing resource | `PATCH /api/tasks/55` |
| `DELETE` | Delete a resource | `DELETE /api/tasks/55` |

### Status codes

| Code | Name | When it is used |
| --- | --- | --- |
| `200` | OK | The request succeeded and a body is returned |
| `201` | Created | A new resource was created |
| `204` | No Content | The request succeeded and there is nothing to return, as after a delete |
| `400` | Bad Request | The request itself is malformed, such as a body that is not valid JSON |
| `401` | Unauthorized | The caller is **not authenticated** — missing, invalid, or expired access token |
| `403` | Forbidden | The caller **is authenticated** but is not allowed to access this resource |
| `404` | Not Found | The requested resource does not exist |
| `422` | Unprocessable Entity | The request was well-formed, but the data failed validation or a business rule |

Three conventions are worth stating explicitly:

- **Validation errors use `422`**, not `400`. The request was understood; its contents were
  just not acceptable. `400` is reserved for requests the server could not parse at all.
- **Unauthenticated requests use `401`.** This is about identity: we do not know who you are.
- **Authenticated-but-not-allowed requests use `403` where appropriate.** This is about
  permission: we know who you are, and the answer is no.

The `401` / `403` distinction is the same one drawn in [section 2](#2-authentication).

No custom or non-standard status codes are used.

---

## 10. Error Response Format

Every error response uses the same JSON shape, so the frontend can handle errors in one
place instead of special-casing each endpoint.

```json
{
  "message": "Validation failed",
  "errors": {
    "email": [
      "The email field is required."
    ]
  }
}
```

| Field | Type | Description |
| --- | --- | --- |
| `message` | string | A human-readable summary of what went wrong |
| `errors` | object | Field-by-field details. Each key is a field name; each value is an array of messages for that field |

`errors` is an array per field because one field can fail several rules at once. A simple
error with nothing field-specific to report may include only a `message`:

```json
{
  "message": "This action is unauthorized."
}
```

---

## 11. API Conventions

| Convention | Detail |
| --- | --- |
| Format | Request and response bodies are JSON |
| Content type | `Content-Type: application/json` |
| Authentication | `Authorization: Bearer <access_token>` on protected endpoints |
| Prefix | All endpoints live under `/api` |
| Identifiers | Resource IDs appear in the URL path, as in `/api/tasks/55` |
| Validation authority | Backend validation is authoritative |
| Frontend validation | Exists for user experience, and is not a security boundary |

The last two lines restate the rule from [REQUIREMENTS.md](./REQUIREMENTS.md): the frontend
may check input to give quick feedback, but the backend re-checks everything, because a
request can arrive from anywhere and never has to pass through the frontend at all.

---

## Endpoint Summary

| Method | Endpoint | Purpose | Auth |
| --- | --- | --- | --- |
| `POST` | `/api/auth/register` | Create an account and sign in | No |
| `POST` | `/api/auth/login` | Sign in | No |
| `POST` | `/api/auth/refresh` | Get a new access token | Refresh cookie |
| `POST` | `/api/auth/logout` | End the session | Refresh cookie |
| `GET` | `/api/task-lists` | List the user's task lists | Yes |
| `POST` | `/api/task-lists` | Create a task list | Yes |
| `GET` | `/api/task-lists/{taskList}` | Retrieve one task list | Yes |
| `PATCH` | `/api/task-lists/{taskList}` | Rename a task list | Yes |
| `DELETE` | `/api/task-lists/{taskList}` | Delete a task list and its tasks | Yes |
| `GET` | `/api/task-lists/{taskList}/tasks` | List the tasks in a task list | Yes |
| `POST` | `/api/task-lists/{taskList}/tasks` | Create a task in a task list | Yes |
| `PATCH` | `/api/tasks/{task}` | Rename a task or change its completed state | Yes |
| `DELETE` | `/api/tasks/{task}` | Delete a task | Yes |

"Auth: Yes" means a valid access token is required in the `Authorization` header.
