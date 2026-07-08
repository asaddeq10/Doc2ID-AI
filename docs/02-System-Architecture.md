# System Architecture Document (SAD)

**Project:** Doc2ID AI  
**Document Version:** 1.0  
**Status:** Approved  
**Date:** July 8, 2026  
**Author:** Senior Solutions Architect  

---

## 1. INTRODUCTION

### 1.1 Purpose
This System Architecture Document (SAD) provides a comprehensive overview of the technical architecture for the Doc2ID AI SaaS platform. It serves as the definitive guide for engineering teams, DevOps, and stakeholders to understand the system's structural design, component interactions, scalability strategies, and deployment model. 

### 1.2 Scope
This document covers the high-level architecture, subsystem boundaries, data flow pipelines, asynchronous processing infrastructure, storage solutions, and security architecture required to implement the MVP and scale to enterprise volumes (100,000+ cards per batch).

---

## 2. ARCHITECTURAL OVERVIEW

### 2.1 High-Level Architecture Pattern
Doc2ID AI utilizes a **Cloud-Native, Event-Driven Microservices-Oriented Architecture** pattern, structured around a clear separation between a stateless presentation tier, an API-driven business logic tier, and a highly decoupled asynchronous processing tier. 

*   **Presentation Tier (Frontend):** A Server-Side Rendered (SSR) and Client-Side React application delivering a responsive, single-page application (SPA) experience.
*   **Business Logic Tier (API Gateway/Core Server):** A stateless RESTful API that handles HTTP requests, authentication, tenant isolation, and synchronous CRUD operations.
*   **Asynchronous Processing Tier (Workers):** A horizontally scalable fleet of background workers connected via a high-throughput message queue, responsible exclusively for CPU-intensive rendering (PDF/PNG generation) and file processing (ZIP extraction).
*   **Data & Storage Tier:** A managed relational database for structured state and multi-tenant configurations, paired with highly durable object storage for binary assets.

---

## 3. COMPONENT ARCHITECTURE

### 3.1 Frontend Client (Next.js / React)
*   **Framework:** Next.js (App Router) for hybrid static & server rendering, combined with React context and state management.
*   **Styling:** Tailwind CSS for rapid, design-system-compliant UI development.
*   **Responsibilities:**
    *   Routing and View presentation.
    *   Client-side validation of file uploads (size/mime-type checks).
    *   Drag-and-drop template canvas rendering (via HTML5 Canvas or SVG abstraction).
    *   WebSocket/Polling management for real-time job progress updates.

### 3.2 Core API Server (Node.js / Express)
*   **Framework:** Express.js running on Node.js (TypeScript).
*   **Responsibilities:**
    *   **Authentication & Authorization:** Validating JWTs, checking Role-Based Access Control (RBAC) via middleware.
    *   **Tenant Routing:** Ensuring every query enforces `organization_id` boundaries.
    *   **Orchestration:** Receiving requests to initiate batch jobs and dispatching payloads to the message queue.
    *   **Storage Gateway:** Generating secure, pre-signed S3 URLs for client-side direct uploads and downloads to minimize API bandwidth saturation.

### 3.3 Background Worker Fleet (Node.js / BullMQ)
*   **Framework:** Node.js workers leveraging BullMQ (backed by Redis) for robust job scheduling, retries, and rate limiting.
*   **Worker Types:**
    1.  **Ingestion Workers:** Handle extracting massive ZIP files, calculating file hashes, and saving objects to S3.
    2.  **Generation Workers:** Headless browser instances (e.g., Puppeteer/Playwright) or native PDF libraries (e.g., PDFKit/ReportLab) that map JSON data into physical print layouts, render the images/PDFs, and stream the outputs back to S3.
*   **Characteristics:** Completely stateless. Can be scaled from 1 instance to 100+ instances based on the active queue length.

### 3.4 Data & Caching Layer
*   **Primary Relational Database:** PostgreSQL (Managed, e.g., AWS RDS or GCP Cloud SQL). Holds core structured data, JSONB templates, multi-tenant maps, and audit logs.
*   **In-Memory Cache & Message Broker:** Redis (Managed, e.g., AWS ElastiCache). Used for session caching, rate-limiting counters, and as the backing store for BullMQ job queues.
*   **Object Storage:** AWS S3 (or equivalent). Stores raw uploaded ZIPs, CSVs, extracted profile photos, and finalized generated PDFs/PNGs.

---

## 4. SYSTEM DIAGRAM (C4 Level 2 - Container Diagram)

```text
                               ┌─────────────────────────┐
                               │                         │
                               │  Web Browser (Client)   │
                               │  (Next.js React App)    │
                               │                         │
                               └───────┬─────────▲───────┘
                                       │         │
                 1. HTTPS REST / WebSockets      │ 2. Direct S3 Pre-signed
                                       │         │    Upload/Download
                                       ▼         │
┌────────────────────────────────────────────────────────────┐
│                       Load Balancer                        │
└──────┬───────────────────────────────────────────────┬─────┘
       │                                               │
       ▼                                               │
┌─────────────────────────┐                            │
│                         │                            │
│     Core API Server     │                            │
│    (Node.js/Express)    │                            │
│                         │                            │
└──────┬────────────┬─────┘                            │
       │            │                                  │
       │ 3. Enqueue │ 4. CRUD                          │
       │    Jobs    │    Data                          │
       ▼            ▼                                  ▼
┌─────────────┐  ┌───────────────────┐       ┌───────────────────┐
│             │  │                   │       │                   │
│ Redis Cache │  │    PostgreSQL     │       │ AWS S3 / Blob     │
│  (BullMQ)   │  │ (Primary State)   │       │ (Object Storage)  │
│             │  │                   │       │                   │
└──────┬──────┘  └───────────────────┘       └───────────────────┘
       │                                               ▲
       │ 5. Dequeue                                    │
       │    & Process                                  │
       ▼                                               │
┌──────────────────────────────────────────────────┐   │
│                                                  │   │
│              Background Worker Fleet             │   │
│  (Ingestion Workers & PDF Generation Workers)    ├───┘ 6. Read/Write 
│                                                  │        Assets
└──────────────────────────────────────────────────┘

## 5. CORE DATA FLOWS

### 5.1 Direct-to-S3 File Upload Flow (Bypassing API bottlenecks)
To handle 500MB ZIP files without crashing the Core API node:
1.  **Client** requests a secure upload token: `GET /api/uploads/presigned-url?filename=photos.zip`.
2.  **Core API** verifies permissions, records an `Upload` entry as `PENDING` in PostgreSQL, and requests a pre-signed URL from S3.
3.  **Core API** returns the URL to the **Client**.
4.  **Client** performs a `PUT` request directly to **S3** with the binary data.
5.  **Client** notifies the API: `POST /api/uploads/{id}/complete`.
6.  **Core API** enqueues an `extract-zip` job in **Redis**.

### 5.2 Asynchronous Batch Generation Pipeline
1.  **Client** finalizes the template and clicks "Generate". Sends `POST /api/projects/{id}/generate`.
2.  **Core API** validates the project state, creates a `Generation_Job` in PostgreSQL, and enqueues a `master-generation-job` in **Redis**.
3.  **Core API** immediately returns `202 Accepted` to the client.
4.  **Worker Fleet** picks up the master job. It paginates through the `Import_Records` in PostgreSQL.
5.  For every card needed, it creates a `Generation_Job_Item` in PostgreSQL and enqueues micro-jobs (`render-card`).
6.  Multiple **Workers** consume `render-card` jobs concurrently. They download the required photo from **S3**, apply the JSON template, render the final asset, and upload it back to **S3**.
7.  As each card finishes, the worker increments the `processed_items` counter.
8.  Once all micro-jobs complete, a final `package-batch` job runs, zipping the PNGs or merging the PDFs into a single file.
9.  **Worker** updates the PostgreSQL `Generation_Job` to `COMPLETED` and fires a WebSocket/Email notification event.

---

## 6. SCALABILITY & HIGH AVAILABILITY

### 6.1 Stateless Compute
All application servers (Next.js frontend host, Express Core API, Worker nodes) are 100% stateless. They store no local files and hold no local session state. This allows them to be deployed in an Auto-Scaling Group (ASG) or Kubernetes Cluster, scaling up and down purely based on CPU utilization and Redis queue length.

### 6.2 Database Scaling
*   **Connection Pooling:** PgBouncer is deployed between the Core API/Workers and PostgreSQL to prevent connection exhaustion during massive batch jobs.
*   **Read Replicas:** Read-heavy dashboard queries and data-grid views can be routed to a PostgreSQL Read Replica, keeping the Primary node dedicated to heavy transactional writes during generation.

### 6.3 Queuing Strategy
BullMQ relies on Redis. To ensure jobs are not lost during instance failure, BullMQ utilizes a "two-phase commit" pattern via Redis lists. If a worker node crashes mid-render, the job is not acknowledged and is automatically moved back to the active queue after a timeout to be processed by a healthy worker.

---

## 7. SECURITY & COMPLIANCE ARCHITECTURE

### 7.1 Multi-Tenant Data Isolation
*   **Application Level:** All Core API middleware injects the user's `organization_id` (extracted from the verified JWT) into the request context. All database queries are strictly scoped: `WHERE organization_id = req.ctx.orgId`.
*   **Database Level (Defense in Depth):** Future implementation of PostgreSQL Row-Level Security (RLS) policies acting as a fail-safe against application code bugs.

### 7.2 Storage Security
*   **S3 Buckets:** Buckets are set to strictly Private. All files are encrypted at rest using AES-256. 
*   **Access Pattern:** Users cannot access files via public URLs. The Core API generates temporary, pre-signed URLs (expiring in 15 minutes) only after verifying the user holds the correct Role/Permissions for that specific file.
*   **Lifecycle Rules:** To comply with data minimization and GDPR, an S3 object lifecycle rule automatically permanently deletes all temporary rendering artifacts and original uploaded ZIPs 30 days after creation.

### 7.3 Network & Transport
*   **WAF & DDoS:** A Web Application Firewall (WAF) sits in front of the Load Balancer, protecting against SQL injection, cross-site scripting (XSS), and rate-limiting abusive IPs.
*   **TLS/SSL:** All data in transit between the client, load balancer, API, and internal services is encrypted using TLS 1.3.

---

## 8. INFRASTRUCTURE & DEPLOYMENT (AWS Target Model)

*   **DNS & CDN:** Route 53 / Amazon CloudFront (Caching frontend assets).
*   **Compute:** Amazon Elastic Container Service (ECS) with AWS Fargate. Docker containers for the API and Workers.
*   **Database:** Amazon Aurora PostgreSQL (Multi-AZ deployment for high availability).
*   **Cache/Queue:** Amazon ElastiCache for Redis.
*   **Storage:** Amazon S3 (Standard for active files, transitioning to Intelligent-Tiering if needed).
*   **CI/CD Pipeline:** GitHub Actions. Triggers automated tests on PR. On merge to `main`, builds Docker images, pushes to Amazon ECR, and triggers a rolling deployment to ECS with zero downtime.

---

## 9. THIRD-PARTY INTEGRATIONS
*   **Transactional Email:** SendGrid or AWS SES for password resets, invitations, and job completion notifications.
*   **Monitoring & Observability:** Datadog or AWS CloudWatch for log aggregation, APM (Application Performance Monitoring), and alerting on high worker-error rates.
*   **Future Integrations:**
    *   *Stripe:* For SaaS subscription billing.
    *   *AWS Textract / Google Cloud Vision:* For phase-2 OCR data extraction capabilities.

---
**End of Document**
```
