---
name: Place a spot order and track it to a terminal state
description: >-
  Check the book, place a LIMIT order on the Ajaib Coin Exchange, and follow it
  from acknowledgement to a terminal status without duplicating it.
api: openapi/ajaib-coin-exchange-openapi.yml
operations:
  - getExchangeInfo
  - getDepth
  - getPrice
  - getPortfolio
  - createOrder
  - getOrder
  - getTrades
generated: '2026-08-06'
method: generated
source: https://ajaib.gitbook.io/coin-exchange/api-references/spot-trading
---

# Place a spot order and track it

This flow moves real funds. Read the retry warning at the bottom before you
write any retry logic.

Authenticate every call as described in `ajaib-signed-request.md`.

## Steps

1. **Validate the market.** `getExchangeInfo` — confirm the `symbol` exists and
   read `base_precision` and `quote_tick`. Round your `quantity` to
   `base_precision` decimals and your `price` to a multiple of `quote_tick`.
2. **Check you can pay.** `getPortfolio` — find the asset you are spending
   (`quote_asset` for a `BUY`, `base_asset` for a `SELL`) and confirm `free`
   covers the order. `locked` is already committed to open orders and is not
   available. Skipping this is how you get `insufficient_wallet`.
3. **Price it.** `getPrice` for the last traded value, and `getDepth` with a
   `limit` up to 100 for the live bids and asks. For a `LIMIT_MAKER` order,
   price it so it will not cross the book — if it would be a taker it is
   expired instead of filled.
4. **Place it.** `createOrder` (`POST /coin/internal/v1/order`) with
   `{side, symbol, type, price, quantity}`. Only `LIMIT` and `LIMIT_MAKER` are
   accepted on this endpoint even though `MARKET` appears in the type
   enumeration. You get back `{order_id, status}` — typically `status: NEW`.
5. **Understand what you just got.** A `200` means the exchange *received* the
   instruction. It is not a guarantee that the order was accepted by the
   matching engine. `NEW` means received but not yet valid; `OPEN` means the
   matching engine has it.
6. **Poll to a terminal state.** `getOrder` with `order_id` **and** `symbol`
   (both are required — the order id alone does not identify the pair). Watch
   `status`:
   - in flight: `NEW`, `OPEN`, `PARTIAL_FILLED`
   - terminal: `FILLED`, `CANCELLED`, `PARTIAL_CANCELLED`, `REJECTED`,
     `EXPIRED`, `EXPIRED_IN_MATCH`, `PARTIALLY_EXPIRED_IN_MATCH`

   Track `executed_quantity` against `quantity` and `avg_price` for the blended
   fill price. `avg_price` is `0` while nothing has matched.
7. **Reconcile the fills.** `getTrades` with the same `symbol`, paging with
   `from_trade_id` (exclusive) and `limit` (default 100, max 1000). Filter to
   your `order_id`. Each trade carries `price`, `quantity`, `timestamp`,
   `is_maker` and `is_self_trade`.

## Retry rule — read this

Ajaib documents **no idempotency key**. The published response-code table
mentions HTTP `409` possibly arising from "the same idempotent key", but no key
header or parameter exists on any endpoint.

So: **never blind-retry `createOrder`.** If a call times out or the connection
drops, do not resend. Instead call `getOpenOrders` for the symbol and inspect
recent orders to determine whether yours landed, then decide. A naive retry
places a second real order.

## Errors

- `insufficient_wallet` — not enough `free` balance. Re-read `getPortfolio`.
- `invalid_request` — missing or invalid parameter; check precision and tick
  rounding first.
- `invalid_client` — signature or timestamp problem; see the signing skill.
