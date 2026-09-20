---
generated: '2026-09-19'
method: generated
name: Buy a Wundership app plan through an MPP 402 challenge
description: Discover the paid endpoint, receive the HTTP 402 Machine Payments Protocol challenge, settle it, and retry the identical request to receive a structured software plan (then, optionally, a BuilderStudio preview).
api: openapi/aureliusagent-dev-wundership-mpp-api-openapi.yml
operations: [postWundershipPlan, postBuilderPreview]
source: >-
  operationIds verified in openapi/aureliusagent-dev-wundership-mpp-api-openapi.yml; the 402 shape was
  observed live on https://mpp.openmodel.sh/v1/plan on 2026-09-19; prices from
  https://mpp.openmodel.sh/agent-products.json.
---

# Buy a Wundership app plan through an MPP 402 challenge

Turn a product idea into an implementation-ready plan for USD 1.00, paid per call with the Machine Payments Protocol. There is no API key: the first request is refused with a priced challenge, and the same request succeeds once the challenge is paid.

## Auth
- None up front. Payment is the authorization: the `mppPayment` security scheme is `http` with scheme `Payment`. See `authentication/aureliusagent-dev-authentication.yml`.
- Settle challenges with an MPP-capable client or a proxy such as the provider's own ArgentShell (`argent.sh`, the `proxyUrl` in `/.well-known/mpp.json`). Never retry blindly: every retry that succeeds is a purchase (`conventions/aureliusagent-dev-conventions.yml`, idempotency `coverage: none`).

## Steps
1. **Discover** — `GET https://mpp.openmodel.sh/.well-known/mpp.json` (free). Read `endpoints[]` for `/v1/plan` and its `payment.offers[]` (method `stripe`, intent `charge`, amount `100` usd). `agent-products.json` carries the same product under key `wundership_plan` with `successorPath: /v1/plan`; ignore the `legacyPaths` (`/mpp/plan`, `/mpp/aurelius/plan`), which are deprecated and returned HTTP 500 when probed.
2. **Request the plan (unpaid)** — `postWundershipPlan` (`POST /v1/plan`) with JSON `{ "prompt": "<idea or task>" }`; optional `project_name`, `preferred_stack`, `target_customer`, `output_format` (`implementation_plan` | `product_outline` | `feature_map`). Expect **402**: `WWW-Authenticate: Payment id="<uuid>", realm="mpp.openmodel.sh", method="stripe", intent="charge", request="<base64 {amount,currency}>"`, `X-Wundership-Agent-Price: 1.00 USD`, `Link: </v1/plan>; rel="payment"`, and a `PaymentRequired` body whose `requestId` equals the challenge id.
3. **Pay** — hand the challenge to your MPP client; it returns an MPP `Authorization` credential (or, through a verifying proxy, the `X-Wundership-Agent-Payment-Verified` / `X-Wundership-Agent-Payment-Receipt` headers named in the 402 body's `payment` object).
4. **Retry the identical request** — same body, plus the credential. Expect **200** with `ok: true`, `paymentVerified: true`, a `receipt`, and `result.productBrief`, `result.buildPhases[]`, `result.architecture[]`, `result.risks[]` and `result.recommendedNextPaidCall`. Keep the receipt.
5. **Optional: preview** — if `recommendedNextPaidCall` points at it, `postBuilderPreview` (`POST /v1/builder/preview`, USD 5.00) follows the same 402 → pay → retry cycle and returns a BuilderStudio preview package.

## Errors
- `402` is the designed first answer, not a failure. `502 Paid forwarding failed` (`ErrorResponse {error, message, fallback}`) means payment was taken but fulfilment failed; no refund path is published, so retain the receipt and escalate through the `humanFallback` URLs in the 402 body. `403 {"error":"Origin is not allowed."}` is the origin's not-found shape. See `errors/aureliusagent-dev-problem-types.yml`.

## Notes
- No rate limits are published or signalled (`rate-limits/aureliusagent-dev-rate-limits.yml`).
- Prices: `plans/aureliusagent-dev-plans-pricing.yml`. The identical API is served from `rpc.aureliusagent.dev`, `api.wundership.com` and `api.walton.bot`; the realm in the challenge is always `mpp.openmodel.sh`.
