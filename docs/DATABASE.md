# Database Design

This document describes the agreed database design for Toremon App.

It covers the entities, their fields, the relationships between them, and the boundary
between what the database enforces and what the application enforces. For the product
rules this design supports, see [REQUIREMENTS.md](./REQUIREMENTS.md); for the technology
decisions, see [TECH_STACK.md](./TECH_STACK.md).

This design is confirmed. No tables, fields, or rules beyond those documented here should
be added without updating this document first.

---

## Database

Toremon App uses **MySQL** as its relational database.

The design has three core entities:

- `users`
- `task_lists`
- `tasks`

Every table has an auto-incrementing integer primary key named `id`, and `created_at` and
`updated_at` timestamps.

---

## Entities

### users

| Field | Description |
| --- | --- |
| `id` | Primary key, auto-incrementing |
| `name` | Required |
| `email` | Required, unique |
| `password` | Required |
| `created_at` | Timestamp |
| `updated_at` | Timestamp |

Notes:

- `email` is unique across the table. No two users may share an email address.

### task_lists

| Field | Description |
| --- | --- |
| `id` | Primary key, auto-incrementing |
| `user_id` | Foreign key referencing `users.id` |
| `name` | Required, maximum 100 characters |
| `created_at` | Timestamp |
| `updated_at` | Timestamp |

Rules:

- A user can have a maximum of 5 active task lists at a time.
- A task list can contain zero tasks.
- Task lists use **hard deletion**. A deleted row is removed from the table.
- There is no `deleted_at` field. Soft deletes are not used.
- Deleting a task list deletes all tasks belonging to it.

### tasks

| Field | Description |
| --- | --- |
| `id` | Primary key, auto-incrementing |
| `task_list_id` | Foreign key referencing `task_lists.id` |
| `title` | Required, maximum 255 characters |
| `completed` | Boolean |
| `created_at` | Timestamp |
| `updated_at` | Timestamp |

Rules:

- A task list can contain a maximum of 100 tasks.
- A task can be completed or not completed. This state is held entirely by the `completed`
  boolean.
- A completed task can be marked as incomplete again, so `completed` can move in both
  directions.
- There is no separate `status` field such as `not_started`, `in_progress`, or `done`.
- Progress is derived data and is not stored as a database field.

---

## Relationships

- A **User** has many **Task Lists**.
- A **Task List** belongs to a **User**.
- A **Task List** has many **Tasks**.
- A **Task** belongs to a **Task List**.

```
users
  └── task_lists        (users.id → task_lists.user_id)
        └── tasks       (task_lists.id → tasks.task_list_id)
```

Tasks are related to a user only indirectly, through the task list they belong to. There is
no direct link between `tasks` and `users`.

---

## Foreign Keys and Deletion

- `task_lists.user_id` references `users.id`.
- `tasks.task_list_id` references `task_lists.id`.
- Deleting a task list should **cascade-delete its tasks**. Removing a `task_lists` row
  removes every `tasks` row that references it, so no orphaned tasks remain.

Deletion of a user is intentionally left undefined here, because account deletion is not
part of the MVP.

---

## Database Responsibility vs Application Responsibility

The database and the application enforce different kinds of correctness, and the split is
deliberate.

**The database enforces structural integrity.** This is the kind of correctness that must
hold for the data to be well-formed at all:

- The uniqueness of `users.email`.
- Foreign key relationships between `task_lists` and `users`, and between `tasks` and
  `task_lists`.
- Cascade deletion of tasks when their task list is deleted.
- Required fields and the maximum lengths of `task_lists.name` and `tasks.title`.

**Laravel enforces application-level business rules.** These are counting and policy rules
that the schema cannot express:

- The maximum of 5 active task lists per user.
- The maximum of 100 tasks per task list.

**The database does not store calculated task-list progress.** Progress is derived from the
tasks belonging to a task list and is computed when needed. There is no progress column, and
no cached or denormalized progress value anywhere in the schema.

---

## Summary

| Table | Key fields | References |
| --- | --- | --- |
| `users` | `id`, `name`, `email` (unique), `password` | — |
| `task_lists` | `id`, `user_id`, `name` (max 100) | `users.id` |
| `tasks` | `id`, `task_list_id`, `title` (max 255), `completed` | `task_lists.id` (cascade delete) |
