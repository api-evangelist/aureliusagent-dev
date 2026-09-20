---
generated: '2026-09-19'
method: generated
name: Quote, pay for and hand off a capacity session
description: Browse live GPU, inference, edge-compute and workspace offers, take a free quote, pay for a session through the MPP 402 challenge, exchange peering details and confirm the direct connection; cancel if the session allows it.
api: openapi/aureliusagent-dev-walton-capacity-mpp-api-openapi.yml
operations: []
paths: ['GET /mpp/capacity/offers', 'POST /mpp/capacity/quotes', 'POST /mpp/capacity/sessions', 'GET /mpp/capacity/sessions/{sessionId}', 'POST /mpp/capacity/sessions/{sessionId}/peering-profile', 'POST /mpp/capacity/sessions/{sessionId}/connection-confirmed', 'POST /mpp/capacity/sessions/{sessionId}/cancel']
source: >-
  The provider's spec declares NO operationIds, so steps cite method + path verbatim from
  openapi/aureliusagent-dev-walton-capacity-mpp-api-openapi.yml; the names in parentheses are the ones
  overlays/aureliusagent-dev-walton-capacity-mpp-api-overlay.yaml assigns and are not the provider's. GET
  /mpp/capacity and /mpp/capacity/offers were fetched live on 2026-09-19; the flow narrative follows the
  provider's llms.txt "Safe capacity marketplace" section and /.well-known/mpp.json capacity block.
---

# Quote, pay for and hand off a capacity session

Walton "coordinates payment and direct handoff metadata without relaying provider capacity traffic": you pay through MPP, then buyer and provider exchange public peering details and connect directly (WIREGUARD, SSH, TAILSCALE, HTTPS_API or MANUAL).

## Auth
- Offers and quotes are free and anonymous. Paying for a session is the MPP 402 → retry cycle; the `201` returns `Payment-Receipt` and `X-Capacity-Access-Token` headers, and that token (also `CapacitySession.accessToken`) is the required header on every later session call. Provider-side routes (`POST /capacity/assets`, `POST /capacity/sessions/{sessionId}/peering-offer`) are `bearerAuth` and not for buyers. See `authentication/aureliusagent-dev-authentication.yml`.

## Steps
1. **Browse offers** — `GET /mpp/capacity/offers` (`listCapacityOffers`, free). Returns `{generatedAt, assets:[CapacityOffer]}` across `GPU_CAPACITY`, `MODEL_INFERENCE`, `EDGE_COMPUTE`, `SHARED_WORKSPACE`, `SERVICE_ROBOT`, `THREE_D_PRINTER` (`GET /mpp/capacity` lists the enabled asset types; transport, freight, delivery and drone profiles are disabled by default). Note `mppEnabled`, `amountCents`, `depositCents` and `expiresAt` on the offer you want.
2. **Take a quote** — `POST /mpp/capacity/quotes` (`createCapacityQuote`, free) with a `CapacityQuoteRequest` for that offer. The `CapacityQuote` carries `amountCents`, `depositCents`, `platformFeeCents`, `ownerPayoutCents`, `currencyCode`, `paymentRail` and an `expiresAt`; quote before it expires.
3. **Pay and reserve** — `POST /mpp/capacity/sessions` (`createCapacitySession`) with `{ "quoteId": "<quote id>" }` (the only required field of `CapacitySessionRequest`). "First request may return HTTP 402. Retry the identical request with the MPP Authorization credential." Expect **201** with the `CapacitySession` body, `Payment-Receipt` and `X-Capacity-Access-Token`.
4. **Publish your peering profile** — `POST /mpp/capacity/sessions/{sessionId}/peering-profile` (`updateBuyerPeeringProfile`) with a `BuyerPeeringProfile`, header `X-Capacity-Access-Token`. Poll `GET /mpp/capacity/sessions/{sessionId}` (`getCapacitySession`) until `peering` (a `PeeringExchange`) carries the provider's offer and `handoffUrl` / `resourceAccess` / `gpuAccess` are populated.
5. **Confirm the connection** — once connected directly, `POST /mpp/capacity/sessions/{sessionId}/connection-confirmed` (`confirmCapacityConnection`).
6. **Cancel if needed** — `POST /mpp/capacity/sessions/{sessionId}/cancel` (`cancelCapacitySession`) is allowed only while `CapacitySession.cancellable` is `true`. No cancellation window or refund rule is published; `refundable_deposit` is listed among pricing models but its terms are not written anywhere (`conventions/aureliusagent-dev-conventions.yml`, reversibility grade `documented`).

## Errors
- `402 MPP payment challenge` on session creation is expected; anything else is undocumented in this spec (no error schema is declared). The origin answers `403 {"error":"Origin is not allowed."}` for unknown routes. See `errors/aureliusagent-dev-problem-types.yml`.

## Notes
- `GET /mpp/capacity` reported `paymentNetwork: tempo-testnet` on 2026-09-19 - the marketplace's payment rail is a test network.
- Base URL `https://api.walton.bot` (spec `servers[]`); the identical spec and routes are served from `mpp.openmodel.sh` and `rpc.aureliusagent.dev`.
