# Cloudflare Cookbook Design

Date: 2026-09-03
Status: approved for specification review

## Purpose

Create a public reproducible working surface for evaluating Cloudflare as an alternative serverless substrate while keeping the Vercel investigation intact and resumable.

Cloudflare and Vercel remain independent experiments with the same acceptance criteria.

## Goals

1. Test the official Cloudflare ChatGPT connector/plugin as an authenticated control surface.
2. Build a minimal Worker canary with health, synthetic ingest, durable storage, readback, and reconciliation.
3. Compare auth/connect, create, deploy, readback, logs, storage, secret/config management, and repeatability.
4. Record connector gaps, provider/WAF behavior, receipts, and upstream issue links as field notes.
5. Adapt official Cloudflare skills to the actual ChatGPT connector surface without assuming CLI-only capabilities.

## Non-goals

- No production migration.
- No mutation or closure of Vercel evidence because Cloudflare behaves differently.
- No assumption that Cloudflare and Vercel failures share a cause.
- No WAF bypass/evasion work.
- No secrets committed to Git.
- No production Worker until a dev/preview canary passes end-to-end.

## Repository layout

```text
cloudflare-cookbook/
├── README.md
├── docs/
│   ├── field-notes/
│   ├── research/
│   └── superpowers/specs/
├── canaries/worker-watch/
└── receipts/
```

## First canary

```text
Cloudflare Worker
    +-- GET /health
    +-- POST /synthetic-event
    +-- GET /receipts
    +-- GET /reconcile
              |
              v
       durable Cloudflare storage
```

Candidate authority stores are D1 and Durable Objects. The first implementation should choose the smallest native option that provides deterministic persistence and independent readback without adding an external database dependency.

## Acceptance criteria

1. Authenticated connector/plugin connection succeeds.
2. A dev/preview Worker can be created or deployed without manual secret disclosure.
3. `/health` succeeds through an independent read path.
4. A synthetic receipt can be written durably.
5. A second invocation can read the same receipt.
6. Duplicate delivery does not create a second logical receipt.
7. Reconciliation distinguishes FRESH, STALE, and ABSENT without treating failed storage access as absence.
8. Runtime logs or equivalent observability are readable.
9. Storage/config/secret management has a documented supported route.
10. The result is reproducible from repository state plus documented account setup.

## Connector smoke test

Before Worker implementation inspect the official Cloudflare ChatGPT connector for actual exposed actions and record separately:

- identity/account visibility;
- Workers read/write actions;
- D1/Durable Objects/KV/R2 actions if present;
- logs/analytics actions;
- secret/env/config mutation actions;
- read-after-write behavior;
- differences between documented MCP/CLI capabilities and ChatGPT-exposed capabilities.

An absent ChatGPT action is a connector boundary, not proof of a Cloudflare platform limitation.

## Storage decision rule

Prefer native Cloudflare storage.

- D1 is preferred when schema/query/readback are simple and reproducible.
- Durable Objects are preferred when stateful receipt semantics are simpler without extra provisioning complexity.
- KV/R2 are later candidates, not first-choice receipt authorities.

## Skill isomorphism

After connector smoke testing, research official Cloudflare skills and agent/MCP guidance. Classify patterns as directly_mappable, partially_mappable, cli_bound, connector_missing_capability, or out_of_scope.

## Evidence rules

For every material discovery record operation intent, exact surface, observed result/error class, whether failure occurred before target execution, independent readback when available, what the evidence proves, what it does not prove, and upstream references.

Do not infer platform failure from connector failure.

## Initial issues after spec review

1. Cloudflare ChatGPT connector capability smoke test
2. Cloudflare Worker durable canary
3. Cloudflare skills -> connector isomorphism

## Vercel coexistence

Vercel is frozen, not abandoned. Its Neon/storage work and upstream connector issues remain authoritative in `vercel-cookbook`. Cloudflare findings do not overwrite Vercel evidence.

## Success condition

The design phase is complete when the public repository and this specification are independently readable. Implementation begins only after specification approval.
