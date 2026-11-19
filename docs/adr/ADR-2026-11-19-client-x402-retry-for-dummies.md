# ADR for Dummies: Retry x402 payments with a fresh authorization on expiry or nonce reuse

- **Status:** Proposed, awaiting maintainer approval
- **Date:** 2026-11-19
- **Technical ADR:** [ADR-2026-11-19-client-x402-retry](ADR-2026-11-19-client-x402-retry.md)

## The idea

When a Tollway agent tries to pay for a resource and the payment is rejected, the agent today just gives up. We want the agent to try once more, but only when it is safe to do so, and never in a way that could charge the same person twice.

## What people see

1. An agent developer calls `pay()` on the Tollway client to buy a resource.
2. The service asks for payment and the agent signs a short-lived authorization.
3. The service rejects it because the authorization has already expired (the agent took too long) or because the nonce was already used. This is safe to retry: nothing moved.
4. The client signs a fresh authorization and tries again. It runs the policy check a second time first, because a budget may have changed in between.
5. If the service instead says "I am already settling your last attempt, come back in a moment", the client waits the suggested time and resends the exact same payment. The service's own de-duplication returns the original receipt.
6. If the service returned a vague error after the payment was sent, like "settle failed" or the network timed out, the client does not retry. It tells the caller that the outcome is unknown so the caller can reconcile. This is the case that could otherwise cause a double charge.
7. The agent developer can see the retries through a new event on the client, so observability tools know when a payment needed a second attempt.

## Important limitation

The client does not reconcile unknown outcomes on its own. If a payment left the agent and the response never came back, the client surfaces an "ambiguous" error and stops. A separate feature (issue #211) will later use the service's own receipt-lookup to decide what actually happened. Retrying blindly in that case is exactly the double-pay this change is designed to avoid.

## What changes

- The Tollway client gains a `retry` option with a safe default. Only two attempts, with a hard upper bound.
- The client emits a new `retry` event so logs and dashboards can see the second attempt.
- Payment rejections now travel through a typed shape with codes like `auth-expired` and `settlement-in-progress`, instead of free-form strings. This closes a door where a change in wording upstream could silently break retry behaviour.
- The client README and the getting-started docs explain all of the above, including which cases are retried and which are not.

## What does not change

- The server gate, the facilitator adapter, and the receipt format are untouched.
- Callers that do not configure retries keep the current behaviour: one attempt, same errors, no new work for them to do.
- The policy engine still records a payment once per successful settlement. No double counting.
- Tollway never holds, routes, or escrows funds. The retry happens on the agent side; the money path on chain is unchanged.
