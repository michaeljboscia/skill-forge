# Skill-Forge Roadmap

**Last Updated:** 2026-07-22
**Shipped:** 150 skills across 16 packages
**Pipeline:** 20 packages (~107 skills)

---

## Shipped Packages

| # | Package | Skills | Status |
|---|---------|--------|--------|
| 1 | mx-aws-* | 25 | Shipped |
| 2 | mx-gcp-* | 16 | Shipped |
| 3 | mx-hubspot-* | 15 | Shipped |
| 4 | mx-supa-* | 10 | Shipped |
| 5 | mx-go-* | 10 | Shipped |
| 6 | mx-react-* | 8 | Shipped |
| 7 | mx-nextjs-* | 8 | Shipped |
| 8 | mx-rust-* | 8 | Shipped |
| 9 | mx-tw-* | 8 | Shipped |
| 10 | mx-ts-* | 8 | Shipped |
| 11 | mx-wp-* | 7 | Shipped |
| 12 | mx-gsap-* | 4 | Shipped |
| 13 | mx-gpu-* | 3 | Shipped |
| 14 | mx-lottie-* | 4 | Shipped 2026-04-06 |
| 15 | mx-k3s-* | 8 | Shipped 2026-04-08 |
| 16 | mx-ghl-* | 8 | Shipped 2026-07-22 |

---

## Pipeline — Tier 0: In Progress

### ~~0. mx-lottie-*~~ SHIPPED 2026-04-06
**Skills:** core, interaction, perf, observability (4)
**Lines:** 1,024 SKILL.md + 1,856 reference = 2,880 total
**Anti-rat rules:** 17
**Pressure test:** 9 defects found, 9 fixed, PASS
**Status:** Deployed to ~/.claude/skills/mx-lottie-*/

---

## Pipeline — Tier 1: Language & Runtime

These are horizontal skills — make you better at writing code in a specific language/runtime.

### 1. mx-python-*
**Skills:** core, async, data, web-fastapi, testing, project, observability, perf (8)
**Why:** Glue code, automation, ML tooling, scripts everywhere. Required foundation for mx-prefect-*.
**Status:** Not started

### 2. mx-postgres-*
**Skills:** schema, queries, indexing, migrations, perf, reliability, security (7)
**Why:** Most backend bugs/perf issues are data-layer mistakes. Complements mx-supa-* (Supabase-specific) — this is raw Postgres for any deployment.
**Status:** Not started

### 3. mx-docker-k8s-*
**Skills:** docker-core, compose, k8s-core, deploy, observability, security (6)
**Why:** Closes the "works locally, fails in deploy" gap. Required for Langfuse, daemon deployment patterns.
**Status:** Not started

### 4. mx-node-backend-*
**Skills:** express/fastify, queues, auth, testing, observability, perf (6)
**Why:** Complements TS frontend batteries with server-side depth.
**Status:** Not started

### 5. mx-java-* or mx-kotlin-* (pick one first)
**Skills:** core, spring, data, testing, observability, perf (6)
**Why:** Enterprise interoperability and API integration.
**Status:** Not started
**Decision needed:** Java or Kotlin first?

### 6. mx-csharp-*
**Skills:** .net-core, webapi, efcore, testing, observability, perf (6)
**Why:** Common in B2B customer ecosystems.
**Status:** Not started

### 7. mx-cpp-*
**Skills:** core, concurrency, build, testing, perf, safety (6)
**Why:** Systems-level and performance-critical integrations.
**Status:** Not started

---

## Pipeline — Tier 2: Systems Problems (Vertical)

These are vertical skills — they target *problems*, not languages. They cross every language boundary. This is where the hardest human challenges live.

### 8. mx-prefect-*
**Skills:** flows, tasks, deployments, error-handling, observability (4-5)
**Why:** Mandated orchestration tool (no n8n/Make/Zapier). Currently zero coverage for the only workflow engine we use.
**Depends on:** mx-python-*
**Status:** Not started

### 9. mx-security-*
**Skills:** threat-modeling, secrets, supply-chain, input-validation, headers-cors, rbac (5-6)
**Why:** Cross-cutting, existential risk. Most devs skip it because invisible until breach. Per-cloud security skills exist (AWS, GCP) but nothing cross-cutting.
**Status:** Not started

### 10. mx-integration-*
**Skills:** adapter-patterns, circuit-breakers, retry-jitter, webhook-verification, rate-budgeting (4-5)
**Why:** 60% of B2B backend code is glue between vendor APIs. 5 APIs, 5 auth schemes, silent failures. This is the daily reality of enterprise software.
**Status:** Not started

### 11. mx-api-design-*
**Skills:** rest-modeling, versioning, pagination, error-contracts, spec-first (4-5)
**Why:** Prevents breaking changes in year 2. Forces spec-first development, pagination on all list endpoints, semantic versioning.
**Status:** Not started

### 12. mx-pipeline-*
**Skills:** idempotency, checkpointing, backfill, schema-evolution, data-quality (4-5)
**Why:** Silently drops 2% of records in prod. Every step must be idempotent, every write deduplicated. Overlaps with Prefect for orchestration layer.
**Status:** Not started

### 13. mx-distributed-*
**Skills:** consensus, exactly-once, circuit-breakers, crdts, distributed-locking (4-5)
**Why:** Bugs are non-deterministic, unreproducible in dev, catastrophic in prod. Forces idempotency + timeout + circuit breaker on every external call.
**Status:** Not started

### 14. mx-modernization-*
**Skills:** strangler-fig, anti-corruption-layer, domain-extraction, dual-write, contract-testing (4-5)
**Why:** Highest-value enterprise problem. Humans take 6-18 months to map a monolith. AI + skills: weeks. Legacy system migration is where the money is.
**Status:** Not started

---

## Pipeline — Tier 3: Platform & Domain

### 15. mx-langfuse-*
**Skills:** tracing, cost-tracking, prompt-management, evaluation (3-4)
**Why:** Running in production NOW on server (port 3300), zero skill coverage. LLM observability for multi-agent systems.
**Status:** Not started

### 16. mx-graphql-*
**Skills:** schema-design, resolvers, subscriptions, codegen, federation (4-5)
**Why:** Used in WPGraphQL, Magento GraphQL, triumvirate. Cross-cutting query language with no standalone skills.
**Status:** Not started

### 17. mx-redis-*
**Skills:** caching, pub-sub, streams, clustering, persistence (4-5)
**Why:** Active in triumvirate (graphql-redis-subscriptions, ioredis). Session caching, real-time data flows.
**Status:** Not started

### 18. mx-magento-*
**Skills:** core, graphql, catalog, checkout, deploy, observability, perf (7)
**Why:** Headless Magento/Adobe Commerce — same headless pattern as mx-wp-*. E-commerce counterpart.
**Status:** Not started
**Existing corpus:** 39 deep reference docs (2MB) at `/Users/you/projects/EcommCodingIntegration`. Phase 1-2 research largely pre-done.

### 19. mx-cloudflare-*
**Skills:** tunnels, workers, pages, r2, d1 (4-5)
**Why:** Tunnels in production (Langfuse), Workers/Pages in exploration. Edge compute platform.
**Status:** Not started

---

## Pipeline — Tier 4: Gated

### 20. mx-mobile-* (if product needs it)
**Skills:** swift-ios, kotlin-android, react-native batteries (3+)
**Why:** Mobile product development if/when needed.
**Status:** Not started
**Gate:** Only build when a product requires it.

---

## Totals

| Category | Skills |
|----------|--------|
| Shipped | 150 |
| Tier 0 — In progress | 0 (Lottie shipped) |
| Tier 1 — Language/Runtime | 45 |
| Tier 2 — Systems Problems | 30 |
| Tier 3 — Platform/Domain | 24 |
| Tier 4 — Gated | 3 |
| **Pipeline total** | **~107** |
| **Projected total** | **~237** |
