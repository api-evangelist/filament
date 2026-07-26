---
name: Place and manage a perpetual order on Filament
description: Discover a market, sign and submit a perpetual order (limit or market) to the Filament DEX on Sei, then poll its status and open positions.
api: https://docs.filament.finance/market-makers/filament-api
base_url: https://orderbook.filament.finance/sei
auth: wallet-signature
operations:
  - GET /filament/api/v1/assets
  - POST /filament/api/v1/exchange
  - POST /v1/orders/latest-status
  - GET /v1/orders/open-orders/paginated/account/:account
  - GET /v1/positions/trades/paginated/:account
---

# Place and manage a perpetual order on Filament

Filament is a perpetual DEX on Sei. Orders are authenticated by signing the order
id with an ethers.js wallet — there are no API keys. The signer's `account` address
must be lowercase in every payload.

## Prerequisites
- An ethers.js v5 wallet with USDC collateral bridged/deposited on Sei
  (see docs guides: bridging, connecting-to-sei, depositing-funds).
- `nanoid` for order ids.

## Steps

1. **List tradable markets** — `GET /filament/api/v1/assets`.
   Pick the `indexToken` / `assetName` (e.g. BTC) and note `markPrice`,
   `fundingRate`, `maxLeverage`, `minLeverage`.

2. **Build the order** — generate `orderId = nanoid().toLowerCase()`.
   Sign it: `signature = await signer.signMessage(orderId)`.
   Assemble the payload:
   ```json
   {
     "type": "order",
     "referralCode": null,
     "orders": [{
       "account": "<lowercase_address>",
       "indexToken": "BTC",
       "orderId": "<orderId>",
       "signature": "<signature>",
       "isBuy": true,
       "size": 22,
       "leverage": 1.1,
       "reduceOnly": false,
       "orderType": { "type": "limit", "limit": { "tif": "Gtc", "limitPrice": "57985" } }
     }]
   }
   ```
   For a market order use `"orderType": { "type": "trigger", "trigger": { "isMarket": true, "slippage": 5 } }`.

3. **Submit** — `POST /filament/api/v1/exchange` with the payload.
   Keep `size` within `minLeverage`/`maxLeverage` bounds for the asset.

4. **Confirm status** — `POST /v1/orders/latest-status` with the order id(s), or
   `GET /v1/orders/open-orders/paginated/account/:account?page=0&size=20`.

5. **Check the resulting position** —
   `GET /v1/positions/trades/paginated/:account?page=0&size=20` for open positions
   (entryPrice, quantity, collateral, liquidationPrice, leverage).

## Conventions & caveats
- Idempotency: none documented. Reusing an `orderId` is not a documented safe retry
  — treat each submission as a new order and reconcile via status endpoints.
- Pagination: `page` + `size` (plus `token`/`side` filters).
- Errors: standard HTTP status codes; no structured error catalog is published.
- Modify collateral with `{"type":"updateIsolatedMargin", ...}` (also signed).
- Cancel with `{"type":"cancel","cancels":[{account,orderId,signature}]}`.
- Stream fills/updates over WebSocket topic `/topic/order-updates/{account}`.
