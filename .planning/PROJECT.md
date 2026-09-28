# rsrouter

## What This Is

rsrouter is a lean, simplified re-implementation of [OmniRoute](https://github.com/diegosouzapw/OmniRoute) in Rust: a self-hosted AI router that sits between AI client tools (opencode, Claude Code, etc.) and many upstream AI providers. It exposes every registered model and every named combo through **both** OpenAI-compatible and Anthropic-compatible APIs, routes each request through upstream API keys (and optional proxies) with automatic fallback and cooldowns, and provides a beautiful server-rendered Rust admin UI to manage everything. It is a public open-source project meant for individuals/small groups — not an OpenRouter competitor.

## Core Value

A client tool pointed at rsrouter with a single client API key can pick any model or combo, in either OpenAI or Anthropic format, and get a transparent, cache-friendly response from the first healthy upstream key — without the client ever noticing fallbacks, rate limits or exhausted keys.

## Requirements

### Validated

(None yet — ship to validate)

### Active

**Routing core**
- [ ] Combos: named, ordered lists of models; v1 strategy is sequential fallback (try next on error/exhaustion/cooldown); client socket is held open until the first model replies. Strategy is pluggable so round-robin/weighted/hedging/etc. can be added later
- [ ] Providers: built-in and user-registered custom providers (OpenAI-like or Anthropic-like upstream API)
- [ ] Multiple upstream API keys (credentials) per provider
- [ ] Model registration, testing, and enable/disable are **per upstream API key**, not per provider
- [ ] Model discovery via the provider's `/models` endpoint using a specific upstream API key
- [ ] Custom models per provider, each able to target different endpoints/formats
- [ ] Multiple named client API keys (e.g. `opencode` → generated key); router records first/last use; keys may be shared with others
- [ ] `/models` listing exposes `provider-prefix/model` and `combo-name` entries to clients

**API surface & translation**
- [ ] Endpoints: Chat, Embeddings, Rerank, Images, Video, Audio Speech — OpenAI-like and Anthropic-like where applicable
- [ ] Every model reachable in both OpenAI and Anthropic formats: 1:1 passthrough when formats match; translation where it reasonably fits (chat/messages both ways incl. streaming, tools, thinking; route Anthropic-chat-based modalities to backend endpoints where sensible); completely incompatible combos are out of scope (research determines the matrix)
- [ ] Minimal prompt modification — client and upstream talk as transparently as possible; only change what routing/translation strictly requires
- [ ] Practices that maximize upstream prompt caching (stable prefixes, preserve cache_control, sticky keys)

**Cache affinity**
- [ ] Fingerprint recent requests by (client API key + stable prefix hash: system prompt + first N messages) and map to the resolved upstream key + model; reuse that same upstream key/model for follow-up turns until it is exhausted/cooled down

**Rate-limit protection & quota**
- [ ] Cooldowns per upstream key (+model) triggered by: 429/quota errors (honor Retry-After/reset headers), proactive user-configured RPM/TPM/concurrency limits, low probed quota, repeated 5xx/timeouts
- [ ] Quota information displayed for providers that implement quota probing

**Proxies**
- [ ] Proxy pool: HTTP/HTTPS and SOCKS5 proxies with optional auth
- [ ] Proxy health checks, dead proxies removed from rotation
- [ ] Assign upstream API keys to proxies

**Providers (day-1, ported from OmniRoute's implementations)**
- [ ] Claude Code, Codex, Antigravity, Copilot, Cursor (OAuth/subscription)
- [ ] FreeInference, NVIDIA, Cloudflare, OpenRouter, DeepSeek, Z.ai
- [ ] OAuth login flows like OmniRoute's, but rsrouter shows the auth URL for the user to open in their own logged-in browser; the user pastes back the callback URL or token; tokens auto-refresh

**Observability**
- [ ] Interactive logs showing the route taken per request (client key, combo, each attempt/skip/cooldown, final upstream key/model/proxy) and chat data (request/response bodies)
- [ ] Configurable log retention (days/size pruning) and toggle for storing bodies

**UI & operations**
- [ ] Beautiful server-rendered Rust admin UI (Axum + templates + htmx, SSE for live logs) managing everything
- [ ] Admin UI + UI API served on a **separate port** from the AI API, so the UI can be firewalled while the AI API stays open
- [ ] SQLite storage with automated migrations on startup
- [ ] Ships as a single binary (embedded assets) and a multi-arch (amd64/arm64) Docker image with sample compose

### Out of Scope

- Built-in admin authentication — UI is expected to be admin-only, protected by network/firewall or reverse proxy SSO (e.g. Pangolin); separate port makes this easy
- Combo strategies other than sequential fallback in v1 — architecture leaves room, implementation deferred
- Translating between fundamentally incompatible modalities/formats — only where a sensible mapping exists
- Billing, multi-tenant accounts, public marketplace features — not competing with OpenRouter
- Opening OAuth URLs from the server's browser — user opens them in their own browser

## Context

- Reference implementation: OmniRoute (https://github.com/diegosouzapw/OmniRoute) — navigate its codebase to understand provider implementations (auth flows, headers, quirks, quota probing) and port them, but design a better, cleaner architecture.
- Target usage flow:
  1. User creates client key `opencode` → rsrouter generates key `abc`.
  2. User registers provider `rsrouter` in opencode with key `abc`; opencode calls `/models`; rsrouter records the key's use.
  3. opencode lists `provider-prefix/model` and `combo-name` entries; user picks a combo.
  4. Prompt sent with `abc` → rsrouter identifies `opencode`, logs combo, tries each combo model in order, skipping cooled-down ones, holding the socket until the first succeeds.
  5. Same flow works from Claude Code via the Anthropic-compatible endpoints.
- Author's homelab: mixed amd64/arm64 servers, Docker, Pangolin reverse proxy with SSO — a natural first deployment target.
- Author is not a provider-API expert; research must determine the format/modality compatibility matrix.

## Constraints

- **Tech stack**: Rust for everything, including the UI (server-rendered) — author's explicit choice
- **Database**: SQLite with automated migrations — simple self-hosting, zero ops
- **Resource usage**: As lean as possible, low memory footprint, while handling multi-threaded, highly concurrent (streaming) AI API requests well — self-hosted on small machines (e.g. 1GB VMs, Raspberry Pi)
- **Architecture**: Loosely coupled, components swappable (providers, translators, strategies, storage, proxy transport) — long-lived public project
- **Transparency**: Minimal prompt/body modification between client and upstream — preserves client behavior and upstream caching
- **Portability**: amd64 + arm64

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Sequential fallback as sole v1 combo strategy, behind a strategy trait | Simplest correct behavior; extensible later | — Pending |
| Server-rendered UI (Axum + templates + htmx + SSE) | Single binary, no WASM build, lean | — Pending |
| No built-in UI auth; UI on separate port | Admin-only; firewall/SSO handles protection | — Pending |
| Models, testing, enable/disable scoped per upstream key | Different keys/plans expose different models | — Pending |
| OAuth via copy-URL / paste-callback | Works headless/remote; mirrors OmniRoute | — Pending |
| Cache affinity via client key + stable prefix hash | Aligns with how upstream prompt caching works | — Pending |
| Port provider logic from OmniRoute | Proven quirks/auth handling from day 1 | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-09-28 after initialization*
