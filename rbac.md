# RBAC Overview

- Production-style role-based access control (RBAC) for all APIs.
- Keycloak JWT roles are now the source of truth.
- Project access is consistently enforced using centralized validation.

## Business outcome

- Stronger access control with clear, predictable permissions.
- Reduced risk of unauthorized project access.
- Consistent behavior across APIs (read/write/delete scenarios).

## Roles and permissions

| Role | Scope | Read | Write | Delete |
|---|---|---|---|---|
| Admin | All projects | ✅ | ✅ | ✅ |
| Project Manager | Assigned projects | ✅ | ✅ | Own project |
| Reviewer | Assigned projects | ✅ | ✅ | Own project |
| Viewer | Assigned projects | ✅ | ❌ | ❌ |


- **Project creator can perform soft delete on their own project.**
- Delete behavior is intentionally guarded and audited.
- Create project is allowed for roles with write capability (`admin`, `project_manager`, `reviewer`).

## How identity and access work

1. User signs in via Keycloak.
2. Backend validates JWT and reads realm roles.
3. Roles are synced to DB for operational consistency.
4.  All APIs validates project access before processing request.

## Technical highlights

- Centralized RBAC policy module.
- API-level authorization checks on key routes.
- Reviewer assignments now linked by user foreign keys.
- Audit-safe and maintainable authorization design.

## Summary

- Easier to reason about who can do what.
- Cleaner governance and onboarding for users.
- Foundation ready for future role expansion without major rewrites.
