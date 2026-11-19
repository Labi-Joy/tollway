# ADR: Retry x402 payments with a fresh authorization on expiry or nonce reuse

- **Status:** Proposed, awaiting maintainer approval
- **Date:** 2026-11-19
- **Issue:** #203
- **Scope:** `packages/client` changes. `packages/core` grows a typed settler-rejection discriminator. `packages/server` and `apps/web` get documentation updates. No change to the server gate, the facilitator adapter, or the receipt format.

## Context

`docs/design.md`, section "Failure semantics", commits the agent to "x402 retries with a fresh auth entry on expiry or nonce reuse." Nothing in the client implements this today.

`packages/client/src/index.ts` (lines 125 to 170) calls `options.settler.settle(...)` exactly once. Any rejection flows through `normalizeError` at line 247, is emitted as a `{ type: "error", ... }` event, and is thrown out of `pay()`. The `ClientEvent` union at lines 34 to 40 has no `retry` variant.

On the gate side, a rejection is `{ kind: "rejected"; status: number; reason: string }` (`packages/server/src/gate.ts` line 82). The three rejection shapes the client needs to tell apart are already distinct on the wire but only in HTTP status plus a free-form reason string:

- 402 verification failure (`gate.ts` line 198), where `reason` comes from `invalidReason` or `invalidMessage` on the facilitator response (`packages/server/src/facilitator.ts` lines 174 and 186). An expired authorization or a reused nonce surfaces here, but its reason text is whatever the upstream facilitator chose.
- 409 with `reason: "settlement in progress"` (`gate.ts` lines 155 to 157), set when another in-flight request owns the same payment nonce.
- 409 offer mismatch (`gate.ts` lines 149 to 150), set when a cached receipt was issued for a different offer.
- 502 settle failure (`gate.ts` line 436), set when `/settle` itself errored after the payment was sent. The transaction may have landed on chain; the response did not.

Branching on the English reason text to decide "retry with a new signature" would be wrong, because the facilitator owns that text. #161 adds a machine-readable `code` field and a `Retry-After` header to every rejection; this ADR consumes that contract.

The distinction matters because signing a new authorization after a 502 (or a timeout after `/settle` was invoked) can charge the payer twice. The policy engine's budget accounting is the second half: `record()` runs after a successful settlement (`packages/core/src/policy.ts` line 214), so double-recording would also double-count the budget. `check()` at line 114 is the pre-flight gate; the current type has no `reserve` (that is #189).

`@x402/core` and `@x402/stellar` are pinned at `^2.16.0` in `packages/server/package.json`. `PaymentRequirements.maxTimeoutSeconds` bounds how long a signed authorization stays valid, so a stale set of requirements must not be reused on a retry.

## Decision

1. **Add a `retry` option to `createClient`** of shape `{ maxAttempts: number; clock?: Clock; sleep?: (ms: number) => Promise<void> }`. The default is `{ maxAttempts: 2 }` and the hard upper bound is 5, documented and enforced at config parse. `clock` and `sleep` are injectable so tests drive time deterministically. #201's paid-fetch helper honours the same option.

2. **Define a typed settler-rejection discriminator** in `@vegaforge/tollway-core`:
   ```ts
   type SettlerFailure =
     | { kind: "retryable-with-fresh-auth"; code: "auth-expired" | "nonce-reused"; retryAfterMs?: number }
     | { kind: "retryable-with-same-auth"; code: "settlement-in-progress"; retryAfterMs: number }
     | { kind: "ambiguous"; code: "settle-failed" | "timeout"; receiptCandidate?: unknown }
     | { kind: "terminal"; code: string; reason: string };
   ```
   The settler returns this on failure. The client does not parse English reason strings. The settler maps #161's gate codes onto this type at its own boundary.

3. **Retry only `retryable-with-fresh-auth`** by signing again and resubmitting. The client re-runs `options.policy.check(intent)` before each fresh signature and stops if the policy now blocks (returning `PolicyBlockedError`). If the challenge changed between attempts (amount, asset, payee, or network differ from the original offer), the client stops and returns a terminal error named `OfferChangedError`.

4. **Replay `retryable-with-same-auth` with the identical payment header** after `retryAfterMs`, bounded by a maximum wait documented alongside `maxAttempts`. The gate's idempotency (keyed by the payment nonce; `packages/server/src/facilitator.ts` `deriveNonce`) returns the original receipt, so this is a safe way to resolve the 409.

5. **Never retry `ambiguous`.** The client throws a typed `AmbiguousPaymentError extends TollwayError` with `code: "ambiguous"` and a `receiptCandidate` field when the settler saw one. Reconciliation is the caller's job and is #211's territory.

6. **Record exactly once.** `options.policy.record(intent)` runs once on the first successful attempt, no matter how many retries were needed. Every other path, including terminal failures, skips `record`.

7. **New `ClientEvent`:** `{ type: "retry"; resource: string; attempt: number; reason: SettlerFailure["code"] }`. Emitted before each attempt after the first.

8. **Documentation.** `packages/client/README.md` and `apps/web/src/content/docs/getting-started.mdx` document the retry option, the four failure kinds, and the explicit rule that ambiguous outcomes are not retried.

## Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Caller
    participant Client as createClient().pay()
    participant Policy
    participant Settler
    participant Gate

    Caller->>Client: pay(request)
    Client->>Policy: check(intent)
    Policy-->>Client: allowed
    loop up to maxAttempts
        Client->>Settler: settle({offer, model})
        Settler->>Gate: POST /resource (X-Payment)
        alt 200 receipt
            Gate-->>Settler: signed receipt
            Settler-->>Client: receipt
            Client->>Policy: record(intent)
            Client-->>Caller: {receipt, model}
        else 402 retryable-with-fresh-auth
            Gate-->>Settler: {code: "auth-expired"|"nonce-reused"}
            Settler-->>Client: SettlerFailure
            Client->>Policy: check(intent)
            Note over Client: emit retry; new signature on next loop
        else 409 settlement-in-progress
            Gate-->>Settler: {code, Retry-After}
            Settler-->>Client: SettlerFailure
            Note over Client: wait, replay same header
        else 502 / timeout
            Gate-->>Settler: ambiguous
            Settler-->>Client: SettlerFailure
            Client-->>Caller: AmbiguousPaymentError
        else offer changed
            Client-->>Caller: OfferChangedError
        else policy now blocks
            Client-->>Caller: PolicyBlockedError
        end
    end
```

## Consequences

- The client surface grows by one option and one event. Both are additive; callers that do not set `retry` keep the current single-attempt behaviour but now get typed errors they can branch on.
- The settler interface grows a typed failure channel. x402 and MPP settlers both target the same shape, so later channel work in #176 can reuse the ambiguous/retryable discriminator.
- Double-pay is structurally prevented for the three known cases: the same `X-Payment` is never resent against a 402, a 502 never triggers a new signature, and `record` runs once.
- A misbehaving facilitator that reports `auth-expired` when the authorization is actually valid could waste signatures up to `maxAttempts`. The bound is the mitigation; the per-attempt policy re-check is the second line.
- The retry loop pulls fresh `PaymentRequirements` from the next 402, so `maxTimeoutSeconds` is honoured without caching stale requirements on the agent side.

## Alternatives considered

- **Parse rejection reason strings.** Rejected. The facilitator owns that text, and it moves without warning.
- **Infinite jittered backoff.** Rejected. Money moves on retry; a bounded attempt count is the point.
- **Put retry logic in the settler.** Rejected. The settler should stay a thin signer and submitter so other models (channels) can share the orchestration shape. Retry policy, policy accounting, and events are the client's job.
- **Treat 502 as retryable with a new signature.** Rejected. A tx may have landed on chain and only the response was lost. See #211 for reconciliation.

## Verification

Red first for the three known defects named in the issue:

1. A scripted settler that returns `retryable-with-fresh-auth` then success. Assert two signatures and one `record`.
2. A scripted settler that returns `retryable-with-same-auth` with `retryAfterMs`, then success. Assert the same `X-Payment` is replayed and `record` ran once.
3. A scripted settler that returns `ambiguous`. Assert `AmbiguousPaymentError` is thrown, `record` did not run, and the error carries any `receiptCandidate` the settler saw.

Also:

- A test that a changed offer on the retry stops the loop with `OfferChangedError`.
- A test that `maxAttempts` is honoured and the last failure is surfaced, not swallowed.
- A test that `policy.check` is re-run before each fresh signature and that a now-blocking policy stops the loop with `PolicyBlockedError`.
- A fake clock test that `retryAfterMs` is respected and bounded.

All through `packages/client/src/client.test.ts`. `./scripts/preflight.sh` must pass.
