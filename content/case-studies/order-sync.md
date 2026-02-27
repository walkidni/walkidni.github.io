---
title: "Case Study: Designing a Multi-Tenant, Dynamically Scheduled Sync Engine"
date: 2026-02-27
draft: false
tags:
  - "Systems Design"
  - "Event-Driven"
  - "Scheduling"
  - "FastAPI"
  - "Celery"
---

## 1. Problem Context

E-commerce teams often rely on spreadsheets as their operational source of truth. Meanwhile, e-commerce platforms expose order data through APIs and webhooks.

Manual exports introduce:

- Data drift  
- Operational overhead  
- No retry semantics  
- No consistency guarantees  

The objective was not simply to export orders to Google Sheets.

It was to design a **fault-tolerant, shared-database multi-tenant synchronization engine with strong tenant isolation** that:

- Keeps Google Sheets in sync with order data  
- Supports user-defined export configurations  
- Executes reliably via manual triggers, webhooks, or dynamic schedules  
- Scales safely across tenants  
- Remains idempotent under retries and webhook replay  

---

## 2. High-Level Architecture

The system consists of three runtime roles:

### API Server (FastAPI)

- Authenticates tenant installations  
- Manages export automation configuration  
- Validates constraints  
- Creates sync runs  
- Enqueues background jobs  
- Optionally runs a scheduler loop  

### Worker (Celery + Redis)

- Executes long-running sync runs  
- Streams data from upstream APIs  
- Writes to Google Sheets  
- Updates run state  

### MongoDB

- Shared-database multi-tenant persistence  
- Tenant-scoped partitioning and query isolation  
- Automation definitions  
- Run history  
- Scheduler locking  
- Idempotency keys  

This architecture supports:

- Clear separation of concerns  
- Safe background execution  
- Durable state  
- Horizontal scalability  

---

## 3. Multi-Tenant Isolation Model

All data is scoped by a `tenant_id`.

Each installation:

- Exchanges an auth code  
- Receives a locally minted bearer token  
- Has all data partitioned by tenant  

Tokens are:

- Stored hashed (SHA-256)  
- Revocable  
- TTL-controlled  

Design principle:

> Shared database, strict tenant-level isolation. No intentional cross-tenant queries.

This enables safe horizontal scaling and reduces the risk of accidental data leakage from application-level mistakes.

---

## 4. Google OAuth Linking

Each tenant can link one or more Google accounts.

Security measures:

- Server-stored OAuth state nonce  
- One-time state consumption  
- Expiring state records  
- Linked accounts scoped per tenant  
- Exactly one default account per tenant (enforced via partial unique index)  

This helps ensure:

- Safe OAuth flow  
- No replay attacks  
- Multi-account flexibility  

---

## 5. Automation as a First-Class Entity

The core abstraction is the **Automation**.

An automation defines:

- What to export  
- Optional scope filters  
- Destination spreadsheet and worksheet  
- Dynamic column schema  
- Optional scheduling configuration (cron)  

Constraints applied:

- No two automations can target the same spreadsheet tab  
- Cron must respect minimum execution interval  
- Only active automations can execute  

Exports become durable, declarative configuration — not ad-hoc jobs.

---

## 6. Dynamic Scheduling Design (Why Not Celery Beat?)

Scheduling is **per-automation and user-defined**.

Each automation has:

- Its own cron expression  
- Enable/disable state  
- Next execution timestamp  
- Locking lifecycle  

Schedules are dynamic and mutable at runtime.

Celery Beat is designed for centrally defined periodic tasks registered at startup. It is not a natural fit for:

- Thousands of user-configurable schedules  
- Runtime schedule mutations  
- Tenant-scoped periodic jobs  

Instead, the system uses a **data-driven scheduler**:

- Eligible automations are selected by `next_run_at <= now()`  
- An atomic update sets `scheduler_locked_until` to claim execution  
- The scheduler enqueues a run and computes the next execution time  
- The next run timestamp is persisted  

This enables:

- Safe horizontal scaling  
- Reduced risk of duplicate scheduling  
- No centralized scheduler coordination  
- Fully dynamic per-tenant scheduling  

The scheduler is stateful and data-driven, not static and process-bound.

---

## 7. Sync Runs as Durable Units of Work

Each execution is represented by a `sync_run` document.

Every run:

- Has a lifecycle (pending → running → completed/failed)  
- Stores metadata  
- Stores error summaries  
- Is linked to an automation  
- Can be retried with idempotency safeguards  

Triggers:

- Manual  
- Webhook-driven  
- Periodic scheduler  

Persisting runs provides:

- Observability  
- Debuggability  
- Auditability  
- Retry safety  

---

## 8. Webhook Ingestion & Idempotency

Webhooks are untrusted and at-least-once by nature.

The system enforces:

- HMAC-SHA256 signature verification  
- Tenant resolution via headers  
- Event idempotency key stored in Mongo  
- TTL window for deduplication  

This is intended to provide:

- Replay protection  
- Reduced risk of duplicate sync runs  
- Robustness under retry storms  

The system is explicitly designed for at-least-once delivery.

---

## 9. Streaming Export Pipeline

The main technical challenge was preventing memory blowup during large exports.

Naive approach:

- Materialize full dataset  
- Transform  
- Write  

This scales linearly with export size.

Final design:

- Stream rows incrementally  
- Transform per-row  
- Group consecutive rows by unique order identifier  
- Flush bounded batches to Sheets  

Explicit batch caps:

- 300 order groups  
- 1200 rows  
- 1.2MB payload  

Effects:

- Memory remains bounded  
- External API payload size controlled  
- Large exports handled safely  

The system scales predictably.

---

## 10. Idempotent Sheets Writes

Google Sheets writes use upsert semantics keyed by a unique order identifier.

Write strategy:

- Ensure worksheet exists  
- Rewrite header if schema changes  
- Read existing row identities  
- Update existing rows  
- Append new rows  

This helps ensure:

- Safe retries  
- No logical duplication  
- Consistency under worker restarts (assuming idempotency keys remain stable)  

Design goal:

* At-least-once execution must not corrupt downstream state.

---

## 11. Async ↔ Worker Boundary Stability

FastAPI services are async.  
Celery tasks are synchronous entrypoints.

To prevent event loop instability in prefork workers:

- Each task creates a fresh async database client  
- `asyncio.run()` is called inside the task  
- No event loop reuse across executions  

This avoids:

- `RuntimeError: Event loop is closed`  
- Loop contamination across tasks  

A subtle but important production stability detail.

---

## 12. Tradeoffs & Engineering Decisions

**Why streaming instead of full materialization?**  
Memory safety and scalability.

**Why DB-backed scheduler instead of Celery Beat?**  
Dynamic per-tenant scheduling and atomic claim safety.

**Why upsert instead of append-only?**  
Retry safety and webhook idempotency.

**Why tenant-scoped auth?**  
Hard isolation boundaries and simpler access control.

**Why persist sync runs?**  
Operational debugging and audit trail.

---

## 13. Reliability & Observability

Each run records:

- Start and end timestamps  
- Status  
- Errors  
- Metadata  

This enables:

- Operational inspection  
- Debugging  
- Manual re-execution  
- Failure transparency  
