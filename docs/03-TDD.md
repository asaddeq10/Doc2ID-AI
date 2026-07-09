```markdown
# Technical Design Document (TDD)

**Project:** Doc2ID AI  
**Document Version:** 1.0  
**Status:** Pending Approval  
**Author:** Lead Technical Architect  

---

## 1. INTRODUCTION

### 1.1 Purpose
This Technical Design Document (TDD) outlines the detailed engineering implementation plan for Doc2ID AI. It bridges the conceptual design presented in the System Architecture Document (SAD) and Database Specification (DBS) with concrete code structures, patterns, and workflows. This document acts as the definitive technical blueprint for senior software engineers to begin implementation.

### 1.2 Scope
This document covers the frontend architecture, backend REST API architecture, asynchronous background worker implementation, external service integrations, security protocols, and testing strategies.

---

## 2. TECHNOLOGY STACK

*   **Frontend:** Next.js (App Router), React, TypeScript, Tailwind CSS, Zustand (State Management), React-Konva/Fabric.js (Canvas Engine).
*   **Backend API:** Node.js, Express.js, TypeScript.
*   **Database:** PostgreSQL 16+ (via Prisma ORM or standard connection pooling with Kysely/Slonik).
*   **Caching & Queueing:** Redis, BullMQ.
*   **Background Workers:** Node.js, BullMQ Workers, Puppeteer (PDF Rendering / HTML-to-Image rendering).
*   **Storage:** AWS S3 (Object Storage).
*   **Validation:** Zod (Request payload and JSONB schema validation).

---

## 3. ARCHITECTURE DESIGN PATTERNS

### 3.1 Backend Architecture
The backend strictly adheres to a **Layered (N-Tier) Architecture** to ensure separation of concerns and testability.
1.  **Controller Layer:** Handles HTTP request/response, extracts parameters, delegates to services.
2.  **Service Layer:** Contains core business logic, orchestrates data flow, applies tenant isolation.
3.  **Repository Layer:** Abstracts direct database interactions.

### 3.2 Backend Folder Structure
```text
backend/
 ├── src/
 │    ├── config/          # Environment variables and config loaders
 │    ├── controllers/     # Route controllers
 │    ├── middlewares/     # Auth, Tenant scoping, Error handlers
 │    ├── models/          # ORM schemas / DB types
 │    ├── repositories/    # Database access logic
 │    ├── routes/          # Express route definitions
 │    ├── services/        # Business logic
 │    ├── utils/           # Helper functions (S3 URL generation, Hashers)
 │    ├── workers/         # BullMQ worker processors
 │    └── app.ts           # Express setup
 ├── package.json
 └── tsconfig.json
```

### 3.3 Frontend Architecture
The frontend uses **Next.js App Router** for Server-Side Rendering (SSR) of public pages and Client-Side Rendering (CSR) for the highly interactive dashboard and canvas editor.
*   **Component-Driven:** UI is built with atomic components (Atoms -> Molecules -> Organisms).
*   **State Management:** Local state via `useState`/`useReducer`. Global state (e.g., active user, active project, canvas elements) via `Zustand`.

### 3.4 Frontend Folder Structure
```text
frontend/
 ├── src/
 │    ├── app/             # Next.js App Router (pages & layouts)
 │    ├── components/      # Reusable UI components
 │    │    ├── ui/         # Base atoms (Buttons, Inputs)
 │    │    └── features/   # Complex blocks (TemplateEditor, DataGrid)
 │    ├── hooks/           # Custom React hooks
 │    ├── lib/             # Utility functions, API clients (Axios/Fetch)
 │    ├── store/           # Zustand global state slices
 │    └── types/           # TypeScript interfaces
 ├── tailwind.config.js
 └── package.json
```

---

## 4. CORE IMPLEMENTATION FLOWS

### 4.1 Authentication and Authorization Flow
*   **Strategy:** Stateless JWT Bearer tokens.
*   **Flow:**
    1. User submits credentials to `/auth/login`.
    2. Backend validates against `users.password_hash` (Argon2).
    3. Backend generates a JWT containing `{ user_id, email, organization_id, role }`.
    4. Frontend stores JWT in a secure `HttpOnly` cookie or memory (depending on exact security posture; `HttpOnly` cookie preferred to prevent XSS).
*   **Middleware:** An Express middleware `requireAuth` verifies the JWT. A subsequent `requireRole(role)` middleware checks RBAC permissions. All valid requests attach `req.user` for tenant scoping.

### 4.2 File Upload Pipeline (Direct to S3)
To prevent Node.js memory exhaustion from 500MB ZIP files, we utilize the Direct-to-S3 Pre-signed URL pattern.

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant S3
    participant WorkerQueue
    Client->>API: POST /uploads/presigned-url (file metadata)
    API->>S3: Request Pre-signed PUT URL
    S3-->>API: Return URL
    API-->>Client: Return { upload_id, url }
    Client->>S3: PUT /bucket/key (Binary Data)
    S3-->>Client: 200 OK
    Client->>API: POST /uploads/{id}/complete
    API->>WorkerQueue: Enqueue 'extract-zip' / 'parse-csv' Job
    API-->>Client: 202 Accepted
```

### 4.3 Template Rendering Engine
*   **Canvas Storage:** The frontend visual editor generates a JSON payload representing the layout (`template_elements.properties`).
*   **Rendering Process:**
    1. The Worker receives the JSON payload and the target data row (e.g., `First_Name: John`).
    2. The Worker injects the data into the JSON schema.
    3. The Worker renders an HTML/CSS string representing the card.
    4. A headless browser (Puppeteer) captures the HTML as a high-resolution PNG or PDF buffer.
    5. The buffer is streamed directly to S3.

### 4.4 Batch Processing & Queue Workflow
Handling 10,000 to 100,000 cards per batch requires a Master/Worker fan-out pattern using BullMQ.

```mermaid
sequenceDiagram
    participant API
    participant Redis(BullMQ)
    participant MasterWorker
    participant RenderWorkers
    participant Database

    API->>Redis(BullMQ): Enqueue MasterJob (ProjectID)
    Redis(BullMQ)->>MasterWorker: Process MasterJob
    MasterWorker->>Database: Fetch all Import_Records
    loop Every Record
        MasterWorker->>Redis(BullMQ): Enqueue MicroJob (RenderCard)
    end
    Redis(BullMQ)->>RenderWorkers: Distribute MicroJobs (Concurrent)
    RenderWorkers->>Database: Update Generation_Job_Item (Success/Fail)
    RenderWorkers->>Database: Increment Job Progress
```

---

## 5. API INTEGRATION FLOW
*   **Client Setup:** The frontend utilizes Axios with a centralized interceptor.
*   **Interceptors:** Automatically append the JWT to headers. If a `401 Unauthorized` is returned, the interceptor redirects to `/login`.
*   **Tenant Headers:** For users belonging to multiple organizations, the frontend sets a custom header `X-Organization-ID` which the backend uses to scope the `organization_id` context.

---

## 6. ERROR HANDLING STRATEGY
*   **Backend:** 
    *   Custom `AppError` class extending `Error` with HTTP status codes and operational flags.
    *   Centralized Express error middleware catches all thrown errors, preventing stack traces from leaking to the client in production.
    *   Database errors (e.g., Unique Constraint Violations) are caught in the Repository layer and translated into user-friendly `AppError` instances (e.g., "Email already exists").
*   **Frontend:**
    *   Global Error Boundary in React to catch render failures.
    *   Toast notification system for displaying API errors gracefully.

---

## 7. LOGGING AND MONITORING
*   **Logging Library:** `Pino` (JSON-formatted, high-performance logger) for Node.js.
*   **Log Levels:** `debug` (Dev), `info` (Prod access logs), `warn` (Recoverable errors), `error` (Fatal exceptions, stack traces).
*   **Audit Logging:** Mutating requests (`POST`, `PUT`, `DELETE`) trigger a non-blocking background write to the `audit_logs` database table.
*   **APM:** Implementation of Datadog or Sentry. Express middleware intercepts `500` errors and pushes them to Sentry with context (`user_id`, `organization_id`).

---

## 8. SECURITY IMPLEMENTATION
*   **Data at Rest:** Database encrypted via AWS KMS.
*   **Row-Level Security (RLS):** Enabled in PostgreSQL. Node.js backend executes `SET LOCAL app.current_org_id = 'uuid'` before executing queries in the transaction.
*   **Input Validation:** `Zod` middleware validates all incoming `req.body` and `req.query`.
*   **Rate Limiting:** Redis-backed rate limiter on `auth/*` routes to prevent brute-force attacks.
*   **CORS:** Strictly configured to allow only the production and staging frontend domains.

---

## 9. CONFIGURATION & ENVIRONMENT VARIABLES
Environment variables are validated on app startup using Zod to prevent runtime crashes due to missing config.

```env
# Backend .env example
NODE_ENV=production
PORT=4000
DATABASE_URL=postgresql://user:pass@host:5432/doc2id
REDIS_URL=redis://host:6379
JWT_SECRET=super_secure_random_string
JWT_EXPIRES_IN=1d
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_S3_BUCKET=doc2id-prod-assets
AWS_REGION=us-east-1
```

---

## 10. CACHING STRATEGY
*   **Redis Usage:**
    *   Session tracking / JWT invalidation (Blacklist approach for logouts).
    *   Idempotency Keys: Caching request hashes for 5 seconds on `/generate` endpoints to prevent double-clicks triggering dual massive batch jobs.
    *   *Not* used for primary data entities to avoid cache-invalidation complexity in the MVP phase.

---

## 11. TESTING STRATEGY
*   **Unit Testing (Backend):** `Jest`. Testing Service layer business logic and utility functions. Mocks applied to Repositories.
*   **Integration Testing (Backend):** `Supertest` alongside a dedicated test PostgreSQL instance. Validates Controller -> Service -> Repository flow.
*   **Frontend Testing:** `React Testing Library` for component unit tests; `Cypress` or `Playwright` for E2E critical paths (Login, Upload, Drag-and-drop, Generate).

---

## 12. PERFORMANCE OPTIMIZATION & SCALABILITY
*   **Horizontal Scaling:** Both the API nodes and Worker nodes are stateless. They can scale from 1 to N instances behind a Load Balancer.
*   **Database Pooling:** `PgBouncer` is required. Worker nodes open connections rapidly during micro-job execution. PgBouncer multiplexes these connections to prevent PostgreSQL connection exhaustion.
*   **Pagination:** Limit/Offset pagination enforced heavily on `import_records` and `downloads` to prevent memory bloat on large organizational datasets.

---

## 13. DEPLOYMENT CONSIDERATIONS
*   **Containerization:** `Dockerfile` definitions for both API and Worker nodes.
*   **CI/CD:** GitHub Actions.
    *   *On PR:* Run Linting, TypeScript compilation, and Jest tests.
    *   *On Merge to Main:* Build Docker images, push to Amazon ECR, deploy to Amazon ECS (Fargate).
*   **Infrastructure as Code (IaC):** Terraform or AWS CDK recommended for provisioning RDS, ElastiCache, S3, and ECS clusters to ensure environment parity (Staging vs. Prod).

---
**End of TDD**
```
