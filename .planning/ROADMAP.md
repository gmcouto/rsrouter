# Roadmap: rsrouter

## Overview

rsrouter is built as a series of working slices. It starts as a single Rust binary with an admin UI and client keys. Next it becomes a byte-stable passthrough proxy for custom OpenAI-like and Anthropic-like providers. The routing core follows in two steps: first combos and credential fallback before the commit point, then cooldowns, credential health and cache affinity. Every route is then made observable in live logs. OpenAI Chat, Anthropic Messages and OpenAI Responses clients can reach any chat model through translation, and non-chat modalities are added. The phases after that add the built-in API-key providers with quota visibility, then OAuth infrastructure together with the Claude Code, Copilot and Codex subscription providers, then proactive limits and a proxy pool. Next the project is packaged for release. The two most fragile providers come last: Antigravity, which starts with a paste-back OAuth spike, and then Cursor. The routing core is already verifiable before any exotic provider or codec is added.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation, Admin Shell & Client Keys** - Single binary, two ports, SQLite migrations, hot config snapshot, CSRF-protected admin UI shell, client API keys
- [ ] **Phase 2: Custom Providers & Same-Format Passthrough** - Custom providers, credentials, per-credential models, and byte-stable chat/messages passthrough for opencode and Claude Code
- [ ] **Phase 3: Fallback & Combos** - Combos, horizontal/vertical credential fallback before the commit point, error classifier, attempt/time budgets, routing-strategy trait
- [ ] **Phase 4: Cooldowns, Health & Affinity** - Cooldowns with half-open recovery, terminal states, credential health dashboard, cache-affinity pinning
- [ ] **Phase 5: Request Logs & Live Route Trace** - Write-behind request logs with route traces, bodies, normalized usage, live SSE tail, retention
- [ ] **Phase 6: Cross-Format Translation & Responses API** - OpenAI Chat, Anthropic Messages and OpenAI Responses clients reach any chat model; passthrough preferred, faithful translation otherwise
- [ ] **Phase 7: Other Modalities** - Embeddings, rerank, image generation, speech and sticky video jobs routed like chat
- [ ] **Phase 8: Built-in API-Key Providers & Quota** - OpenRouter, DeepSeek, Z.ai, NVIDIA, Cloudflare, FreeInference plus passive/probed quota that routing respects
- [ ] **Phase 9: OAuth & Subscription Providers** - Paste-back/device OAuth with single-flight refresh, ToS acknowledgement, Claude Code, Copilot and Codex
- [ ] **Phase 10: Proactive Limits & Proxy Pool** - Per-credential RPM/TPM/concurrency limits and fail-closed, health-checked HTTP/SOCKS5 proxies
- [ ] **Phase 11: Operations & Release** - Static multi-arch binaries, non-root Docker image, graceful drain, memory report, docs
- [ ] **Phase 12: Antigravity** - Paste-back OAuth spike, then Antigravity via the Gemini v1internal upstream codec
- [ ] **Phase 13: Cursor** - Cursor login and ConnectRPC upstream codec, completing every exotic-codec route

## Phase Details

### Phase 1: Foundation, Admin Shell & Client Keys
**Goal**: An operator can run rsrouter as one binary, manage client API keys in a polished admin UI on its own port, and the AI API accepts or rejects those keys immediately
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: FOUND-01, FOUND-02, FOUND-03, FOUND-04, FOUND-05, FOUND-07, KEYS-01, KEYS-02, KEYS-03, KEYS-04, UI-01, UI-05
**Success Criteria** (what must be TRUE):
  1. The operator starts the single binary, configured through environment variables or a config file. The AI API and the admin UI come up on two separately configured ports. A fresh data path gets a new SQLite database with all embedded migrations applied, and existing data survives a restart.
  2. Opening the admin port in a browser shows a polished, responsive admin shell with navigation for every management area. The admin port rejects cross-origin state-changing requests and requests whose Host header is not on the allowlist.
  3. The user creates a named client key (e.g. `opencode`) and sees its secret exactly once, next to copy-paste setup snippets for opencode and Claude Code. The user can list, rename, disable and delete keys.
  4. A newly created key works right away, with no restart, for `GET /v1/models` via either `Authorization: Bearer` or `x-api-key`, and its first-used and last-used timestamps update in the UI. Unknown or disabled keys are rejected with an OpenAI-shaped or Anthropic-shaped error, and the request path never reads the database per request.
  5. Each swappable component (provider adapter, upstream codec, routing strategy, cooldown tracker, quota prober, proxy transport, log sink, storage) sits behind a trait, and a test can swap in a stub implementation for any of them.
**Plans**: TBD
**UI hint**: yes

### Phase 2: Custom Providers & Same-Format Passthrough
**Goal**: A user can register a custom OpenAI-like or Anthropic-like provider, add credentials, and discover, test and enable models per credential. opencode and Claude Code can then chat through rsrouter with byte-stable, cache-preserving passthrough.
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: PROV-01, PROV-02, PROV-03, PROV-04, PROV-05, PROV-06, PROV-07, PROV-08, FOUND-06, API-01, API-02, API-05, API-06, API-07, CACHE-01, CACHE-02
**Success Criteria** (what must be TRUE):
  1. The user registers a custom provider with a prefix, base URL(s) and endpoints per format/modality, then adds several nicknamed credentials in priority order. Credential secrets only ever appear masked in the UI, logs and traces.
  2. The user discovers models through a specific credential's `/models` endpoint and manually adds custom models with their endpoint/format. Testing a model on a credential shows success and latency, or the upstream error. The user can enable or disable individual models per credential, whole credentials and whole providers.
  3. `GET /v1/models` returns every enabled `provider-prefix/model` in OpenAI shape or Anthropic shape, depending on the request headers. Disabled models, credentials and providers disappear from the list.
  4. opencode (`/v1/chat/completions`) and Claude Code (`/v1/messages`, including `?beta=true`) complete streaming and non-streaming conversations against same-format upstreams. Requests with large contexts or images are accepted up to the configured body limit, and every error reaches the client in the client's own API format.
  5. On same-format routes the outbound body matches the client's byte for byte except for model and auth. Key order, unknown fields, `cache_control`, thinking blocks and signatures are preserved, and a multi-turn conversation shows upstream cache reads from turn 2 onward.
**Plans**: TBD
**UI hint**: yes

### Phase 3: Fallback & Combos
**Goal**: Requests to any model or combo fall back across credentials and combo items before the first content byte, so the client never notices a failed candidate, and client-caused errors come straight back without burning other candidates
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: ROUTE-01, ROUTE-02, ROUTE-03, ROUTE-04, ROUTE-05, ROUTE-06, ROUTE-07, ROUTE-08, UI-02, UI-03
**Success Criteria** (what must be TRUE):
  1. In the UI, the user builds and reorders a named combo of `provider/model` items, each optionally pinned to a credential, and reorders credential priority per provider. The combo appears in `/v1/models`, and opencode and Claude Code can request it by name.
  2. Each unpinned combo item expands across that provider's credentials that have the model enabled, in priority order, before the next item is tried. A pinned item uses only its pinned credential. A direct `provider-prefix/model` request falls back across that provider's credentials the same way. This is sequential fallback, implemented behind a routing-strategy trait.
  3. If a candidate fails before producing content (429, 5xx, timeout, or a stall past the first-content timeout), the client gets a normal response from the next candidate and never sees the failure. If a failure happens after content has started, it arrives as an error event in the client's format. Every request stops at its attempt budget and its wall-clock budget.
  4. A malformed client request goes back to the client right away, without trying other candidates and without penalizing any credential. Errors such as model unavailable, context overflow and rate limits advance to the next candidate. The classifier also labels rate-limit, repeated-failure, auth, ban and credit errors as cooldown or terminal outcomes and records that label in the route trace; the cooldown tracker that acts on these labels arrives in Phase 4.
  5. Candidates are skipped, with the reason recorded in the route trace, when the credential has the model disabled or is disabled itself. The candidate filter also asks the cooldown-tracker trait about each credential+model, so a stub tracker that reports a cooldown or terminal state causes that candidate to be skipped.
**Plans**: TBD
**UI hint**: yes

### Phase 4: Cooldowns, Health & Affinity
**Goal**: Rate-limited or broken credentials step aside and recover safely, the user can see every credential's health at a glance, and conversations stay pinned to the same upstream so prompt caching keeps working
**Mode:** mvp
**Depends on**: Phase 3
**Requirements**: LIMIT-01, LIMIT-02, LIMIT-03, LIMIT-08, CACHE-04, CACHE-05, UI-04
**Success Criteria** (what must be TRUE):
  1. A credential+model that returned 429 or a quota error is not used again until its `Retry-After` or provider reset time (from headers or body), and requests meanwhile go to the next candidate. Repeated 5xx responses or timeouts trigger a short cooldown.
  2. When a cooldown expires, exactly one half-open request tests recovery while concurrent requests keep skipping that credential+model. Success returns it to active, and failure puts it back into cooldown.
  3. Credentials with invalid auth, a ban or exhausted credits enter a terminal state and stay disabled until the user acts. A later transient cooldown never overwrites a terminal state.
  4. The dashboard shows each credential's state (active / cooldown until / terminal) at a glance, and the state changes as cooldowns start and expire.
  5. Follow-up turns of a conversation reuse the same credential and model until that pairing is exhausted, cooled down or disabled, and then re-pin to the next healthy candidate. A turn counts as a follow-up when it has the same client key plus the same session id, or the same stable prefix hash (tools + system + first user message) in either client format.
**Plans**: TBD
**UI hint**: yes

### Phase 5: Request Logs & Live Route Trace
**Goal**: Users can see exactly how every request was routed, live and historically, with chat data and normalized token usage, and logging never slows requests down
**Mode:** mvp
**Depends on**: Phase 4
**Requirements**: LOG-01, LOG-02, LOG-03, LOG-04, LOG-05, LOG-06, CACHE-06
**Success Criteria** (what must be TRUE):
  1. Every request is recorded with its client key and the requested model or combo. The record also lists each attempt (credential, model, endpoint, proxy, outcome, skip/cooldown reason), plus the final route, latency, TTFT and normalized usage (input, output, cache-read and cache-write tokens), whatever the client or upstream format.
  2. The user browses and filters request logs in the UI and opens any request to see its route trace and its request/response bodies.
  3. The user watches new requests appear live in a tail view (SSE) as they are routed.
  4. The user can turn body storage on or off, and stored bodies are size-capped. Logs are pruned automatically by the configured age and/or size.
  5. Under sustained streaming load, logging does not measurably affect request latency or throughput, even when the database is busy.
**Plans**: TBD
**UI hint**: yes

### Phase 6: Cross-Format Translation & Responses API
**Goal**: Any chat model can be reached from OpenAI Chat, Anthropic Messages and OpenAI Responses clients: passthrough when the provider offers the client's format, faithful translation otherwise
**Mode:** mvp
**Depends on**: Phase 4
**Requirements**: XLAT-01, XLAT-02, XLAT-03, XLAT-04, XLAT-06, XLAT-07, CACHE-03, API-03, API-04
**Success Criteria** (what must be TRUE):
  1. Claude Code works end-to-end against an OpenAI-format-only model, and opencode works against an Anthropic-format-only model. This holds for streaming and non-streaming requests, including multi-turn tool use (definitions, calls, results, tool_choice).
  2. Images, system prompts, stop sequences, sampling parameters, max tokens and JSON/structured output are translated wherever the target supports them. Reasoning/thinking content and stop reasons map across formats, and no signature is ever fabricated or corrupted.
  3. When an OpenAI client with no cache hints is routed to an Anthropic upstream, the upstream request has automatic prompt caching enabled through top-level `cache_control` only, and later turns show cache reads.
  4. A client using `POST /v1/responses`, streaming or non-streaming, can use any chat model or combo served by chat-format or Anthropic-format upstreams. `POST /v1/messages/count_tokens` returns upstream counts for Anthropic-format models and a local estimate for other models.
  5. If a provider exposes an endpoint in the client's format, rsrouter uses it as passthrough instead of translating, even inside a combo that mixes formats.
**Plans**: TBD

### Phase 7: Other Modalities
**Goal**: Clients can use embeddings, rerank, image generation, speech and video through rsrouter with the same keys, models, combos and fallback as chat
**Mode:** mvp
**Depends on**: Phase 4
**Requirements**: MODAL-01, MODAL-02, MODAL-03, MODAL-04, MODAL-05
**Success Criteria** (what must be TRUE):
  1. A client calls `POST /v1/embeddings` on a model or combo and receives OpenAI-shape embeddings, with fallback across credentials.
  2. A client calls `POST /v1/rerank` (Cohere/Jina shape) and receives ranked results from a rerank-capable upstream.
  3. A client calls `POST /v1/images/generations` and receives generated images. A client calls `POST /v1/audio/speech` and receives audio streamed back.
  4. A client creates a video job via `/v1/videos` and polls it to completion, and every poll for that job goes to the credential that created it.
**Plans**: TBD

### Phase 8: Built-in API-Key Providers & Quota
**Goal**: Users can add OpenRouter, DeepSeek, Z.ai, NVIDIA, Cloudflare Workers AI and FreeInference with nothing more than a key, and see live remaining quota that routing respects
**Mode:** mvp
**Depends on**: Phase 6, Phase 7
**Requirements**: PROVB-01, PROVB-02, PROVB-03, PROVB-04, PROVB-05, PROVB-06, LIMIT-05, LIMIT-06, LIMIT-07
**Success Criteria** (what must be TRUE):
  1. The user adds a credential for any of OpenRouter, DeepSeek, Z.ai, NVIDIA, Cloudflare Workers AI (API token + account id) or FreeInference from a built-in preset, discovers or selects models, and chats with them from both opencode and Claude Code.
  2. For OpenRouter, DeepSeek and Z.ai, requests go to the provider endpoint that matches the client's format (passthrough). NVIDIA rerank models work through `/v1/rerank`.
  3. rsrouter updates each credential's remaining quota from upstream rate-limit headers and periodically probes quota where the provider supports it (DeepSeek balance, Z.ai quota). The UI shows remaining quota and reset time per credential.
  4. Routing skips any credential whose probed quota is below the configured threshold until the quota recovers.
**Plans**: TBD
**UI hint**: yes

### Phase 9: OAuth & Subscription Providers
**Goal**: Users can connect Claude Code, GitHub Copilot and Codex subscription accounts from a headless server using paste-back or device flows. Tokens refresh safely, and each provider's ToS risk is acknowledged explicitly.
**Mode:** mvp
**Depends on**: Phase 8
**Requirements**: OAUTH-01, OAUTH-02, OAUTH-03, OAUTH-04, OAUTH-05, OAUTH-06, PROVB-07, PROVB-08, PROVB-09
**Success Criteria** (what must be TRUE):
  1. Before adding the first Claude Code, Copilot or Codex credential, the user must acknowledge that provider's ToS/account-ban risk.
  2. The user connects a Claude Code account by opening the displayed authorization URL in their own browser and pasting back the callback URL or code. Copilot and Codex accounts connect through a device code and URL shown in the UI, and the flow completes automatically.
  3. OAuth tokens refresh automatically before expiry: 50 concurrent requests with an expired token trigger exactly one refresh, and the rotated refresh token survives a restart. A revoked or failed refresh marks the credential "needs re-auth" in the UI.
  4. opencode and Claude Code can chat with models from Claude Code, Copilot (chat/messages/responses endpoints) and Codex (Responses upstream codec), and each provider's usage/quota shows in the quota view.
  5. Captured outbound requests (chat, token refresh, quota probe, model discovery) for Claude Code, Copilot and Codex match the official tool's traffic — User-Agent, headers, identity blocks/markers, client IDs, OAuth parameters — differing only in prompt content; the fidelity profile (tool versions) is configurable without a rebuild.
**Plans**: TBD
**UI hint**: yes

### Phase 10: Proactive Limits & Proxy Pool
**Goal**: Users can keep each credential under its limits before upstreams complain, and send each credential's traffic through a dedicated, health-checked proxy that never leaks
**Mode:** mvp
**Depends on**: Phase 9
**Requirements**: LIMIT-04, PROXY-01, PROXY-02, PROXY-03, PROXY-04
**Success Criteria** (what must be TRUE):
  1. The user sets RPM, TPM and max-concurrency limits on a credential, and rsrouter routes to the next candidate rather than exceed them.
  2. The user adds HTTP/HTTPS and SOCKS5 proxies (optional auth) to a pool and assigns one to a credential. All of that credential's traffic, including OAuth refreshes and quota probes, egresses through the assigned proxy. If the proxy is down, the credential is skipped and never falls back to a direct connection.
  3. The UI shows each proxy's health, and a proxy is marked unhealthy after N conclusive failed checks.
  4. SOCKS5 proxies resolve DNS on the proxy side. `HTTP_PROXY`, `HTTPS_PROXY` and `ALL_PROXY` environment variables never affect upstream traffic.
**Plans**: TBD
**UI hint**: yes

### Phase 11: Operations & Release
**Goal**: rsrouter ships as a documented, release-ready static binary and multi-arch Docker image that deploys cleanly on small amd64 and arm64 hosts
**Mode:** mvp
**Depends on**: Phase 10
**Requirements**: OPS-01, OPS-02, OPS-03, OPS-04, OPS-05
**Success Criteria** (what must be TRUE):
  1. Release builds produce static linux amd64 and arm64 binaries. A multi-arch Docker image runs as non-root from the sample `compose.yaml` on both architectures.
  2. On SIGTERM, in-flight streams keep going until they finish or the configured drain timeout expires, and then the process exits.
  3. Idle and under-load memory usage are measured on amd64 and arm64, and the results are published.
  4. The documentation covers setup, reverse-proxy SSE buffering, firewalling the admin port, and the ToS risk of subscription providers.
**Plans**: TBD

### Phase 12: Antigravity
**Goal**: Once a spike confirms the paste-back OAuth flow is viable, users can connect Google Antigravity accounts and use Antigravity models from any client format
**Mode:** mvp
**Depends on**: Phase 9, Phase 11
**Requirements**: PROVB-10
**Success Criteria** (what must be TRUE):
  1. Before any provider implementation starts, a spike either confirms that Antigravity's Google OAuth can complete by pasting back the callback URL with no loopback server, or refutes it and documents an alternative.
  2. The user connects an Antigravity account via paste-back, and rsrouter completes project onboarding automatically.
  3. opencode, Claude Code and Responses clients can chat with Antigravity models through the Gemini v1internal upstream codec, including streaming and tool use.
  4. Antigravity models are discovered per credential, and their quota is probed and displayed per credential.
  5. Captured outbound Antigravity traffic (OAuth, onboarding, chat, probes) matches the official Antigravity client's requests (User-Agent, headers, client metadata, session IDs), differing only in prompt content (OAUTH-06).
**Plans**: TBD

### Phase 13: Cursor
**Goal**: Users can log into Cursor and use Cursor models from any client format, which completes every day-1 provider and every exotic upstream codec route
**Mode:** mvp
**Depends on**: Phase 12
**Requirements**: PROVB-11, XLAT-05
**Success Criteria** (what must be TRUE):
  1. The user logs into Cursor through its OAuth/poll login flow, and the credential's tokens refresh automatically.
  2. opencode, Claude Code and Responses clients can chat with Cursor models through the ConnectRPC protobuf upstream codec, including streaming and tool use.
  3. Every exotic upstream (Codex Responses, Antigravity Gemini v1internal, Cursor ConnectRPC) serves `/v1/chat/completions`, `/v1/messages` and `/v1/responses` clients.
  4. Captured outbound Cursor traffic (login, refresh, ConnectRPC calls) matches the official Cursor client's requests (User-Agent, headers, checksums/magic bytes, client version), differing only in prompt content (OAUTH-06).
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10 → 11 → 12 → 13

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation, Admin Shell & Client Keys | 0/TBD | Not started | - |
| 2. Custom Providers & Same-Format Passthrough | 0/TBD | Not started | - |
| 3. Fallback & Combos | 0/TBD | Not started | - |
| 4. Cooldowns, Health & Affinity | 0/TBD | Not started | - |
| 5. Request Logs & Live Route Trace | 0/TBD | Not started | - |
| 6. Cross-Format Translation & Responses API | 0/TBD | Not started | - |
| 7. Other Modalities | 0/TBD | Not started | - |
| 8. Built-in API-Key Providers & Quota | 0/TBD | Not started | - |
| 9. OAuth & Subscription Providers | 0/TBD | Not started | - |
| 10. Proactive Limits & Proxy Pool | 0/TBD | Not started | - |
| 11. Operations & Release | 0/TBD | Not started | - |
| 12. Antigravity | 0/TBD | Not started | - |
| 13. Cursor | 0/TBD | Not started | - |
