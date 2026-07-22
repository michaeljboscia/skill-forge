---
name: mx-ghl-opportunities
description: GoHighLevel / HighLevel API v2 opportunities and pipelines for AI agents — create/update opportunities, required pipelineId + pipelineStageId + status, resolving stage IDs at runtime (no API to create pipelines), monetaryValue handling, moving stages, and won/lost/abandoned status. Load when working with GHL/LeadConnector sales pipelines, deals, or opportunity automation.
---

# GoHighLevel Opportunities — Sales Pipelines for AI Coding Agents

**Loads when you create, move, or report on GoHighLevel opportunities and pipeline stages.**

## When to also load
- `mx-ghl-core` — auth, Version header, rate limits (always)
- `mx-ghl-contacts` — every opportunity ties to a contactId
- `mx-ghl-webhooks` — reacting to `OpportunityStatusUpdate` / `OpportunityStageUpdate`

---

## Level 1: Patterns That Always Work (Beginner)

### 1. Creating an opportunity needs four IDs plus a status
Required: `locationId`, `pipelineId`, `pipelineStageId`, `name`, `status`. Almost always also `contactId`.

```ts
✅ GOOD
await ghlFetch("/opportunities/", {
  method: "POST",
  body: JSON.stringify({
    locationId,
    pipelineId,                 // resolved at runtime, not hardcoded
    pipelineStageId,            // must belong to that pipeline
    name: "Acme — Website Rebuild",
    status: "open",             // open | won | lost | abandoned
    contactId,
    monetaryValue: 4500,        // number
  }),
});
```

### 2. `status` is a fixed enum
`open`, `won`, `lost`, `abandoned`. Anything else → 422. "Closed" is not a value; use `won`/`lost`.

### 3. Resolve pipeline & stage IDs at runtime — you cannot create them via API
There is **no v2 endpoint to create pipelines or stages** (UI only). Fetch existing ones and map by name.

```ts
const { pipelines } = await ghlFetch(`/opportunities/pipelines?locationId=${locationId}`).then(r => r.json());
const pipeline = pipelines.find(p => p.name === "Sales");
const stageId = pipeline.stages.find(s => s.name === "Proposal Sent").id;
```

### 4. `monetaryValue` is a plain number
No currency symbol, no string. `4500`, not `"$4,500"`. The currency is the location's setting; the API stores a bare number.

### 5. Move a stage with an update, don't recreate
```ts
await ghlFetch(`/opportunities/${opportunityId}`, {
  method: "PUT",
  body: JSON.stringify({ pipelineStageId: nextStageId }),
});
```

---

## Level 2: Lifecycle & Correctness (Intermediate)

### Stage must belong to its pipeline
A `pipelineStageId` from pipeline A sent with `pipelineId` B → 422 or a misfiled deal. Always take the stage from the resolved pipeline object.

### Status vs stage are independent
Marking `status: "won"` does **not** move the stage, and moving to a "Closed Won" stage does **not** set status. Set both when you close a deal.

```ts
await ghlFetch(`/opportunities/${id}`, {
  method: "PUT",
  body: JSON.stringify({ status: "won", pipelineStageId: closedWonStageId }),
});
```

### Search opportunities with the dedicated endpoint
```ts
const { opportunities } = await ghlFetch(
  `/opportunities/search?location_id=${locationId}&pipeline_id=${pipelineId}&status=open`
).then(r => r.json());
```
Filters include pipeline, stage, status, assignedTo, contactId, and date. Paginate (see `mx-ghl-core`).

### Don't duplicate a contact's deal
Before creating, check whether an open opportunity already exists for that contact+pipeline if your process should keep one live deal per contact.

---

## Level 3: Pipeline Sync & Reporting (Advanced)

### Build a name→id resolver keyed by pipeline
```ts
async function stageResolver(locationId: string) {
  const { pipelines } = await ghlFetch(`/opportunities/pipelines?locationId=${locationId}`).then(r => r.json());
  const map = new Map<string, { pipelineId: string; stageId: string }>();
  for (const p of pipelines)
    for (const s of p.stages) map.set(`${p.name}::${s.name}`, { pipelineId: p.id, stageId: s.id });
  return (pipeline: string, stage: string) => map.get(`${pipeline}::${stage}`);
}
```
Cache the resolver per location; refresh if a stage name misses.

### Report on value by stage without hardcoding stages
Iterate the pipeline's stages from the API so renamed/added stages don't break your report.

---

## Performance: Make It Fast

- **Fetch pipelines once, cache per location.** Every create/move otherwise wastes a request re-reading stage IDs.
- **Use `/opportunities/search` with filters** instead of listing all and filtering client-side.
- **Bounded concurrency** for bulk stage moves — the 100/10s burst applies (see `mx-ghl-core`).

```ts
❌ BAD — refetch pipelines for every deal
for (const d of deals) { const s = await stageResolver(loc); await create(d, s); }

✅ GOOD — resolve once
const resolve = await stageResolver(loc);
await Promise.all(deals.map(d => limit(() => create(d, resolve))));
```

## Observability: Know It's Working

- Track **422 rate on create/move** — a jump means a pipeline or stage was renamed/deleted and your cached IDs are stale.
- Monitor **stuck opportunities**: count deals whose stage hasn't changed past an SLA; surfaces dead pipeline automation.
- Reconcile **sum of monetaryValue per stage** against your source of truth to catch dropped or duplicated deals.
- Subscribe to `OpportunityStatusUpdate`/`OpportunityStageUpdate` webhooks to observe changes without polling (see `mx-ghl-webhooks`).

---

## Enforcement: Anti-Rationalization Rules

### Rule 1: Never hardcode pipeline/stage IDs
**You will be tempted to:** paste a `pipelineStageId` you saw in a response into a constant because it "won't change."
**Why that fails:** IDs are per-location and change when stages are edited; the deal lands in the wrong stage or 422s in every other sub-account.
**The right way:** resolve pipeline + stage by name at runtime from `/opportunities/pipelines`, cached per location.

### Rule 2: Don't try to create pipelines via API
**You will be tempted to:** POST to a `/pipelines` endpoint to set up stages programmatically.
**Why that fails:** there is no v2 endpoint to create pipelines/stages; you'll invent a URL that 404s.
**The right way:** create pipelines in the UI (or via SaaS setup), then read their IDs via the API.

### Rule 3: Status is an enum, and it's separate from stage
**You will be tempted to:** set `status: "closed"` or assume moving to a "Won" stage marks the deal won.
**Why that fails:** `closed` is invalid (422), and stage vs status are independent fields — reports read `status`, so the deal shows as still open.
**The right way:** use `won`/`lost`/`abandoned`/`open`, and set both status and stage when closing.

### Rule 4: monetaryValue is a number
**You will be tempted to:** pass `"$4,500"` or a formatted string from a UI.
**Why that fails:** the field expects a number; a string 422s or stores garbage, breaking pipeline value reports.
**The right way:** send `4500` as a number; format for display only in your own UI.

### Rule 5: Stage must belong to the pipeline you named
**You will be tempted to:** reuse a stage ID across pipelines because the names look similar.
**Why that fails:** a stage from another pipeline mismatches `pipelineId` → 422 or a misfiled opportunity.
**The right way:** always take `pipelineStageId` from the same pipeline object you took `pipelineId` from.
