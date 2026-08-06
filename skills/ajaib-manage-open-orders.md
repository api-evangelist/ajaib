---
name: Quote a book in batch and unwind it safely
description: >-
  Place a ladder of orders in one request, then cancel selectively or flatten
  the whole trading pair, handling the partial-success responses correctly.
api: openapi/ajaib-coin-exchange-openapi.yml
operations:
  - getExchangeInfo
  - getOpenOrders
  - createBatchOrders
  - cancelOrder
  - cancelBatchOrders
  - cancelAllOpenOrders
generated: '2026-08-06'
method: generated
source: https://ajaib.gitbook.io/coin-exchange/api-references/spot-trading
---

# Quote a book in batch and unwind it safely

Batch endpoints on Ajaib are **not atomic**. Every one of them can partially
succeed, and the response shape tells you which parts did. Treating a `200` as
"all done" is the main failure mode here.

Authenticate every call as described in `ajaib-signed-request.md`.

## Place a ladder

1. `getExchangeInfo` — confirm the `symbol` and read `base_precision` /
   `quote_tick`; round every rung of the ladder to them.
2. `createBatchOrders` (`POST /coin/internal/v1/order/batch`) with a single
   `symbol` and an `orders` array. Every order in the batch shares that symbol.
   Each item is `{side, type, price, quantity}`.
3. The response is an **array** of `{order_id, status}` — one per submitted
   order. Record every `order_id`; you need them to unwind. A `200` here still
   only means the exchange received the instructions.

## See what is live

`getOpenOrders` (`GET /coin/internal/v1/order/open`) with the `symbol`.
Paginate with `from_order_id` — note this cursor is **inclusive** of the
starting id, unlike `from_trade_id` on `getTrades`, which is exclusive. Results
come back most-recent-first. Only `NEW`, `OPEN` and `PARTIAL_FILLED` orders
appear.

## Unwind

Pick the narrowest tool that does the job:

- **One order** — `cancelOrder` (`DELETE /coin/internal/v1/order`) with
  `order_id` and `symbol` as query parameters. Returns
  `{"status": "CANCEL_SENT"}`. If the order already reached a terminal state
  you get `order_already_closed`; that is a benign race, not a failure to fix.
- **A known set** — `cancelBatchOrders` (`DELETE
  /coin/internal/v1/order/batch`) with `{order_ids: [...], symbol}` in the body.
- **Everything on the pair** — `cancelAllOpenOrders` (`DELETE
  /coin/internal/v1/order/open`) with `symbol`. This is the panic button; it
  takes down every open order on that trading pair.

## Handle the partial-success envelope

Both `cancelBatchOrders` and `cancelAllOpenOrders` return:

```json
{ "success": [123456, 789012], "fail": [543210] }
```

Always read **both** arrays. A `200` with a non-empty `fail` is the normal
outcome when some orders filled between your read and your cancel. Re-check any
id in `fail` with `getOrder` before retrying — it has usually just gone
terminal on its own.

## Cautions

- No idempotency key exists. A blind retry of `createBatchOrders` places the
  whole ladder a second time. On a timeout, call `getOpenOrders` and reconcile
  instead of resending.
- `createSelfTradingOrder` (`POST /coin/internal/v1/order/self-trading`) also
  exists but is restricted to internal market-maker exchange clients; it
  returns a paired `{buy_order_id, sell_order_id}`. Do not reach for it as a
  general trading endpoint.
- Ajaib publishes no rate-limit headers. Space out large batches and back off
  exponentially on `429`.
