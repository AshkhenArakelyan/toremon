# Requirements

This document defines **what** Toremon App must do. It is the source of truth for the
product's business rules.

It deliberately does not describe how any of this is built — no database design, API
endpoints, screens, or framework specifics. For technology decisions, see
[TECH_STACK.md](./TECH_STACK.md).

Every rule below is confirmed. Nothing here should be changed, reinterpreted, or extended
without updating this document first.

---

## 1. Authentication

- Users can register.
- Users can log in.
- Users can log out.
- Unauthenticated users cannot access the task management application.
- Unauthenticated users are redirected to the login page.

---

## 2. Task Lists

### Creating a task list

- A user can create task lists.
- A task list name is required.
- A task list name cannot be empty.
- A task list name has a maximum length of 100 characters.

### Daily creation limit

- A user can create a maximum of 5 task lists per calendar day.
- The limit counts task lists **created that day**, not the number of task lists the user
  currently has.
- Deleting a task list does not restore the user's daily task list creation allowance.

### Managing a task list

- A user can rename a task list.
- A user can delete a task list.
- A task list can contain zero tasks.

### Deleting a task list

- Deleting a task list also deletes all tasks belonging to that task list.

---

## 3. Tasks

### Creating a task

- A user can create tasks inside a task list.
- A task title is required.
- A task title cannot be empty.
- A task title has a maximum length of 255 characters.
- A task list can contain a maximum of 100 tasks.

### Managing a task

- A user can rename a task.
- A user can delete a task.

### Completion

- A user can mark a task as completed.
- A completed task can be marked as incomplete again.

---

## 4. Progress

- Progress is calculated as:

```
progress = (completed tasks / total tasks) × 100
```

- Example: 3 completed tasks out of 5 tasks is 60%.
- Progress is derived data and must not be stored as a separate database field.
- If a task list contains zero tasks, no progress bar is displayed.

---

## 5. Authorization

- Users can only access their own task lists.
- Users can only create tasks inside their own task lists.
- Users can only access tasks belonging to their own task lists.
- Users cannot view, modify, or delete another user's task lists or tasks.

---

## 6. Validation Responsibility

- The backend is responsible for enforcing business rules and validation.
- The frontend may provide validation for user experience, but frontend validation is not a
  security or business-rule boundary.

---

## Constraint Summary

The numeric limits stated above, collected for quick reference:

| Constraint | Value |
| --- | --- |
| Maximum task list name length | 100 characters |
| Maximum task lists created per calendar day, per user | 5 |
| Maximum task title length | 255 characters |
| Maximum tasks per task list | 100 |
