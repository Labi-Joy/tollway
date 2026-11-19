# ADR for Dummies: Make channel close crash-safe and resume stuck channels from chain state

- **Status:** Proposed, awaiting maintainer approval
- **Date:** 2026-11-19
- **Technical ADR:** [ADR-2026-11-19-channel-resume](ADR-2026-11-19-channel-resume.md)

## The idea

Today, if the agent dies while closing a channel, the channel is stuck forever. If the network says "I may have closed it, I am not sure", we currently pretend nothing happened, which can let the channel be used again after the chain already closed it. We make close survive a crash and teach the agent to ask the chain what actually happened.

## What people see

1. An agent closes a channel. The agent first writes down "I am about to close, here is my attempt id and the ledger I started from", then tells the network.
2. If everything goes well, the channel ends up closed and the settlement details are saved.
3. If the agent crashes in the middle, the channel is left in `closing` until someone runs resume.
4. On startup (or on a manual call), the agent asks the chain, for each channel in `closing`, what happened:
   - If the chain never saw the close, the agent tries again safely under the same attempt id.
   - If the chain saw it and the money has moved, the agent records the settlement details and marks the channel closed.
   - If the funder started closing it themselves (the "dispute" shape), the agent hands the channel to the recovery feature instead of touching it.
5. If the amount the network settled does not match what the last receipt said, the channel is closed and flagged for review. It is not reopened, because the chain has already acted.

## Important limitation

The chain only keeps event history for a limited time on each RPC provider. If an agent has been offline for a very long time and the events have fallen out of the window, resume can not tell what happened. The agent surfaces this loudly and leaves the channel in `closing` for an operator to look at, rather than guessing. The real `@stellar/mpp` adapter and the funder-initiated recovery path are tracked in separate issues.

## What changes

- Close writes a small "attempt" record before calling the network, so a crash is detectable.
- An adapter error that might have submitted no longer rolls the channel back to `open`.
- The adapter grows two read-only chain queries (balance, close-effective ledger, settlement lookup). The test double is included; the real adapter is tracked separately.
- A new `resume(channelId)` and `resumeAll()` decide from chain state what to do.
- A settlement mismatch becomes a terminal flagged state, not a reopened channel.
- The stale note in `docs/design.md` about `@stellar/mpp` 0.6.0 is updated to the current 0.7.1 surface.

## What does not change

- Open and commit are unchanged.
- The channel data shape is backwards compatible: new fields are optional.
- Funder-initiated recovery stays in its own issue.
- Tollway still never holds, routes, or escrows funds. Resume only reads chain state and writes the same kind of record the manager already writes.
