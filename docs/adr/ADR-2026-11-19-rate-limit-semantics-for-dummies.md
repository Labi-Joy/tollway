# ADR for Dummies: Rate-limit semantics: explicit count, scope and window with bounded memory

- **Status:** Proposed, awaiting maintainer approval
- **Date:** 2026-11-19
- **Technical ADR:** [ADR-2026-11-19-rate-limit-semantics](ADR-2026-11-19-rate-limit-semantics.md)

## The idea

Today's "60 requests per minute" cap is not quite what the docs say it is. It only counts payments that fully settled, mixes every service into one bucket, and is frozen at one minute. We make the rule say exactly what it does, let operators choose the shape, and keep its memory use bounded.

## What people see

1. An agent operator writes a policy with a rate limit. They can now say what is counted (attempts versus settlements), who it is scoped to (everyone together or per service they pay), the maximum, and the length of the window.
2. The old one-line shorthand still works and keeps today's behaviour.
3. When an attempt would cross a limit, the agent blocks it and the decision names the exact rule, the counterparty if any, the count, and the window. Operators can read a decision and know what to change.
4. Many limits can run together, for example "60 a minute per counterparty" and "1000 an hour across the board". The first one that would be exceeded is the one that blocks.
5. Memory is bounded. A flood of unique counterparties will not quietly grow the agent's memory; after a cap is reached, new counterparties are blocked loudly with a clear reason until the operator reacts.
6. The docs page explains all of this with a worked example and the exact point at which the counter advances.

## Important limitation

Counting attempts (rather than only settlements) only works correctly once the budget-reservation seam from issue #189 is in place, because without it concurrent attempts can all read the same count before any of them has advanced it. This ADR describes how the two features fit together; if #189 slips, attempt-scoped limits warn at config load so operators know to wait.

## What changes

- The `rateLimit` field in the policy accepts a list of rules, each with its own count, scope, cap, and window. The old one-liner is translated automatically.
- Counters are kept per counterparty when the rule says so.
- The engine evicts counterparties that have been idle for long enough and caps how many it will ever track.
- The rate-limit window is decoupled from the anomaly window, so changing one does not touch the other.
- The docs page restates what is counted, when, and for whom, with a worked example.

## What does not change

- The server gate is untouched. Service-side rate limiting of payers is a separate issue (#160).
- The default policy behaviour for existing configs is identical to today.
- The engine is still in-memory. No Redis, no shared state.
- Tollway still never holds, routes, or escrows funds. Rate limits live entirely in the agent's own policy.
