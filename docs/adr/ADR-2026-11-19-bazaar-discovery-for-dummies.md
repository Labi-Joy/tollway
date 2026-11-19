# ADR for Dummies: Publish paywalled routes through the x402 Bazaar discovery extension

- **Status:** Proposed, awaiting maintainer approval
- **Date:** 2026-11-19
- **Technical ADR:** [ADR-2026-11-19-bazaar-discovery](ADR-2026-11-19-bazaar-discovery.md)

## The idea

Services using Tollway can describe their paid routes (what the input looks like, what the output looks like, which method, which URL template) and send that description out through a standard extension. Facilitators that run a catalogue can then list those routes, and agents can find them. Tollway itself never runs the catalogue.

## What people see

1. A service operator writes a `PaywallConfig` as today. If they also want the route to appear in a catalogue, they add a `discovery` block with a short description, the method, a URL template with `:param` markers, and either the query parameters or the request-body shape.
2. Nothing changes for routes that leave `discovery` off.
3. When a paid request comes in, the agent and the facilitator exchange a standard extension that describes the route. The facilitator can read it and list the route on its catalogue. Facilitators that do not care just ignore it.
4. An agent developer who wants to browse a facilitator's catalogue uses a small helper (or a five-line recipe we publish) that calls the facilitator's `/discovery/resources` endpoint and returns a validated list.
5. The docs page explains what is published, to whom, how to opt in, and how to avoid leaking identifiers through URL templates.

## Important limitation

Tollway does not itself run a catalogue, and the agent's router does not use the catalogue to pick who to pay. We only publish. The reason is that trusting a catalogue to influence payment routing needs a trust story that does not exist yet; we would rather see operator evidence first. The design-doc open question is closed with that explicit "not yet" answer.

## What changes

- `PaywallConfig` gets one optional `discovery` block, validated at construction time by the upstream library so bad configs fail loud.
- The 402 challenge and the facilitator payload now carry the standard discovery extension when a route opts in.
- A docs page explains what is published and how to opt in or out.
- The `@x402/*` packages and `@x402/extensions` move together to 2.31.x, coordinated with the related issue tracking the upgrade.

## What does not change

- Receipts, policy, and the payment path are untouched.
- Routes that do not opt in publish nothing.
- Tollway still never holds, routes, or escrows funds. It still does not run a catalogue.
- Four adapters continue to share one gate implementation; no adapter-specific behaviour changes.
