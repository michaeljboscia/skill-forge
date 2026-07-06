# Skills-Driven Development: Strategic Gap Analysis

**Date:** 2026-04-06
**Question:** What are the hardest, most valuable things humans do in software — and how do skills packages make them accessible?

---

## The Thesis

A skill package isn't a tutorial. It's a **compressed decision tree** that took an expert 10 years to build, encoded as anti-rationalization rules so an AI can't take the shortcuts that look correct but fail in production.

The most valuable skill packages target problems where:
- Humans alone: months of expertise, expensive mistakes, invisible failure modes
- With skills: the expert's judgment is embedded in the AI's behavior, making the hard thing *routine*

---

## Tier 1: Problems That Break Companies

### 1. Zero-Downtime Database Migrations at Scale
**The human problem:** One wrong `ALTER TABLE` on a billion-row table locks production for hours. Requires deep knowledge of lock behavior per engine, online DDL tools (pt-osc, gh-ost, pgroll), backfill strategies, dual-write patterns. Most teams learn this by causing an outage.

**Skills gap:** `mx-postgres-*` (planned) covers single-db patterns. Missing: **mx-migrations-**** — multi-database migration orchestration, blue-green schema changes, data backfill pipelines, rollback strategies.

**AI leverage:** Skill encodes "never add a NOT NULL column without a default on a table > 1M rows" as a hard rule. The AI physically cannot generate the dangerous migration.

---

### 2. Distributed Systems Correctness
**The human problem:** Consensus, eventual consistency, exactly-once semantics, split-brain recovery. Even senior engineers get this wrong. The bugs are non-deterministic, unreproducible in dev, and catastrophic in prod. CAP theorem tradeoffs are context-dependent.

**Skills gap:** Nothing in shipped or pipeline covers distributed patterns directly. Pieces exist in AWS (SQS, Kinesis) and Go (concurrency) but no unified **mx-distributed-*** package.

**AI leverage:** Skill prevents the AI from generating code that assumes network reliability, forces idempotency keys on every write, requires timeout+retry+circuit-breaker on every external call. The "happy path only" rationalization is blocked.

---

### 3. Security Architecture (Not Just Auth)
**The human problem:** Threat modeling, supply chain security, zero-trust networking, secret rotation, mTLS, PKI, RBAC/ABAC design. Most devs skip it because it's invisible until breach. The cost of getting it wrong is existential.

**Skills gap:** AWS Security (1 skill) and GCP Security (1 skill) exist but are cloud-specific. No cross-cutting **mx-security-*** package for: threat modeling, dependency auditing, secret management patterns, OWASP prevention, CSP headers, CORS, CSRF.

**AI leverage:** Skills make security *structural* — the AI can't generate an endpoint without input validation, can't create a Docker image running as root, can't store secrets in env vars without encryption-at-rest.

---

### 4. Legacy System Modernization
**The human problem:** Taking a 15-year-old monolith and incrementally modernizing it without stopping business. Strangler fig pattern, anti-corruption layers, event sourcing retrofit, dual-write with reconciliation. The hardest part isn't code — it's understanding what the old code *actually does* vs. what docs say it does.

**Skills gap:** Nothing. This is arguably the most valuable problem in enterprise software and we have zero coverage. **mx-modernization-*** — strangler fig, domain boundary identification, incremental extraction, feature flags for migration, contract testing between old and new.

**AI leverage:** The AI reads the legacy codebase, maps actual behavior (not documented behavior), generates the anti-corruption layer, and ensures the new system produces identical outputs via contract tests. Humans alone take 6-18 months to map a monolith. AI + skills: weeks.

---

## Tier 2: Problems That Cost Millions in Engineering Hours

### 5. Full-System Performance Optimization
**The human problem:** Not micro-benchmarks. Finding the bottleneck across network → compute → storage → memory across 20 services. Requires profiling across layers, understanding hardware, cache hierarchies, connection pooling, query plan analysis. Most teams throw hardware at it.

**Skills gap:** Perf skills exist per-language (Go, Rust, TS, React, Next.js, Tailwind, GSAP) but no cross-cutting **mx-perf-systems-*** for: load testing design, flame graph interpretation, latency budget allocation, capacity planning, degradation strategies.

**AI leverage:** Skill forces the AI to measure before optimizing, profile before guessing, and present latency budgets per component. Blocks the "premature optimization" and "throw a cache at it" rationalizations.

---

### 6. Observability That Actually Works
**The human problem:** Not "add logging." Designing a system where you can answer *arbitrary questions* about production behavior. Distributed tracing correlation, metric cardinality management, structured logging with context propagation, alerting that doesn't page at 3AM for noise. Most observability is write-only — terabytes of logs nobody queries.

**Skills gap:** Per-language observability skills exist. Missing: **mx-observability-platform-*** — OpenTelemetry architecture, trace-to-log correlation, metric cardinality control, SLO/SLI definition, alert design, runbook automation. Also missing: **mx-langfuse-*** for LLM-specific observability (you're running it in production right now).

**AI leverage:** Skill requires every new service to emit traces with correlation IDs, every error to include structured context, every metric to have defined cardinality bounds. The "log.info('error occurred')" rationalization is dead on arrival.

---

### 7. Correct Concurrency
**The human problem:** Race conditions, deadlocks, priority inversion, lock-free data structures, actor models. Bugs are non-deterministic, appear under load, disappear under debugger. Testing is nearly impossible — you can't unit test a race condition.

**Skills gap:** Go concurrency (1 skill) and Rust async (1 skill) cover language-specific patterns. No cross-cutting **mx-concurrency-*** for: lock hierarchy design, actor model patterns, CRDT selection, optimistic vs pessimistic concurrency control, distributed locking.

**AI leverage:** Skill encodes lock ordering rules, forces channel/actor patterns over shared mutable state, requires `context.WithTimeout` on every lock acquisition. The AI structurally cannot generate the deadlock.

---

### 8. Cross-System Integration (The Glue Problem)
**The human problem:** Making 5 vendor APIs work together when each has different auth, rate limits, error formats, pagination, retry semantics, and SLAs. The integration code is 60% of most B2B SaaS backends. It's tedious, error-prone, and breaks silently when vendors ship changes.

**Skills gap:** HubSpot (15 skills) covers one vendor deeply. Missing: **mx-integration-*** — circuit breaker patterns, retry with jitter, webhook verification, idempotent receivers, vendor change detection, API gateway patterns, rate limit budgeting across multiple APIs.

**AI leverage:** Skill forces every external API call through a standard adapter (auth, retry, timeout, circuit breaker, logging). The AI can't generate a raw `fetch()` to a vendor API — it must use the adapter pattern. This is exactly what skills are for: making the boring-but-critical thing mandatory.

---

## Tier 3: Problems That Define 10x Teams

### 9. Data Pipeline Reliability
**The human problem:** Idempotent processing, backpressure, dead letter queues, schema evolution in streaming data, exactly-once semantics. Failure modes are subtle and data loss is permanent. Most pipelines are "works on my laptop" → "silently drops 2% of records in prod."

**Skills gap:** AWS Streaming (1 skill) covers Kinesis. Missing: **mx-pipeline-*** — Prefect orchestration (mandated tool, zero coverage), idempotent processing patterns, checkpoint/resume, backfill strategies, data quality gates, schema evolution.

**AI leverage:** Skill requires every pipeline step to be idempotent, every write to have a deduplication key, every batch to checkpoint progress. The "I'll just rerun it" rationalization is blocked.

---

### 10. API Design That Ages Well
**The human problem:** Designing APIs that won't need breaking changes in 3 years. Versioning strategy, backward compatibility, deprecation paths, schema evolution, pagination that works at scale, error contracts. Most teams learn API design by shipping breaking changes.

**Skills gap:** AWS API Gateway (1 skill) covers infrastructure. Missing: **mx-api-design-*** — RESTful resource modeling, GraphQL schema design, versioning strategies, pagination patterns, error taxonomy, rate limiting design, OpenAPI/AsyncAPI spec-first development.

**AI leverage:** Skill forces spec-first development (write the OpenAPI spec, then generate code), requires pagination on all list endpoints, mandates semantic versioning on breaking changes. The AI can't generate an endpoint that returns unbounded arrays.

---

## What This Means for the Roadmap

### New packages surfaced by this analysis:

| Priority | Package | Skills | Why |
|----------|---------|--------|-----|
| HIGH | mx-langfuse-* | 3-4 | Running in prod NOW, zero coverage |
| HIGH | mx-prefect-* | 4-5 | Mandated orchestrator, zero coverage |
| HIGH | mx-security-* | 5-6 | Cross-cutting, existential risk |
| HIGH | mx-integration-* | 4-5 | 60% of B2B backend code |
| MED | mx-distributed-* | 4-5 | Correctness at scale |
| MED | mx-api-design-* | 4-5 | Prevents breaking changes |
| MED | mx-pipeline-* | 4-5 | Data reliability (overlaps Prefect) |
| MED | mx-modernization-* | 4-5 | Highest-value enterprise problem |
| LOW | mx-observability-platform-* | 4-5 | Cross-cutting (per-language exists) |
| LOW | mx-concurrency-* | 3-4 | Cross-cutting (per-language exists) |
| LOW | mx-perf-systems-* | 3-4 | Cross-cutting (per-language exists) |

### Packages that jumped in priority:
- **mx-postgres-*** (already #2) — validated as foundational
- **mx-python-*** (already #1) — required for Prefect
- **mx-graphql-*** — not on roadmap, should be (used in WP, Magento, triumvirate)
- **mx-redis-*** — not on roadmap, should be (active in triumvirate)
- **mx-cloudflare-*** — not on roadmap, low priority but in use (tunnels, workers exploration)
