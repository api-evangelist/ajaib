---
name: Sign an Ajaib Coin Exchange request
description: >-
  Build and sign an authenticated request to the Ajaib Coin Exchange API using
  the issued API key and an ECDSASHA256 request signature. Every other Ajaib
  skill depends on this one.
api: openapi/ajaib-coin-exchange-openapi.yml
operations:
  - getServerTime
  - getExchangeInfo
generated: '2026-08-06'
method: generated
source: https://ajaib.gitbook.io/coin-exchange/getting-started/authentication
---

# Sign an Ajaib Coin Exchange request

Ajaib does not use OAuth. Every authenticated call carries three headers, and
the signature is computed per request over the exact bytes you are about to
send. Get the concatenation wrong and you get `403` with `invalid_client`.

## Before you start

You need two things that cannot be self-served:

1. An `ECDSASHA256` keypair that you generated. Never share the private key.
2. An API key issued by Ajaib. Email your **public** key to `tech@ajaib.co.id`;
   Ajaib returns an API key bound to that public key and to your identity as an
   exchange client.

There is no signup form and no test key. Plan for a human round trip.

## Steps

1. Call `getServerTime` (`GET /coin/internal/v1/public/time`). It needs no
   auth. Use it to check connectivity and to measure your clock offset against
   the exchange. The response is `{"timezone": "UTC", "timestamp": <ms>}`.
2. Call `getExchangeInfo` (`GET /coin/internal/v1/public/exchange-info`) once
   and cache it. It gives you the valid `symbol` values plus `base_precision`
   and `quote_tick` — round every price and quantity to those before you send
   an order.
3. Build the payload string, in this exact order, with nothing between the
   parts:

   ```
   timestamp + method + requestPath + queryParam + requestBody
   ```

   - `timestamp` — unix epoch **milliseconds**, UTC. It must be byte-identical
     to what you put in `X-TIMESTAMP`.
   - `method` — uppercase: `GET`, `POST`, `PUT`, `DELETE`.
   - `requestPath` — leading slash, no trailing slash, e.g.
     `/coin/internal/v1/order`.
   - `queryParam` — all query parameters joined with `&`, no spaces, no
     newlines, e.g. `symbol=IDR&order_id=1`.
   - `requestBody` — the JSON body with every newline and space removed, e.g.
     `{"symbol":"BTC_USDT","type":"LIMIT","side":"BUY","price":100,"quantity":1}`.
     Empty string when there is no body.

4. Sign the payload with your private key using `ECDSASHA256`.
5. Send the request with these headers:

   | Header | Value |
   |---|---|
   | `X-API-KEY` | the API key Ajaib issued you |
   | `X-SIGNATURE` | the signature from step 4 |
   | `X-TIMESTAMP` | the same millisecond timestamp you signed |

6. Lowercase the URL. Ajaib states request URLs are case-sensitive and must be
   lowercase.

## Environments

- Production: `https://api.ajaib.co.id`
- Testnet: `https://api.ajaib.tech`

Same header scheme on both; they differ only by hostname. Ajaib publishes no
seeded testnet balances, so a testnet run may still need funding.

## When it fails

- `invalid_client` — the API key, signature or timestamp did not validate.
  Re-check that `X-TIMESTAMP` matches the timestamp inside the signed payload
  and that you stripped every space and newline from the body.
- `invalid_request` — a required parameter is missing or invalid.
- `429` — back off exponentially. Ajaib publishes no rate-limit headers and no
  quota, so you cannot know your budget before you are throttled.
