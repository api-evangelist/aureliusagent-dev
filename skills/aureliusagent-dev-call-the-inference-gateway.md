---
generated: '2026-09-19'
method: generated
name: Call the OpenModel MPP inference gateway
description: Pick a provider and model from the free discovery routes, send an OpenAI- or Anthropic-shaped request, and let the gateway relay the upstream provider's HTTP 402 challenge and receipt unchanged.
api: openapi/aureliusagent-dev-wundership-mpp-api-openapi.yml
operations: [getMppInferenceCapabilities, getMppInferenceProviders, getMppInferenceModels, postMppChatCompletions, postMppMessages, getProvidersByProviderByUpstreamPath]
source: >-
  operationIds verified in openapi/aureliusagent-dev-wundership-mpp-api-openapi.yml; GET /v1/capabilities,
  /v1/providers and /v1/models fetched live on 2026-09-19; unpaid POST /v1/chat/completions observed to
  return 402 application/problem+json.
---

# Call the OpenModel MPP inference gateway

Use one base URL for many model providers. The gateway is a "transparent upstream MPP relay": it adds no surcharge and needs no gateway key, and the model provider's own machine-payment challenge is what you pay.

## Auth
- No gateway credential (`gatewayApiKeyRequired: false`). The upstream provider's 402 `WWW-Authenticate` challenge is passed through; retry the SAME gateway URL with the MPP `Authorization` credential and the upstream `Authentication-Info` / receipt headers come back unchanged. See `authentication/aureliusagent-dev-authentication.yml`.

## Steps
1. **Read capabilities** — `getMppInferenceCapabilities` (`GET /v1/capabilities`, free). Confirm `enabled: true`, note `operationFamilies[]` and `providerSelection` (`header: X-OpenModel-Provider`, `jsonField: provider`, `modelPrefix: provider/model or provider:model`, `defaultProvider: openrouter`).
2. **Choose a provider and model** — `getMppInferenceProviders` (`GET /v1/providers`) and `getMppInferenceModels` (`GET /v1/models`), both free, unauthenticated arrays (`{object:"list", data:[...]}`). Use a provider `id` and a provider-prefixed model id such as `openai/gpt-4o`.
3. **Send the request** — OpenAI-shaped: `postMppChatCompletions` (`POST /v1/chat/completions`) with `{ "model": "<provider/model>", "messages": [...] }`; Anthropic-shaped: `postMppMessages` (`POST /v1/messages`). Select the provider with the model prefix, a top-level `provider` field, or the `X-OpenModel-Provider` header. Siblings follow the same pattern: `postMppResponses`, `postMppEmbeddings`, `postMppImageGeneration`, `postMppAudioTranscription`, `postMppAudioSpeech`, `postMppModeration`.
4. **Handle the challenge** — expect **402** from the upstream provider (observed as `application/problem+json`) with a `WWW-Authenticate` challenge. Pay it with your MPP client and retry the identical gateway request. `200` is the upstream response body; `202` means the upstream accepted an asynchronous operation. Streaming responses are passed through (`streamingPassthrough: true`).
5. **Provider-native paths** — for anything outside the unified shapes use `relayMppProviderRequest` (`POST|GET /providers/{provider}/{upstreamPath}`); `upstreamPath` may contain several segments and the body, credential, challenge and receipt are preserved.

## Errors
- `400` unknown provider or invalid gateway request (choose from `/v1/providers`); `402` upstream payment required; `502` upstream provider unavailable (retry or switch provider); `503` gateway or provider route disabled (re-check `/v1/capabilities`). See `errors/aureliusagent-dev-problem-types.yml`.

## Notes
- No idempotency key exists and every successful retry is billed by the upstream provider (`conventions/aureliusagent-dev-conventions.yml`).
- The legacy fixed-price `postChatCompletion` (`POST /mpp/chat/completions`, USD 0.50) is deprecated; do not use it (`lifecycle/aureliusagent-dev-lifecycle.yml`).
- No rate limits are published (`rate-limits/aureliusagent-dev-rate-limits.yml`).
