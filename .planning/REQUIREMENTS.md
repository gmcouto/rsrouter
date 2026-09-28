# Requirements: rsrouter

**Defined:** 2026-09-28
**Core Value:** A client tool pointed at rsrouter with a single client API key can pick any model or combo, in either OpenAI or Anthropic format, and get a transparent, cache-friendly response from the first healthy upstream key — without the client ever noticing fallbacks, rate limits or exhausted keys.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Foundation

- [ ] **FOUND-01**: Operator can run rsrouter as a single binary that serves the AI API and the admin UI/API on two separately configurable ports
- [ ] **FOUND-02**: On startup, the SQLite database is created if missing and all pending schema migrations are applied automatically (embedded in the binary)
- [ ] **FOUND-03**: Operator can configure listen addresses, data path and log/retention defaults via environment variables or config file
- [ ] **FOUND-04**: Admin port rejects cross-origin state-changing requests (CSRF protection) and requests with non-allowlisted Host headers
- [ ] **FOUND-05**: Configuration changes made via the admin UI take effect on the AI API without restart and without per-request database reads
- [ ] **FOUND-06**: Upstream secrets (API keys, OAuth tokens) never appear in logs, traces or UI responses beyond a masked form
- [ ] **FOUND-07**: Architecture exposes swappable components behind traits (provider adapter, upstream codec, routing strategy, cooldown tracker, quota prober, proxy transport, log sink, storage)

### Client API Keys

- [ ] **KEYS-01**: User can create a named client API key (e.g. `opencode`) and sees the generated secret exactly once
- [ ] **KEYS-02**: User can list, rename, disable and delete client API keys
- [ ] **KEYS-03**: rsrouter records first-used and last-used timestamps per client key (including `/models` calls)
- [ ] **KEYS-04**: AI API accepts client keys via `Authorization: Bearer` and `x-api-key` headers and rejects unknown/disabled keys with a format-appropriate error

### Providers, Credentials & Models

- [ ] **PROV-01**: User can register a custom provider with a prefix, base URL(s) and upstream format(s) (OpenAI-like and/or Anthropic-like)
- [ ] **PROV-02**: A provider (built-in or custom) can declare multiple endpoints per format/modality, so a model can be served via the endpoint matching the client's format
- [ ] **PROV-03**: User can add multiple upstream credentials (API keys or OAuth accounts) per provider, each with a nickname and a priority order
- [ ] **PROV-04**: User can trigger model discovery via the provider's `/models` endpoint using a specific credential; discovered models are registered to that credential
- [ ] **PROV-05**: User can manually register custom models on a credential, including which endpoint/format each model uses
- [ ] **PROV-06**: User can enable/disable each model per credential
- [ ] **PROV-07**: User can test a model on a specific credential (tiny request) and see success, latency, or the error
- [ ] **PROV-08**: User can enable/disable an entire credential or provider

### Client-Facing API

- [ ] **API-01**: Client can call OpenAI-compatible `POST /v1/chat/completions` (streaming and non-streaming) for any exposed model or combo
- [ ] **API-02**: Client can call Anthropic-compatible `POST /v1/messages` (streaming and non-streaming, `?beta=true` accepted) for any exposed model or combo
- [ ] **API-03**: Client can call `POST /v1/messages/count_tokens` (proxied to Anthropic-format upstreams, local estimate otherwise)
- [ ] **API-04**: Client can call OpenAI Responses-compatible `POST /v1/responses` (streaming and non-streaming) for any exposed model or combo
- [ ] **API-05**: Client can call `GET /v1/models` and receives every enabled `provider-prefix/model` and every combo name, in OpenAI shape or Anthropic shape depending on request headers
- [ ] **API-06**: Errors are returned in the client's API format (OpenAI error object vs Anthropic error object / SSE error event)
- [ ] **API-07**: Request bodies larger than framework defaults (large contexts, images) are accepted up to a configurable limit

### Transparency & Caching

- [ ] **CACHE-01**: When client format equals upstream format, the request body is forwarded byte-stable except for the fields routing strictly requires (model, auth) — key order and unknown fields preserved
- [ ] **CACHE-02**: Anthropic `cache_control` markers, thinking blocks and signatures pass through untouched on same-format routes
- [ ] **CACHE-03**: For OpenAI-client → Anthropic-upstream routes without client cache hints, rsrouter enables Anthropic automatic prompt caching via top-level `cache_control` only
- [ ] **CACHE-04**: rsrouter fingerprints requests by client key + stable prefix (explicit session id when present, otherwise hash of tools + system + first user message, format-independent) and maps it to the resolved upstream credential + model
- [ ] **CACHE-05**: Follow-up requests with a matching fingerprint reuse the same upstream credential + model until it is exhausted, cooled down or disabled
- [ ] **CACHE-06**: Usage (input, output, cache-read, cache-write tokens) is normalized across formats and recorded per request

### Routing & Combos

- [ ] **ROUTE-01**: User can create a named combo as an ordered list of items; each item is a `provider/model` with an optional pinned credential
- [ ] **ROUTE-02**: A combo item without a pinned credential expands to all of that provider's credentials that have the model enabled, tried in credential priority order (horizontal), before advancing to the next combo item (vertical)
- [ ] **ROUTE-03**: Credentials that have the model disabled, are disabled, or are in cooldown/terminal state are skipped (and the skip is traced)
- [ ] **ROUTE-04**: Requesting a direct `provider-prefix/model` (not a combo) applies the same horizontal credential fallback within that provider
- [ ] **ROUTE-05**: The client connection is held open until the first candidate produces real content; fallback happens only before the first content byte reaches the client (commit point); after commit, failures surface as a format-appropriate error event
- [ ] **ROUTE-06**: An error classifier maps upstream responses (status + body signals) to stop / advance / cooldown / terminal — client-caused errors (malformed request) stop without penalizing credentials
- [ ] **ROUTE-07**: Each request has an attempt budget and a total wall-clock budget, including a configurable first-content timeout
- [ ] **ROUTE-08**: Routing strategy is a pluggable trait; v1 ships sequential fallback only

### Rate-Limit Protection & Quota

- [ ] **LIMIT-01**: A credential+model enters cooldown on 429/quota errors, honoring `Retry-After` and provider reset headers/bodies
- [ ] **LIMIT-02**: A credential+model enters a short cooldown after repeated 5xx responses or timeouts
- [ ] **LIMIT-03**: Terminal states (auth invalid, banned, credits exhausted) disable the credential until user action or successful re-auth, and are never overwritten by transient cooldowns
- [ ] **LIMIT-04**: User can set proactive per-credential limits (RPM, TPM, max concurrency); rsrouter skips the credential before exceeding them
- [ ] **LIMIT-05**: rsrouter parses passive rate-limit headers from upstream responses to update remaining-quota state
- [ ] **LIMIT-06**: For providers with quota probing, rsrouter periodically probes quota and the UI shows remaining quota and reset time per credential
- [ ] **LIMIT-07**: A credential whose probed quota is below a configurable threshold is skipped
- [ ] **LIMIT-08**: Cooldown expiry uses half-open probing (one request tests recovery) to avoid thundering herds

### OAuth Infrastructure

- [ ] **OAUTH-01**: For OAuth providers, the UI shows the authorization URL for the user to open in their own browser, and the user pastes back the callback URL / code / token
- [ ] **OAUTH-02**: Device-code and poll flows are supported where the provider offers them (UI shows code + URL and completes automatically)
- [ ] **OAUTH-03**: OAuth tokens refresh automatically before expiry, with at most one concurrent refresh per credential and atomic persistence of rotated refresh tokens
- [ ] **OAUTH-04**: A failed/revoked refresh marks the credential as needing re-auth and is visible in the UI
- [ ] **OAUTH-05**: Before adding the first credential of a subscription provider, the user must acknowledge the ToS/account-ban risk for that provider

### Built-in Providers

- [ ] **PROVB-01**: OpenRouter provider (API key, OpenAI + Anthropic endpoints, model discovery)
- [ ] **PROVB-02**: DeepSeek provider (API key, OpenAI + Anthropic endpoints, balance probe)
- [ ] **PROVB-03**: Z.ai provider (API key, OpenAI + Anthropic endpoints, quota probe)
- [ ] **PROVB-04**: NVIDIA provider (API key, OpenAI endpoint, discovery, rerank)
- [ ] **PROVB-05**: Cloudflare Workers AI provider (API token + account id, OpenAI-compatible endpoint)
- [ ] **PROVB-06**: FreeInference provider
- [ ] **PROVB-07**: Claude Code provider (Anthropic OAuth paste-code flow, minimum required wire contract, usage probe)
- [ ] **PROVB-08**: GitHub Copilot provider (device flow, token exchange, chat/messages/responses endpoints, quota probe)
- [ ] **PROVB-09**: Codex provider (ChatGPT OAuth device/paste flow, Responses upstream codec, usage probe)
- [ ] **PROVB-10**: Antigravity provider (Google OAuth paste flow, Gemini v1internal upstream codec, project onboarding, model/quota probe)
- [ ] **PROVB-11**: Cursor provider (OAuth/poll login, ConnectRPC protobuf upstream codec)

### Translation

- [ ] **XLAT-01**: OpenAI chat requests can be served by Anthropic-format upstreams and vice versa, including streaming
- [ ] **XLAT-02**: Tool definitions, tool calls, tool results and tool_choice translate correctly in both directions (streaming and non-streaming)
- [ ] **XLAT-03**: Image inputs, system prompts, stop sequences, temperature/top_p/max tokens and JSON/structured output translate where the target supports them
- [ ] **XLAT-04**: Reasoning/thinking content and stop reasons map between formats without fabricating or corrupting signatures
- [ ] **XLAT-05**: Chat/messages/responses clients can be served by Responses-only (Codex), Gemini v1internal (Antigravity) and ConnectRPC (Cursor) upstreams via upstream codecs
- [ ] **XLAT-06**: `/v1/responses` client requests can be served by chat-format and Anthropic-format upstreams
- [ ] **XLAT-07**: When a provider offers an endpoint in the client's format, rsrouter prefers it (passthrough) over translation

### Other Modalities

- [ ] **MODAL-01**: Client can call `POST /v1/embeddings` routed to OpenAI-shape embedding upstreams (models and combos)
- [ ] **MODAL-02**: Client can call `POST /v1/rerank` (Cohere/Jina shape) routed to rerank-capable upstreams
- [ ] **MODAL-03**: Client can call `POST /v1/images/generations` routed to image-capable upstreams
- [ ] **MODAL-04**: Client can call `POST /v1/audio/speech` and receives streamed audio from speech-capable upstreams
- [ ] **MODAL-05**: Client can create and poll video generation jobs (`/v1/videos`), with each job sticky to the credential that created it

### Proxies

- [ ] **PROXY-01**: User can add HTTP/HTTPS and SOCKS5 proxies (with optional auth) to a proxy pool
- [ ] **PROXY-02**: User can assign a proxy to an upstream credential; that credential's traffic (including OAuth refresh and probes) always egresses through it and fails closed if the proxy is down
- [ ] **PROXY-03**: rsrouter periodically health-checks proxies and marks unhealthy ones (after N conclusive failures) in the UI
- [ ] **PROXY-04**: SOCKS5 proxies resolve DNS remotely; environment proxy variables are never silently applied

### Observability

- [ ] **LOG-01**: Every request is logged with client key, requested model/combo, each attempt (credential, model, endpoint, proxy, outcome, skip/cooldown reason), final route, latency, TTFT and usage
- [ ] **LOG-02**: User can browse and filter request logs in the UI and open a request to see its route trace and chat data (request/response bodies)
- [ ] **LOG-03**: User can watch a live tail of requests in the UI (SSE)
- [ ] **LOG-04**: User can toggle whether request/response bodies are stored, and bodies are size-capped
- [ ] **LOG-05**: Logs are pruned automatically by configurable age and/or size
- [ ] **LOG-06**: Logging never blocks or slows request handling (write-behind)

### Admin UI

- [ ] **UI-01**: Server-rendered Rust admin UI (no auth, separate port) with a polished, responsive look for managing providers, credentials, models, combos, client keys, proxies, quota and logs
- [ ] **UI-02**: User can build and reorder combos (items and optional pinned credentials) in the UI
- [ ] **UI-03**: User can reorder credential priority per provider in the UI
- [ ] **UI-04**: Dashboard shows credential health (active / cooldown until / terminal / needs re-auth) at a glance
- [ ] **UI-05**: UI shows copy-paste setup snippets for opencode and Claude Code (base URL + key)

### Operations & Packaging

- [ ] **OPS-01**: Release builds produce static binaries for linux amd64 and arm64
- [ ] **OPS-02**: A multi-arch (amd64/arm64) Docker image runs as non-root, with a sample `compose.yaml`
- [ ] **OPS-03**: Graceful shutdown drains in-flight streams up to a timeout
- [ ] **OPS-04**: Idle and under-load memory usage is measured and reported (no hard gate)
- [ ] **OPS-05**: Documentation covers setup, reverse proxy/SSE buffering, firewalling the admin port, and ToS risk of subscription providers

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Security

- **SEC-01**: Upstream credentials encrypted at rest with a key from env/file

### Routing

- **ROUTE2-01**: Additional combo strategies (round-robin, weighted, fastest, hedging)

### API

- **API2-01**: Gemini-native client endpoint (`/v1beta/models/*:generateContent`)
- **API2-02**: Image output via chat (modalities)

## Out of Scope

| Feature | Reason |
|---------|--------|
| Built-in admin authentication | Admin-only UI on a separate port; protected by firewall / reverse-proxy SSO |
| Mid-stream failover | Cannot splice two models' outputs without corrupting the stream; fallback only before commit |
| Fingerprint/TLS cloaking arms race | Maintenance treadmill, ToS-adjacent; only minimum wire contract per provider |
| Prompt compression, tool renaming, system-prompt JSON injection | Violates transparency and breaks prompt caching |
| Semantic response cache | Wrong answers for "similar" prompts; upstream prompt caching covers the cost win |
| Anthropic-format non-chat modalities | Anthropic has no embeddings/image/audio/video APIs; non-chat is OpenAI-shape only |
| Billing, credits, multi-tenant accounts | Not competing with OpenRouter |
| Web-session (cookie/browser) providers | Fragile, heavy, clearly against ToS |
| MCP server, A2A, playground, agent memory | Unrelated to routing |
| Opening OAuth URLs server-side / loopback callback server | Headless deployments; user opens URLs in their own browser |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|

**Coverage:**
- v1 requirements: 96 total
- Mapped to phases: 0
- Unmapped: 96 ⚠️

---
*Requirements defined: 2026-09-28*
*Last updated: 2026-09-28 after initial definition*
