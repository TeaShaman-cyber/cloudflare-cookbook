# Cloudflare Cookbook Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Establish a reproducible Cloudflare dev canary proving connector capabilities, durable receipt storage, independent readback, reconciliation, observability, and documented boundaries.

**Architecture:** Start with connector smoke testing before Worker code. Use a minimal Worker with native persistence; prefer D1 unless live evidence makes Durable Objects simpler. Receipt identity stays deterministic and storage errors fail closed. Vercel evidence remains independent.

**Tech Stack:** Cloudflare Workers, D1 preferred, Wrangler-compatible layout, modern JavaScript, Node test runner, GitHub, official Cloudflare ChatGPT plugin/MCP guidance.

**Spec:** `docs/superpowers/specs/2026-09-03-cloudflare-cookbook-design.md`

## Global Constraints
- No production migration.
- No mutation or closure of Vercel evidence because Cloudflare behaves differently.
- No assumption that Cloudflare and Vercel failures share a cause.
- No WAF bypass/evasion work.
- No secrets committed to Git.
- No production Worker until a dev canary passes end-to-end.
- Missing ChatGPT actions are connector boundaries, not platform limitations.
- Every material result records intent, surface, result/error class, execution boundary, independent readback, proof limits, and upstream references.

---

### Task 1: Cloudflare ChatGPT connector capability smoke test

**Files:**
- Create: `docs/field-notes/cloudflare-connector-smoke.md`
- Modify: `README.md`

**Interfaces:**
- Consumes authenticated official Cloudflare ChatGPT plugin.
- Produces matrix states `AVAILABLE`, `READ_ONLY`, `WRITE_VERIFIED`, `MISSING_ACTION`, `AUTH_REQUIRED`, `CONTROL_PLANE_BLOCKED`, `INCONCLUSIVE`.

- [ ] Install/connect the official plugin and record non-secret account visibility.
- [ ] Inventory actual ChatGPT actions for Workers, D1, Durable Objects, KV/R2, logs/analytics, secrets/config.
- [ ] Probe one harmless dev-only mutation if a documented create action exists; perform separate readback.
- [ ] Record exact action names and classify missing actions as connector boundaries.
- [ ] Commit with `docs: record Cloudflare connector smoke test`.

---

### Task 2: Deterministic Worker receipt core

**Files:**
- Create: `canaries/worker-watch/package.json`
- Create: `canaries/worker-watch/src/receipt.js`
- Create: `canaries/worker-watch/src/reconcile.js`
- Create: `canaries/worker-watch/test/receipt.test.js`
- Create: `canaries/worker-watch/test/reconcile.test.js`

**Interfaces:**
- Produces `normalizeWorkflowRun(payload, receivedAt)` and `reconcileReceipts(receipts, target)`.

- [ ] Write a failing test proving receipt identity is unchanged when only `received_at` changes.
- [ ] Run `cd canaries/worker-watch && npm test`; expect RED because implementation is absent.
- [ ] Implement canonical identity over schema version, source, repository, run id/name/status/conclusion/head SHA/source updated time using Worker-compatible SHA-256; exclude `received_at`.
- [ ] Add tests for malformed authority fields and reconciliation states `ABSENT`, `FRESH`, `STALE`.
- [ ] Implement minimal reconciliation selecting the newest matching receipt.
- [ ] Run tests; expect GREEN.
- [ ] Commit with `feat: add deterministic Cloudflare receipt core`.

---

### Task 3: D1 durable storage adapter

**Files:**
- Create: `canaries/worker-watch/schema.sql`
- Create: `canaries/worker-watch/src/storage.js`
- Create: `canaries/worker-watch/test/storage.test.js`
- Create: `canaries/worker-watch/wrangler.jsonc`

**Interfaces:**
- Consumes D1 binding `env.RECEIPTS_DB`.
- Produces `D1ReceiptStorage.append(receipt)` and `D1ReceiptStorage.list(limit)`.

- [ ] Define `receipts` table with `receipt_id TEXT PRIMARY KEY`, authority fields, timestamps, and full payload.
- [ ] Write failing tests for first insert, duplicate insert, ordered list, and storage error mapping to `{ok:false,state:"INCONCLUSIVE_STORAGE"}`.
- [ ] Implement parameterized `INSERT OR IGNORE`; determine duplicate from D1 result metadata, not pre-read.
- [ ] Configure only a dev D1 binding in `wrangler.jsonc`; no production routes or secrets.
- [ ] Run tests; expect GREEN.
- [ ] Commit with `feat: add D1 receipt storage`.

---

### Task 4: Worker HTTP canary and dev end-to-end proof

**Files:**
- Create: `canaries/worker-watch/src/index.js`
- Create: `canaries/worker-watch/test/http.test.js`
- Create: `canaries/worker-watch/README.md`
- Modify: `receipts/README.md`

**Interfaces:**
- Produces `GET /health`, `POST /synthetic-event`, `GET /receipts`, `GET /reconcile`.

- [ ] Write failing HTTP tests: `/health` 200; first receipt 202; duplicate 202 with `duplicate:true`; D1 failure 503 with `INCONCLUSIVE_STORAGE`.
- [ ] Implement route dispatch and reject unsupported methods with 405.
- [ ] Ensure `/reconcile` never converts storage failure into `ABSENT`.
- [ ] Run local tests; expect GREEN.
- [ ] Deploy through the connector only if Task 1 classified deploy as `WRITE_VERIFIED`; otherwise use the documented approved fallback and record the distinction.
- [ ] Perform separate runtime calls for health, first POST, independent readback, duplicate POST, second readback, reconcile, and logs.
- [ ] Write a bounded receipt containing ids/hashes/results only; no secrets or full provider responses.
- [ ] Commit with `feat: complete Cloudflare Worker durable canary`.

---

### Task 5: Cloudflare skills to connector isomorphism

**Files:**
- Create: `docs/research/cloudflare-skills-connector-isomorphism.md`
- Modify: `docs/field-notes/cloudflare-connector-smoke.md`

**Interfaces:**
- Produces `directly_mappable`, `partially_mappable`, `cli_bound`, `connector_missing_capability`, `out_of_scope` classifications.

- [ ] Inventory official Cloudflare workflows for Workers, D1, logs, bindings, secrets, and deployment.
- [ ] Map each workflow to the actual ChatGPT surface and documented fallback, if any.
- [ ] Treat documentation/connector mismatches as connector gaps rather than normalizing them away.
- [ ] Commit with `docs: map Cloudflare skills to connector surface`.

---

## Final Verification
- [ ] All `canaries/worker-watch` tests pass.
- [ ] Dev `/health` succeeds through an independent read path.
- [ ] First synthetic receipt persists durably.
- [ ] Second invocation reads the same receipt.
- [ ] Duplicate delivery creates one logical row only.
- [ ] Storage failure yields `INCONCLUSIVE_STORAGE`, never `ABSENT`.
- [ ] Reconcile demonstrates FRESH, STALE, and ABSENT.
- [ ] Logs/observability are readable or an explicit connector boundary is recorded.
- [ ] Secret/config management has one documented supported route.
- [ ] No production route/domain or committed secret exists.
- [ ] Vercel evidence remains untouched.
