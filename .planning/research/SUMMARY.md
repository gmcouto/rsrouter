# Project Research Summary

**Project:** rsrouter
**Domain:** Self-hosted AI API router/gateway (OpenAI + Anthropic compatible ingress, multi-provider/multi-key egress, combos with fallback), Rust single binary
**Researched:** 2026-09-28
**Confidence:** MEDIUM-HIGH

> **⚠ USER OVERRIDE (2026-09-28):** The research recommendations against "fingerprint cloaking" and in favor of a "minimum wire contract only" are **superseded**. OAuth/subscription providers (and any provider with an official client tool) MUST mimic the official tool's requests to be indistinguishable from it: User-Agent, headers, magic bytes, identity blocks, client IDs, OAuth params, and transport fingerprint where feasible. Only prompt content may differ. See AGENTS.md and PROJECT.md (Provider fidelity).

## Executive Summary

rsrouter is a lean Rust re-implementation of OmniRoute: a self-hosted gateway that lets tools like opencode and Claude Code reach many upstream providers through one client key, in either OpenAI or Anthropic format, with transparent fallback across keys and models. Every mature gateway researched (OmniRoute, CLIProxyAPI, new-api, Helicone, LiteLLM) has the same six parts: format-specific ingress handlers, a translation layer, per-provider adapters, key selection with cooldowns and refresh, combo/fallback orchestration, and persistence plus logs. OmniRoute's weaknesses are architectural: 6,000-line god handlers, a lossy OpenAI-chat hub format, and several overlapping cooldown layers. The research is clear on this point: **port OmniRoute's wire contracts (URLs, headers, OAuth parameters, quota endpoints, error signals), not its code shape.**

Recommended approach: tokio + axum 0.8 (two routers on two ports) + reqwest 0.13 (one `Client` per proxy in an ArcSwap map) + SQLite via sqlx 0.9 (WAL, split writer/reader pools, embedded migrations) + askama/htmx/Tailwind-standalone UI embedded with rust-embed. Ship it as musl static binaries built with cargo-zigbuild on distroless. The pipeline is **passthrough-first** (byte-stable bodies when client and upstream dialects match, which preserves prompt caching) with a **canonical IR only for cross-dialect translation**. Around that sits a **prime-then-commit streaming protocol**: the router reads upstream until the first content-bearing event, then commits to the client. Fallback only happens before the commit point. An immutable config snapshot (ArcSwap) keeps DB reads off the request path, and logging is write-behind.

Key risks: (1) fallback after bytes reached the client, which corrupts streams; (2) the router silently destroying upstream prompt caching through body rewrites or a bad affinity fingerprint; (3) translation bugs in tools, thinking signatures, and usage accounting that poison conversations; (4) error misclassification that causes cooldown storms or burns quota across every combo target; (5) OAuth refresh races with rotating refresh tokens that revoke whole token families. Subscription providers also carry ToS and ban risk that users must acknowledge. Day-1 OAuth providers need three extra upstream codecs (OpenAI Responses for Codex, Gemini v1internal for Antigravity, ConnectRPC protobuf for Cursor). The PROJECT assumption that upstreams are "OpenAI-like or Anthropic-like" is therefore not enough, and Cursor should move out of v1.

## Key Findings

### Recommended Stack

All versions were checked against the crates.io registry on 2026-09-28, and critical feature flags against crate source (HIGH). There is one runtime and one HTTP implementation (hyper 1) for both server and client. There are no C/OpenSSL dependencies beyond aws-lc via zig. See STACK.md for the full Cargo.toml.

**Core technologies:**
- **Rust 1.98.1 stable, MSRV 1.94, edition 2024**: pinned in `rust-toolchain.toml`.
- **tokio 1.53 + axum 0.8.9 + tower-http 0.7.1**: two listeners in one process. tower-http 0.7 adds `CsrfLayer`, which the admin port needs because it has no auth by design.
- **reqwest 0.13.5 (rustls/aws-lc, http2, socks, stream)**: proxies are per-Client, so keep one Client per proxy and share one rustls `ClientConfig`.
- **sqlx 0.9 SQLite (bundled)**: WAL, one writer pool plus a read pool, and `migrate!()` embedded.
- **askama 0.16 + askama_web, htmx 2 + htmx-ext-sse, Tailwind 4 standalone + daisyUI 5, rust-embed 8**: server-rendered UI with no Node toolchain.
- **tracing** for diagnostics. Domain request logs go through an explicit `mpsc` to a batched SQLite writer, plus a `broadcast` for live SSE.
- **mimalloc** (not jemalloc, which breaks on Pi 5 16K-page kernels). Background jobs are plain tokio loops with CancellationToken. OAuth/PKCE is hand-rolled (the `oauth2` crate pulls stale deps).
- **Builds**: cargo-zigbuild to x86_64/aarch64 musl, `distroless/static:nonroot`. No compiling under QEMU.

### Expected Features

**Must have (table stakes):**
- `/v1/chat/completions`, `/v1/messages` (+ `count_tokens`, `?beta=true`, both `x-api-key` and Bearer auth), and `/v1/models`, with the response shape chosen by request headers
- OpenAI<->Anthropic chat translation, including streaming, tools, images-in, thinking, usage, and errors
- Providers with multiple credentials each. Models are scoped, discovered, tested, and enabled **per credential**
- Custom OpenAI-like and Anthropic-like providers
- Combos with sequential fallback and commit-point semantics
- Per-(credential, model) cooldowns that honor Retry-After and reset headers, plus terminal states (banned, credits exhausted, auth invalid)
- Hashed named client keys with last-used tracking
- Request log with route trace, body-capture toggle, and retention
- Admin UI on a separate port
- Credential test ("ping") button, token usage normalization, and embeddings passthrough
- Multi-arch Docker image and single binary

**Should have (differentiators):**
- **Transparent passthrough-first routing.** This is the thesis: no cloaking, no prompt injection, no tool renames
- **Format-aware endpoint selection.** DeepSeek, Z.ai, OpenRouter, and Copilot expose both formats, so picking the matching one makes the route a passthrough
- **Correct cache affinity.** Use explicit session keys first (Claude Code session id, OpenAI `prompt_cache_key`). Otherwise hash (tools + system + first user message) after stripping `x-anthropic-billing-header`
- **Paste-back, device, and poll OAuth** for headless servers
- First-class route trace UI over live SSE
- Lean footprint (runs on a 1 GB VM or a Pi)
- Proxy pool with fail-closed per-key assignment
- Quota probing, starting with passive header parsing
- Proactive RPM/TPM/concurrency limits
- **O->A caching:** add top-level `cache_control` when the client sent none. It is one field and gives near-100% cache reads after turn 1

**Defer (v1.x / v2+):**
- v1.x: Antigravity (Gemini codec + loopback OAuth problem), quota probing UI, proactive limits, rerank/images/speech passthrough, a `/v1/responses` client endpoint, round-robin strategy
- v2+: **Cursor** (about 3,400 lines in OmniRoute, ConnectRPC protobuf, most fragile), video (async job stickiness), image output via chat, Gemini-native client endpoint
- Never: fingerprint-cloaking arms race, extra combo strategies, semantic cache, prompt compression, mid-stream failover, billing/multi-tenant, web-session providers, MCP/A2A

**Compatibility matrix conclusion:** the Anthropic-like surface is **chat only** (messages, count_tokens, models). Every non-chat modality is OpenAI-shape passthrough (Cohere-shape for rerank) with no cross-format translation.

### Architecture Approach

This is a Cargo workspace with crates for core types, store, transport, translate, providers, and engine, plus a binary. The central patterns are:
1. **Declarative `ProviderSpec` plus behavioral `Adapter` hooks** (`Arc<dyn Adapter>`, async_trait). Quirks live in adapters, not in the shared translator.
2. **Hybrid translation.** Same dialect uses a raw passthrough that only rewrites model and auth. Different dialects go through a canonical IR with a lossless enough design for thinking signatures and cache_control. Upstream-only codecs cover Responses, Gemini, and ConnectRPC.
3. **Prime-then-commit streaming.** The router uses a bounded peek (capped at about 1 MiB) until the first content event, then replays the buffered prefix.
4. **Immutable config snapshot (ArcSwap), small mutable runtime state (cooldowns, token cache), and write-behind persistence.**
5. **One task per request with poll-driven streams and RAII finalization** of the log record.
6. **The strategy only orders candidates.** The attempt loop and commit logic belong to the engine.
7. **Explicit failure taxonomy** (advance / stop / cooldown / terminal). Caller-caused failures never count against upstream keys, and a transient cooldown never overwrites a terminal state.

**Major components:**
1. **api**: ingress per client dialect, client-key auth, and error dialect rendering
2. **translate**: pure IR plus codecs and stream aggregator, tested against golden fixtures
3. **providers**: adapters, discovery descriptors, OAuth flows, and quota probers
4. **engine**: resolver, strategy, attempt loop, cooldown tracker, affinity, and token cache (single-flight refresh)
5. **transport**: per-proxy reqwest client cache that fails closed
6. **store**: SQLite repos, migrations, and LogSink
7. **admin**: control plane plus askama/htmx UI on the second port

### Critical Pitfalls

1. **Fallback after the stream commit point.** Fail over only before the first content-bearing event, after translation. After commit, emit an error event. Test with fake upstreams that return a pre-content 429, a mid-stream error, and a stall.
2. **Destroying upstream prompt caching.** Keep bodies byte-stable on passthrough and preserve `cache_control`. Strip the rotating billing header only on A->non-A routes. Build the affinity fingerprint on a format-independent view. Verify with a cache regression test (turn 2+ cache reads > 0) and golden diffs of outbound bodies.
3. **Thinking-signature and tool-call corruption.** Pass signature-only blocks through untouched. Strip-and-retry once. Sanitize tool IDs deterministically. Merge and split tool results correctly and drop orphans. Use round-trip property tests.
4. **Error misclassification and quota-burning fan-out.** A malformed client 400 must stop, while model-unsupported or context-overflow must advance. Port OmniRoute's `statusDecisionTable.ts`. Parse reset hints robustly (z.ai uses local time). Use half-open probes to avoid thundering herds. Budget both attempts and wall-clock time.
5. **OAuth refresh races.** Use a per-credential async mutex (single-flight) with atomic persistence of rotated tokens, and design it in before any OAuth provider ships. Test that 50 concurrent requests with an expired token produce exactly 1 refresh.

Also important: SSE buffering, idle timeouts, and disconnect handling (use `read_timeout`, not a total timeout). SQLite BUSY under logging load. Memory blowups from buffered bodies on a 1 GB VM. Secrets at rest and header leakage. An unauthenticated admin UI is exposed to cross-origin POSTs (use CsrfLayer and a Host allowlist). Proxies can leak DNS (normalize to `socks5h`), fail open to a direct connection, or inherit env proxies.

## Implications for Roadmap

### Phase 1: Foundation & Skeleton
**Rationale:** Everything else depends on it.
**Delivers:** Workspace and crates, core types and traits, SQLite schema v1 with embedded migrations (WAL, split pools), config snapshot via ArcSwap, two listeners with graceful shutdown, CsrfLayer and Host allowlist on the admin port, encrypted/secret-safe credential storage, HTTP client factory (per-proxy cache, shared rustls config, no env proxies), and client API keys (hashed, last-used).
**Addresses:** T10, D10
**Avoids:** Pitfalls 12, 14, 16

### Phase 2: Same-Format Passthrough Proxy
**Rationale:** This validates the streaming mechanics (the hardest runtime piece) with no translation in the way.
**Delivers:** Generic openai_compat and anthropic_compat adapters, `/v1/chat/completions` and `/v1/messages` passthrough (stream and non-stream), SSE framing with a tap, header policy, error dialects, body limits, `/v1/models` in both shapes, write-behind LogSink, and a cache regression test.
**Addresses:** T1, T2, T3, T6, D1
**Avoids:** Pitfalls 2, 11, 13

### Phase 3: Multi-Credential Routing Core
**Rationale:** This is the core value. Fallback is easiest to verify with same-format traffic, so it comes before translation.
**Delivers:** Per-credential model scoping, resolver, sequential strategy behind a trait, combos, prime-then-commit, failure taxonomy and classifier, CooldownTracker (Retry-After/reset parsing, terminal states, half-open), attempt/skip route-trace data model, and cache affinity. Affinity is small and central, so pull it in here.
**Addresses:** T5, T8, T9, T12 (data model), D2
**Avoids:** Pitfalls 1, 6, 7, 8
**Test:** wiremock upstreams that return 429, 500, and in-band errors, and that stall.

### Phase 4: Admin UI (minimal) + Live Logs
**Rationale:** The router must be configurable without SQL before provider work multiplies.
**Delivers:** askama/htmx pages for providers, credentials, models (discover/test/enable), combos, and client keys; a live SSE log tail with route trace; and a body-logging toggle.
**Addresses:** T7, T12, T13, T14, D6

### Phase 5: OpenAI<->Anthropic Translation IR
**Rationale:** This is the largest bug surface and deserves its own phase. The crate is pure, so it can be developed in parallel with Phases 3-4.
**Delivers:** Canonical IR, request/response/SSE codecs both ways, tools, thinking (plus a signature cache), usage and stop-reason mapping, `cache_control` handling (top-level injection for O->A), format-aware endpoint selection, and a fixture corpus captured from real opencode and Claude Code traffic.
**Addresses:** T4, T15, D3
**Avoids:** Pitfalls 3, 4, 5

### Phase 6: OAuth Infrastructure + API-Key Providers
**Rationale:** The cheap providers exercise the whole router, and the token lifecycle must exist before any OAuth provider.
**Delivers:** TokenCache (single-flight refresh, rotation persistence), paste-back/device/poll OAuth UI, discovery descriptors, and the providers NVIDIA, OpenRouter, DeepSeek (both endpoints), Z.ai (both endpoints), Cloudflare, and FreeInference.
**Addresses:** T7, T11 (infra), D5
**Avoids:** Pitfall 9

### Phase 7: Subscription OAuth Providers
**Rationale:** Order these by increasing difficulty. Each provider is effectively a mini-project with ToS risk.
**Delivers:** Claude Code (Anthropic dialect, paste code), then Copilot (device flow, three endpoints), then Codex (device flow + Responses upstream codec). Add a stored risk acknowledgement and configurable identity versions.
**Addresses:** T11, D5
**Avoids:** Pitfall 10

### Phase 8: Protection, Quota & Proxies
**Rationale:** These features enhance the working router. Probers are per provider, so they depend on Phases 6-7.
**Delivers:** Passive rate-limit header parsing, quota probers and quota UI, proactive RPM/TPM/concurrency limits, and a proxy pool (HTTP/HTTPS/SOCKS5 with auth, health checks, fail-closed assignment per credential).
**Addresses:** D4, D8, D9
**Avoids:** Pitfall 15

### Phase 9: Other Modalities
**Rationale:** These are OpenAI-shape passthrough only and depend on the router, not on the translator.
**Delivers:** Embeddings, rerank (Cohere shape, plus a NVIDIA adapter), images, audio speech, and `count_tokens` (passthrough, or a local estimate).
**Addresses:** T16

### Phase 10: Operations & Release
**Rationale:** This can overlap with anything after Phase 4. It is last only for polish.
**Delivers:** Retention/pruning UI, cargo-zigbuild musl multi-arch builds, a distroless Docker image, sample compose (including a Pangolin SSO example), and reverse-proxy SSE docs.
**Avoids:** Pitfalls 11 (proxy buffering docs) and 16

### Phase 11 (v1.x, optional): Antigravity
Gemini v1internal codec, project onboarding, and a paste-back workaround for OAuth loopback. Cursor stays in v2.

### Phase Ordering Rationale

- Foundation and passthrough come first because nearly every pitfall depends on byte-faithful streaming, body limits, and the caching regression test being right.
- Routing comes before translation because commit-point and classifier semantics are verifiable with same-format traffic. Translation is then layered on without confusing the two bug classes.
- The route-log data model is born in Phase 3, so observability grows incrementally instead of being retrofitted.
- OAuth comes late: it is the most volatile area, it carries ban risk, and it needs the codec and IR work first.

### Research Flags

Phases likely needing `/gsd-plan-phase --research-phase`:
- **Phase 3:** priming timeout defaults per provider type, and building the classifier's fixture corpus from real error bodies.
- **Phase 5:** IR design spike and fixture capture from real clients. Thinking-signature replay behavior needs validation.
- **Phase 7:** per-provider porting from OmniRoute source (Codex Responses codec, Copilot token exchange, Claude OAuth `metadata.user_id` format). The ToS landscape shifted repeatedly in 2026.
- **Phase 8:** per-provider quota endpoints and header semantics (NVIDIA limits are LOW confidence).
- **Phase 11:** Antigravity paste-flow viability (MEDIUM confidence).

Standard patterns (skip research): Phases 1, 2, 4, 9, 10.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Versions come from crates.io, and key features were checked in crate source. UI and build pipeline details are MEDIUM. |
| Features | HIGH | OmniRoute source was read directly at commit 8e3b486, and Anthropic docs were official. Competitor features and some provider limits are MEDIUM/LOW. |
| Architecture | HIGH | Boundaries and the commit protocol were cross-checked in four gateways' source. IR shape and memory numbers are MEDIUM (reasoned, not measured). |
| Pitfalls | MEDIUM-HIGH | Mostly primary-source issue trackers. Subscription ToS and ban behavior is MEDIUM and still moving. |

**Overall confidence:** MEDIUM-HIGH

### Gaps to Address

- **Scope conflict with PROJECT.md:** PROJECT lists Cursor and Antigravity as day-1, but the research recommends moving Cursor to v2 and Antigravity to v1.x. Confirm with the user during requirements.
- **PROJECT's "OpenAI-like or Anthropic-like upstream" model is too narrow:** an `UpstreamCodec` abstraction is required for Responses, Gemini, and ConnectRPC.
- **Admin UI with no auth:** CSRF and Host allowlist are mandatory mitigations. Document the Pangolin/SSO deployment.
- **Memory footprint claims** (1 GB VM / Pi) are unmeasured. Add a cgroup load test in Phase 2 or 10.
- **Thinking-signature replay** for OpenAI clients on Anthropic upstreams is degraded. Validate how strictly Anthropic enforces it.
- **Timeout defaults** for priming and idle reads per provider are not yet specified.

## Sources

### Primary (HIGH confidence)
- crates.io registry API and published `.crate` sources: reqwest, sqlx, axum, tower-http, libmimalloc-sys (versions and features)
- OmniRoute source (commit `8e3b486`, release/v3.8.51): translator, executors, combo, tokenRefresh, OAuth, and migrations
- CLIProxyAPI, new-api, and Helicone ai-gateway source (translator, conductor, adapter, and endpoint layouts)
- Anthropic official API docs: messages, streaming, prompt caching, thinking, output_config, errors
- Issue trackers for OmniRoute, LiteLLM, and CLIProxyAPI (root-caused failure modes)

### Secondary (MEDIUM confidence)
- LiteLLM, TensorZero, and 9router feature sets (web search plus docs)
- Subscription provider ToS and ban behavior (2026 policy changes)

### Tertiary (LOW confidence)
- NVIDIA rate limits and FreeInference details

---
*Research completed: 2026-09-28*
*Ready for roadmap: yes*
