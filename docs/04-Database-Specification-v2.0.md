Database Specification Document (DBS)

Project: Doc2ID AI
Document Version: 2.0 (Enterprise Specification)
Status: Approved
Author: Chief Database Architect
Target RDBMS: PostgreSQL 16+


---

1. DOCUMENT INFORMATION

Title: Doc2ID AI Enterprise Database Specification

Purpose: Definitive schema and architecture guide for the Doc2ID AI SaaS platform.

Audience: Backend Engineers, DBAs, DevOps, Data Engineers.


2. PURPOSE

This document defines the physical and logical database architecture for Doc2ID AI. It guarantees that the data layer will support strict multi-tenant isolation, massive asynchronous batch processing (100,000+ records per job), highly dynamic user schemas, and robust auditability suitable for SOC 2 compliance.

3. DATABASE ARCHITECTURE OVERVIEW

The database is structured around a centralized Relational Database Management System (RDBMS). It enforces a strict relational model for core business logic (Users, Organizations, Projects, Auth) while intelligently leveraging NoSQL patterns (JSONB) for dynamic, user-defined data structures (Imported spreadsheet rows, Template configurations).

4. DATABASE TECHNOLOGY SELECTION

Primary Database: PostgreSQL (Version 16 or newer).

Why PostgreSQL?

Unmatched support for JSONB operations (GIN indexing).

Native Row-Level Security (RLS) for multi-tenant isolation.

Advanced concurrent indexing and partitioning for scale.

ACID compliance and unparalleled data integrity constraints.


Connection Pooling: PgBouncer (Transaction mode) to handle thousands of concurrent background worker connections.


5. ENTITY RELATIONSHIP OVERVIEW

The schema is organized into four logical domains:

1. Identity & Access Management (IAM): organizations, users, organization_members


2. Core Workspaces: projects, templates, template_elements


3. Ingestion & Data Mapping: uploaded_files, uploaded_photos, import_jobs, import_records


4. Processing & Audit: generation_jobs, generation_job_items, downloads, audit_logs, notifications



6. CORE DATABASE PRINCIPLES

Primary Keys: UUIDv7 is mandated across all tables for time-ordered, distributed generation without B-Tree index fragmentation.

Naming Conventions: snake_case for all identifiers (tables, columns, indexes). Tables are strictly pluralized (e.g., users).

Timestamps: All dates are stored as TIMESTAMP WITH TIME ZONE (UTC).

Foreign Keys: Strict referential integrity. Action rules (CASCADE vs RESTRICT) are explicitly defined to prevent orphaned data while protecting against accidental deletion.

Soft Deletes: Standardized deleted_at columns on all primary entities.



---

7. COMPLETE DATABASE SCHEMA & 8. DETAILED TABLE SPECIFICATIONS

8.1 organizations

Purpose: The root boundary for multi-tenancy. Every piece of operational data belongs to an organization.

Primary Key: id (UUID)


Column	Data Type	Null	Default	Keys/Constraints	Description

id	UUID	No	uuid_generate_v7()	PK	Unique org identifier.
name	VARCHAR(255)	No	-	-	Name of the institution/company.
slug	VARCHAR(100)	No	-	UNIQUE	URL-friendly identifier.
billing_tier	VARCHAR(50)	No	'FREE'	CHECK	e.g., FREE, PRO, ENTERPRISE.
settings	JSONB	No	'{}'	-	Configs (e.g., default timezone).
created_at	TIMESTAMPTZ	No	NOW()	-	Creation time.
updated_at	TIMESTAMPTZ	No	NOW()	-	Last modification time.
deleted_at	TIMESTAMPTZ	Yes	NULL	-	Soft delete flag.


Example Record:

{ "id": "01h45g...", "name": "Springfield High", "slug": "springfield-high", "billing_tier": "PRO" }


8.2 users

Purpose: Individual human identities interacting with the platform.

Primary Key: id (UUID)


Column	Data Type	Null	Default	Keys/Constraints	Description

id					
