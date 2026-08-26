---
name: shop-prime-store
description: >-
  Search the PRIME (Prime Hydration) catalog, build a cart, and take a buyer through checkout
  on drinkprime.com using the store's Universal Commerce Protocol MCP endpoint - stopping short
  of payment, which requires explicit human approval and cannot be undone.
api: PRIME Storefront Agentic Commerce API (UCP / MCP)
endpoint: https://drinkprime.com/api/ucp/mcp
transport: MCP (JSON-RPC 2.0 over HTTP POST)
auth: none
operations:
  - search_catalog
  - lookup_catalog
  - get_product
  - create_cart
  - get_cart
  - update_cart
  - cancel_cart
  - create_checkout
  - get_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
grounding: mcp/prime-hydration-mcp-tools.json
generated: '2026-08-26'
method: generated
source: >-
  Tool names, required arguments and semantics taken verbatim from the live tools/list response
  at https://drinkprime.com/api/ucp/mcp (HTTP 200, 2026-08-26). Policy rules taken verbatim from
  https://drinkprime.com/llms.txt and https://drinkprime.com/robots.txt.
---

# Shop the PRIME store

PRIME (Prime Hydration, LLC) sells hydration drinks, zero-sugar hydration, energy drinks,
hydration sticks and protein shakes at `drinkprime.com`. The store implements the Universal
Commerce Protocol over MCP. You can call it anonymously.

## Before you start

- Read `https://drinkprime.com/agents.md` — the provider names it the canonical agent
  instruction document. `https://drinkprime.com/llms.txt` mirrors it.
- Confirm capabilities at `GET https://drinkprime.com/.well-known/ucp`.
- Every tool call requires `meta.ucp-agent.profile` — a URI identifying you. Send it every time.

## The one rule that matters

**Do not call `complete_checkout` without explicit, contemporaneous approval from the buyer.**
The provider states this in both `robots.txt` and `llms.txt`. It is not advisory:

> Checkouts are for humans. Do NOT complete checkout, payment, or order placement automatically
> — no scripted form fills, browser automation, or end-to-end agent flows that finalize payment
> without an explicit, contemporaneous human approval step.

And know the consequence before you act: **there is no refund path.** The published refund
policy (`https://drinkprime.com/policies/refund-policy`) says all direct purchases are final
sale. There is no refund, void or reverse tool in the tool set. Once `complete_checkout`
succeeds, the money is spent.

## Steps

1. **Discover** — `GET /.well-known/ucp` and confirm `dev.ucp.shopping` is present at a version
   you support (`2026-04-08` is latest stable; `2026-01-23` is also served).
2. **Search** — call `search_catalog` with `catalog.query` (natural language) and/or filters.
   At least one is required. Pass `catalog.context.address_country` and
   `catalog.context.currency` so prices and availability are right for this buyer.
3. **Read more** — `get_product` for full detail on one item, `lookup_catalog` to resolve
   several products or variants by identifier at once.
4. **Cart** — `create_cart` with the chosen line items. Keep the returned
   `gid://shopify/Cart/...` id. Use `update_cart` to change quantities, `get_cart` to re-read.
5. **Check out** — `create_checkout`. Keep the returned `gid://shopify/Checkout/...` id.
6. **Fulfil** — `update_checkout` to set shipping address and method. This store ships to **one
   destination per checkout** and supports only the `shipping` method combination
   (`allows_multi_destination.shipping: false` in the UCP profile) — if the buyer wants items
   sent to two addresses, that is two checkouts.
7. **Present, then ask** — show the buyer the line items and totals. Convert prices first:
   amounts are integers in ISO 4217 **minor units**, so `{"amount": 600, "currency": "USD"}`
   is **$6.00**. Never quote `600`.
8. **Complete — only on approval** — `complete_checkout` with `meta.idempotency-key` set to a
   value you generate and reuse on any retry. It is a required field. Returns an order ID and a
   Thank You Page URL.
9. **Confirm** — `get_order` with the returned order id.

## Backing out

- `cancel_cart` reverses a cart. `cancel_checkout` reverses a checkout.
- Neither has a published validity window, so do not assume how long you have. Cancel promptly.
- Nothing reverses `complete_checkout`.

## Failure handling

- **429** — you are rate limited per IP. Back off. The provider publishes no numeric limit, no
  window and no `Retry-After` or `RateLimit-*` header contract, so use exponential backoff.
- Transport failures come back as JSON-RPC 2.0 errors. There is no error-code registry and no
  RFC 9457 problem+json; do not pattern-match on error strings.
- On a `complete_checkout` timeout, **retry with the same `meta.idempotency-key`** — that is
  what it is for. Retrying with a fresh key risks charging the buyer twice.

## What this store does not offer

No sandbox or test mode. No dry run. No refunds. No webhooks or event stream. No OpenAPI. If
you need to rehearse, the closest safe approximation is `create_checkout` followed immediately
by `cancel_checkout` against the real store — use it sparingly.
