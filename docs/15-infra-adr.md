# 15 — Infrastructure Architecture Decision Records (ADR)

> Last updated: 2026-03-09

This document records key infrastructure decisions, their rationale, trade-offs considered, and the conditions under which each decision should be revisited.

---

## ADR-001: Defer NATS JetStream to Phase 4

**Status:** Accepted  
**Date:** 2026-03-09

### Context

The initial architecture included NATS JetStream as a domain event bus starting from Phase 0. However:

- The project uses a **modular monolith** with a **single worker process** — there is exactly one publisher and one consumer
- Redis + Asynq is already required for async job processing (PDF generation, bulk email, GDPR erasure, etc.)
- Running a production-grade NATS cluster requires 3 nodes for HA — significant ops overhead for a pre-revenue product

### Decision

Remove NATS JetStream from Phase 0-3. Use the **transactional outbox → Asynq (Redis)** pipeline for all domain event processing.

Flow:
```
DB transaction (mutation + domain_events row)
  → Outbox poller (500ms)
  → Asynq enqueue (Redis)
  → givernance-worker (at-least-once, retries, dead-letter)
```

### Rationale

| Option | Verdict |
|---|---|
| NATS JetStream from Phase 0 | ❌ One publisher + one consumer — 3-node cluster for zero architectural benefit |
| Postgres LISTEN/NOTIFY | ❌ No durability, no dead-letter, no retry semantics |
| Asynq (Redis) via outbox | ✅ At-least-once, retries, DLQ, job inspection, already required |
| Direct in-process call | ❌ Loses outbox durability guarantee (dual-write risk) |

Asynq provides natively: at-least-once delivery, configurable retry with backoff, dead-letter queues, job deduplication, cron scheduling, and a web UI for job inspection. No separate message broker needed.

The transactional outbox pattern intentionally abstracts the publish backend. Switching from Asynq-direct to NATS in Phase 4 requires changing **one file** (`pkg/events/publisher.go`) — zero domain logic changes.

### Consequences

- ✅ Phase 0 infrastructure: 8 services instead of 9 (simpler Docker Compose, faster local dev)
- ✅ No NATS operational knowledge required until Phase 4
- ✅ Webhook fan-out handled by Asynq retry mechanism (sufficient for <1,000 webhooks/org at Phase 0-3 scale)
- ⚠️ No event replay in Phase 0-3 — if needed, query `domain_events` table directly
- ⚠️ Multi-subscriber fan-out not available until Phase 4 — enforce single-consumer contract in outbox design

### Revisit when

- A second autonomous service is extracted from the monolith and needs domain events independently
- Outbound webhooks require multi-subscriber fan-out at scale (>1,000 active webhook endpoints)
- Event replay is needed for audit enrichment or system debugging
- Team size exceeds 4 engineers (shared Redis job queue coordination cost increases)

---

## ADR-002: Managed Infrastructure for SaaS Deployment

**Status:** Accepted  
**Date:** 2026-03-09

### Context

The initial architecture assumed fully self-hosted PostgreSQL + PgBouncer + Redis + MinIO for all deployment scenarios. For the **SaaS managed offering**, this creates unnecessary operational burden for a small founding team: database backups, failover configuration, Redis cluster management, and S3-compatible storage administration.

### Decision

For the **SaaS managed deployment**, use managed cloud services:

| Component | Managed Service | Rationale |
|---|---|---|
| PostgreSQL | [Neon.tech](https://neon.tech) EU region | Managed Postgres, built-in pooling, branching, WAL backup, EU data residency |
| Redis | [Upstash Redis](https://upstash.com) EU region | Serverless, pay-per-use, GDPR-compliant, no cluster to manage |
| Object Storage | [Cloudflare R2](https://developers.cloudflare.com/r2/) EU | S3-compatible, no egress fees, EU storage |

For **self-hosted NPO deployments** (Docker Compose), retain self-managed PostgreSQL 16 + PgBouncer + Redis 7 + MinIO. The S3 / Postgres compatible APIs ensure zero application code differences.

### Rationale

**Neon.tech vs alternatives:**
| Option | Verdict |
|---|---|
| Self-hosted Postgres (VPS) | ❌ Backup automation, failover, patching = ops overhead |
| Supabase PostgreSQL | ✅ Valid alternative — same Postgres, EU region. Rejected as *all-in-one* (see ADR-003), but usable as managed Postgres only |
| AWS RDS eu-west-3 | ⚠️ Valid but higher cost, more config surface |
| Neon.tech | ✅ Best DX, database branching for staging, built-in pooler, generous free tier, EU region |

**Upstash vs alternatives:**
| Option | Verdict |
|---|---|
| Self-hosted Redis Cluster (3 nodes) | ❌ Over-engineered for pre-scale; ops overhead |
| Elasticache (AWS) | ⚠️ Valid but high fixed cost, VPC dependency |
| Upstash | ✅ Serverless, pay-per-request, EU region, GDPR DPA available |

**Cloudflare R2 vs alternatives:**
| Option | Verdict |
|---|---|
| MinIO (self-hosted SaaS) | ❌ Storage admin + backup = ops overhead on SaaS |
| AWS S3 eu-west-3 | ✅ Valid, but egress fees add up (receipts, reports = read-heavy) |
| Cloudflare R2 | ✅ S3-compatible, **zero egress fees**, EU storage, DPA available |

### Consequences

- ✅ SaaS deployment simplified: 4 self-managed services (API, Worker, Web, Keycloak) instead of 8
- ✅ PostgreSQL feature parity: Neon supports all required extensions (uuid-ossp, pgcrypto, pg_trgm, ltree, pg_audit)
- ✅ GDPR data residency: all managed services offer EU-region storage with DPA
- ✅ WAL archiving, PITR, and read replicas available on Neon managed tier
- ⚠️ Neon free tier has cold-start latency (~500ms after inactivity) — use paid tier for production
- ⚠️ If monthly Neon cost exceeds ~€150/month, evaluate self-hosted Postgres on dedicated VPS

### Revisit when

- Monthly PostgreSQL cost on Neon exceeds the self-hosted equivalent (typically at €300+/month infra spend)
- A PostgreSQL extension is required that Neon does not support
- Data sovereignty requirement is stricter than EU-region managed (e.g., specific country jurisdiction required by an NPO's data protection authority)
- Upstash Redis latency is unacceptable for rate-limiting use case (measure p99 before switching)

---

## ADR-003: Reject Convex.dev and Supabase as All-in-One Backend Replacements

**Status:** Accepted  
**Date:** 2026-03-09

### Context

Evaluated Convex.dev and Supabase as potential all-in-one backend platforms to reduce infrastructure complexity for Phase 0.

### Decision

Reject both as primary backend replacements. Supabase PostgreSQL remains a valid managed database option (equivalent to Neon.tech — see ADR-002).

### Rationale — Convex.dev rejected

| Criterion | Assessment |
|---|---|
| Language support | ❌ TypeScript/JavaScript only — incompatible with Go backend |
| EU data residency | ❌ US-hosted; no standard EU-region option — incompatible with GDPR requirements for European NPO data |
| PostgreSQL features | ❌ Proprietary data model and query language — loses RLS (tenant isolation), pg_audit (7-year audit), pg_trgm (duplicate detection), ltree (org hierarchy) |
| Self-hosted option | ❌ No self-hosted path — incompatible with NPO on-premise deployment model |
| Vendor lock-in | ❌ Extreme — proprietary query language, reactive model, and function format |

### Rationale — Supabase all-in-one rejected

| Component | Assessment |
|---|---|
| Self-hosted Supabase | ❌ 12+ containers (Kong, GoTrue, PostgREST, Realtime, Storage, Studio, etc.) — more complex than the current stack |
| Supabase Auth (GoTrue) | ❌ Missing: SAML 2.0 bridge, MFA enforcement by role, magic-link for volunteers, brute-force protection by role — all specified in auth requirements |
| Supabase Realtime | ❌ Postgres LISTEN/NOTIFY: no durability, no dead-letter, no at-least-once semantics — insufficient to replace the transactional outbox |
| Supabase PostgreSQL only | ✅ Valid as managed Postgres (same as Neon.tech) — see ADR-002 |

### Consequences

- ✅ Keycloak retained — full auth feature set preserved (SAML 2.0, MFA, magic-link, brute-force protection)
- ✅ PostgreSQL with RLS, pg_audit, pg_trgm retained — GDPR tenant isolation and audit patterns preserved
- ✅ Self-hosted deployment path preserved — no cloud-only dependency
- ✅ Go backend retained — performance, single binary, operational simplicity
- ⚠️ Auth infrastructure requires self-hosting Keycloak — adds one container to manage in all deployment modes

---

*ADRs are append-only. To supersede a decision, add a new ADR referencing the one it replaces, and update the superseded ADR's status to "Superseded by ADR-XXX".*
