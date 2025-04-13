# Project and API Documentation for `nest-rbac-auth`

## Overview

`nest-rbac-auth` is a NestJS-based monolith implementing Role-Based Access Control (RBAC) with Prisma and PostgreSQL. The project is designed to be modular, extensible, and database-agnostic (initially targeting PostgreSQL, with plans to support MySQL, SQLite, and others). The ultimate goal is to package it as an npm package (`@nest-rbac-auth/core`) for reusable RBAC functionality in NestJS applications.

This document outlines:
- What we’ll implement now (MVP for PostgreSQL).
- Future plans (additional databases, npm package, advanced features).
- All possible routes for the API.
- Detailed API documentation.

## Implementation Plan

### Phase 1: Current Implementation (MVP with PostgreSQL)

**Goal**: Build a functional RBAC system running on PostgreSQL, with a clean, modular structure that requires minimal refactoring for other databases.

**Features**:
- **User Management**:
  - Register user (email + hashed password).
  - Login user (returns JWT).
  - Get all users.
  - Delete user (soft delete with `deletedAt`).
- **Role Management**:
  - Add role.
  - Get all roles.
  - Delete role.
  - Assign role(s) to user.
  - Remove role from user.
- **Permission Management**:
  - Add permission (e.g., `read:post`, `delete:user`).
  - Get all permissions.
  - Assign permission(s) to role.
  - Remove permission(s) from role.
- **Access Check**:
  - Endpoint: `POST /access/check` to verify if a user has a specific permission.
- **Authentication**:
  - JWT-based authentication with a guard to protect routes.
  - Permissions guard (e.g., `@UseGuards(PermissionsGuard('delete:user'))`).
- **Database**:
  - Prisma ORM with PostgreSQL.
  - Schema supporting users, roles, permissions, and relationships.
  - Soft deletes for users, roles, and permissions.
- **Testing**:
  - E2e tests using Dockerized PostgreSQL.
  - Validate schema portability with SQLite locally.

**Tech Stack**:
- NestJS (TypeScript).
- Prisma ORM (PostgreSQL).
- JWT for authentication.
- bcrypt for password hashing.
- Docker for testing.

**Project Structure**:
```
src/
├── auth/              # Login, register, JWT strategy
├── users/             # User CRUD
├── roles/             # Role CRUD
├── permissions/       # Permission CRUD
├── access-control/    # Role-permission checks, /access/check
├── prisma/            # Prisma service and schema
├── common/            # Guards, decorators, utils
└── test/              # E2e tests
```

**Prisma Schema**:
```prisma
datasource db {
  provider = env("DATABASE_PROVIDER")
  url      = env("DATABASE_URL")
}

model User {
  id        String    @id @default(uuid())
  email     String    @unique
  password  String
  roles     UserRole[]
  deletedAt DateTime?
}

model Role {
  id          String         @id @default(uuid())
  name        String         @unique
  users       UserRole[]
  permissions RolePermission[]
  deletedAt   DateTime?
}

model Permission {
  id        String           @id @default(uuid())
  action    String           @unique
  roles     RolePermission[]
  deletedAt DateTime?
}

model UserRole {
  id     String @id @default(uuid())
  user   User   @relation(fields: [userId], references: [id])
  userId String
  role   Role   @relation(fields: [roleId], references: [id])
  roleId String
}

model RolePermission {
  id           String     @id @default(uuid())
  role         Role       @relation(fields: [roleId], references: [id])
  roleId       String
  permission   Permission @relation(fields: [permissionId], references: [id])
  permissionId String
}
```

**Environment Setup**:
- `.env` for database configuration:
  ```env
  DATABASE_PROVIDER=postgresql
  DATABASE_URL=postgresql://user:password@localhost:5432/nestrbac?schema=public
  JWT_SECRET=your-secret-key
  ```
- `.env.example` for reference.

**Routes** (detailed in API Documentation below):
- Auth: `/auth/register`, `/auth/login`.
- Users: `/users`, `/users/:id`.
- Roles: `/roles`, `/roles/:id`, `/roles/assign`, `/roles/remove`.
- Permissions: `/permissions`, `/permissions/:id`, `/permissions/assign`, `/permissions/remove`.
- Access Control: `/access/check`.

**Deliverables**:
- Fully functional RBAC system.
- E2e tests for all routes.
- PostgreSQL migrations.
- Documentation (this file).

### Phase 2: Future Implementation

**Goal**: Expand database support, add advanced features, and package as an npm package.

**Features**:
- **Database Support**:
  - Add MySQL and SQLite (minimal refactoring due to Prisma).
  - Test with SQL Server and CockroachDB (optional).
  - Explore MongoDB with a separate schema (no join tables).
- **Advanced RBAC**:
  - Role hierarchies (e.g., `admin` inherits `editor` permissions).
  - Permission wildcards (e.g., `read:*`).
  - Audit logs for role/permission changes.
- **Performance**:
  - Caching for permission checks (e.g., Redis via `@nestjs/cache-manager`).
  - Batch queries to reduce database load.
- **npm Package**:
  - Publish as `@nest-rbac-auth/core`.
  - Export modules: `AuthModule`, `UsersModule`, `RolesModule`, `PermissionsModule`, `AccessControlModule`.
  - Peer dependencies: `@nestjs/core`, `@prisma/client`.
  - CLI package (`@nest-rbac-auth/cli`) for setup:
    ```bash
    npx @nest-rbac-auth/cli init --db postgresql
    ```
  - Comprehensive docs with examples for each database.
- **Additional Features**:
  - Admin panel integration (e.g., endpoints for UI dashboards).
  - OpenAPI (Swagger) documentation.
  - Multi-tenant support (e.g., roles scoped to tenants).
- **Testing**:
  - CI/CD matrix testing for PostgreSQL, MySQL, SQLite.
  - Performance benchmarks for permission checks.

**Timeline**:
- After MVP is stable (Phase 1 complete).
- Prioritize MySQL/SQLite support first, then npm packaging.

## API Routes

Below are all possible routes for the `nest-rbac-auth` project, covering the MVP (Phase 1) and potential future endpoints (Phase 2).

### Authentication Routes
| Method | Route             | Description                     | Protected? | Required Permission |
|--------|-------------------|---------------------------------|------------|---------------------|
| POST   | `/auth/register`  | Register a new user             | No         | None                |
| POST   | `/auth/login`     | Login user, return JWT          | No         | None                |

### User Routes
| Method | Route             | Description                     | Protected? | Required Permission |
|--------|-------------------|---------------------------------|------------|---------------------|
| GET    | `/users`          | Get all users (non-deleted)     | Yes        | `read:user`         |
| DELETE | `/users/:id`      | Soft delete a user              | Yes        | `delete:user`       |

### Role Routes
| Method | Route             | Description                     | Protected? | Required Permission |
|--------|-------------------|---------------------------------|------------|---------------------|
| POST   | `/roles`          | Create a new role               | Yes        | `create:role`       |
| GET    | `/roles`          | Get all roles (non-deleted)     | Yes        | `read:role`         |
| DELETE | `/roles/:id`      | Soft delete a role              | Yes        | `delete:role`       |
| POST   | `/roles/assign`   | Assign role to user             | Yes        | `assign:role`       |
| POST   | `/roles/remove`   | Remove role from user           | Yes        | `assign:role`       |

### Permission Routes
| Method | Route                  | Description                        | Protected? | Required Permission    |
|--------|------------------------|------------------------------------|------------|------------------------|
| POST   | `/permissions`         | Create a new permission            | Yes        | `create:permission`    |
| GET    | `/permissions`         | Get all permissions (non-deleted)  | Yes        | `read:permission`      |
| POST   | `/permissions/assign`  | Assign permission to role          | Yes        | `assign:permission`    |
| POST   | `/permissions/remove`  | Remove permission from role        | Yes        | `assign:permission`    |

### Access Control Routes
| Method | Route             | Description                     | Protected? | Required Permission |
|--------|-------------------|---------------------------------|------------|---------------------|
| POST   | `/access/check`   | Check if user has permission    | Yes        | `check:access`      |

### Future Routes (Phase 2, Potential)
| Method | Route                     | Description                           | Protected? | Required Permission     |
|--------|---------------------------|---------------------------------------|------------|-------------------------|
| GET    | `/users/:id/roles`        | Get roles for a user                  | Yes        | `read:user`             |
| GET    | `/roles/:id/permissions`  | Get permissions for a role            | Yes        | `read:role`             |
| GET    | `/audit/logs`             | Get audit logs (e.g., role changes)   | Yes        | `read:audit`            |
| POST   | `/roles/hierarchy`        | Define role inheritance               | Yes        | `manage:role`           |

## API Documentation

### Base URL
`http://localhost:3000` (default for development)

### Authentication
- All protected routes require a JWT in the `Authorization` header:
  ```
  Authorization: Bearer <jwt-token>
  ```
- Obtain JWT via `/auth/login`.
- Permissions are enforced using a `PermissionsGuard` checking the user’s roles.

### Error Responses
- **400 Bad Request**: Invalid input (e.g., missing fields).
  ```json
  {
    "statusCode": 400,
    "message": "Email is required",
    "error": "Bad Request"
  }
  ```
- **401 Unauthorized**: Missing or invalid JWT.
  ```json
  {
    "statusCode": 401,
    "message": "Unauthorized"
  }
  ```
- **403 Forbidden**: User lacks required permission.
  ```json
  {
    "statusCode": 403,
    "message": "Forbidden"
  }
  ```
- **404 Not Found**: Resource doesn’t exist.
  ```json
  {
    "statusCode": 404,
    "message": "User not found"
  }
  ```

### Routes

#### POST /auth/register
Register a new user.

- **Request Body**:
  ```json
  {
    "email": "user@example.com",
    "password": "password123"
  }
  ```
- **Response (201 Created)**:
  ```json
  {
    "id": "uuid",
    "email": "user@example.com"
  }
  ```
- **Errors**: 400 (invalid email/password), 409 (email already exists).

#### POST /auth/login
Login user and return JWT.

- **Request Body**:
  ```json
  {
    "email": "user@example.com",
    "password": "password123"
  }
  ```
- **Response (200 OK)**:
  ```json
  {
    "access_token": "jwt-token"
  }
  ```
- **Errors**: 401 (invalid credentials).

#### GET /users
Get all non-deleted users.

- **Headers**: `Authorization: Bearer <jwt-token>`
- **Response (200 OK)**:
  ```json
  [
    {
      "id": "uuid",
      "email": "user@example.com"
    }
  ]
  ```
- **Errors**: 401, 403 (`read:user` required).

#### DELETE /users/:id
Soft delete a user.

- **Headers**: `Authorization: Bearer <jwt-token>`
- **Params**: `id` (user UUID)
- **Response (200 OK)**:
  ```json
  {
    "message": "User deleted"
  }
  ```
- **Errors**: 401, 403 (`delete:user` required), 404 (user not found).

#### POST /roles
Create a new role.

- **Headers**: `Authorization: Bearer <jwt-token>`
- **Request Body**:
  ```json
  {
    "name": "admin"
  }
  ```
- **Response (201 Created)**:
  ```json
  {
    "id": "uuid",
    "name": "admin"
  }
  ```
- **Errors**: 401, 403 (`create:role` required), 409 (role exists).

#### GET /roles
Get all non-deleted roles.

- **Headers**: `Authorization: Bearer <jwt-token>`
- **Response (200 OK)**:
  ```json
  [
    {
      "id": "uuid",
      "name": "admin"
    }
  ]
  ```
- **Errors**: 401, 403 (`read:role` required).

#### DELETE /roles/:id
Soft delete a role.

- **Headers**: `Authorization: Bearer <jwt-token>`
- **Params**: `id` (role UUID)
- **Response (200 OK)**:
  ```json
  {
    "message": "Role deleted"
  }
  ```
- **Errors**: 401, 403 (`delete:role` required), 404 (role not found).

#### POST /roles/assign
Assign a role to a user.

- **Headers**: `Authorization: Bearer <jwt-token>`
- **Request Body**:
  ```json
  {
    "userId": "user-uuid",
    "roleId": "role-uuid"
  }
  ```
- **Response (200 OK)**:
  ```json
  {
    "id": "uuid",
    "userId": "user-uuid",
    "roleId": "role-uuid"
  }
  ```
- **Errors**: 401, 403 (`assign:role` required), 404 (user/role not found).

#### POST /roles/remove
Remove a role from a user.

- **Headers**: `Authorization: Bearer <jwt-token>`
- **Request Body**:
  ```json
  {
    "userId": "user-uuid",
    "roleId": "role-uuid"
  }
  ```
- **Response (200 OK)**:
  ```json
  {
    "message": "Role removed"
  }
  ```
- **Errors**: 401, 403 (`assign:role` required), 404 (user/role not found).

#### POST /permissions
Create a new permission.

- **Headers**: `Authorization: Bearer <jwt-token>`
- **Request Body**:
  ```json
  {
    "action": "read:post"
  }
  ```
- **Response (201 Created)**:
  ```json
  {
    "id": "uuid",
    "action": "read:post"
  }
  ```
- **Errors**: 401, 403 (`create:permission` required), 409 (permission exists).

#### GET /permissions
Get all non-deleted permissions.

- **Headers**: `Authorization: Bearer <jwt-token>`
- **Response (200 OK)**:
  ```json
  [
    {
      "id": "uuid",
      "action": "read:post"
    }
  ]
  ```
- **Errors**: 401, 403 (`read:permission` required).

#### POST /permissions/assign
Assign a permission to a role.

- **Headers**: `Authorization: Bearer <jwt-token>`
- **Request Body**:
  ```json
  {
    "roleId": "role-uuid",
    "permissionId": "permission-uuid"
  }
  ```
- **Response (200 OK)**:
  ```json
  {
    "id": "uuid",
    "roleId": "role-uuid",
    "permissionId": "permission-uuid"
  }
  ```
- **Errors**: 401, 403 (`assign:permission` required), 404 (role/permission not found).

#### POST /permissions/remove
Remove a permission from a role.

- **Headers**: `Authorization: Bearer <jwt-token>`
- **Request Body**:
  ```json
  {
    "roleId": "role-uuid",
    "permissionId": "permission-uuid"
  }
  ```
- **Response (200 OK)**:
  ```json
  {
    "message": "Permission removed"
  }
  ```
- **Errors**: 401, 403 (`assign:permission` required), 404 (role/permission not found).

#### POST /access/check
Check if a user has a specific permission.

- **Headers**: `Authorization: Bearer <jwt-token>`
- **Request Body**:
  ```json
  {
    "userId": "user-uuid",
    "action": "read:post"
  }
  ```
- **Response (200 OK)**:
  ```json
  {
    "allowed": true
  }
  ```
- **Errors**: 401, 403 (`check:access` required), 404 (user not found).

## Development Workflow

1. **Setup**:
   - Initialize NestJS project: `nest new nest-rbac-auth`.
   - Install dependencies: `@prisma/client`, `prisma`, `@nestjs/jwt`, `@nestjs/passport`, `passport-jwt`, `bcrypt`.
   - Setup Prisma: `npx prisma init`.
   - Configure `.env` and `schema.prisma`.

2. **Implement Modules** (in order):
   - `prisma`: PrismaService for database access.
   - `auth`: Register, login, JWT strategy.
   - `users`: CRUD operations.
   - `roles`: CRUD, assign/remove roles.
   - `permissions`: CRUD, assign/remove permissions.
   - `access-control`: Permission checks, `/access/check`.
   - `common`: PermissionsGuard, JWT guard.

3. **Testing**:
   - Write e2e tests for each route.
   - Use Dockerized PostgreSQL in CI/CD.
   - Validate schema with SQLite locally.

4. **Documentation**:
   - Update this file as features are added.
   - Generate Swagger docs (future).

5. **Future**:
   - Test MySQL/SQLite.
   - Package as npm module.
   - Add CLI and advanced features.

## Notes
- Start with PostgreSQL to ensure stability.
- Keep queries database-agnostic (use Prisma’s standard methods).
- Test portability early with SQLite to avoid refactoring.
- Prioritize clean module exports for future npm packaging.
- Document each endpoint’s permission requirements clearly.

This document serves as the roadmap. Work through the routes and modules one by one, starting with `prisma` and `auth`, then users, roles, permissions, and access-control.