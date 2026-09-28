# Feature Research

**Domain:** Self-hosted AI API router/gateway (lean Rust re-implementation of OmniRoute)
**Researched:** 2026-09-28
**Confidence:** HIGH for OmniRoute implementation details (read directly from source at commit `8e3b486`, branch `release/v3.8.51`, 2026-09-28). MEDIUM for competitor feature sets (web search plus prior knowledge). HIGH for Anthropic API facts (official docs fetched). MEDIUM/LOW where marked.

> **How to read this file.** Sections 1-5 follow the standard template: table stakes, differentiators, anti-features, dependencies, MVP. Section 6 maps OmniRoute's implementation so it can be ported, with file paths. Section 7 has per-provider porting notes. Section 8 is the endpoint/format compatibility matrix. Every OmniRoute path below exists in the repo; each one was checked with a script after cloning.
>
> OmniRoute paths are relative to the repo root (`https://github.com/diegosouzapw/OmniRoute`). `open-sse/` is the streaming engine, `src/` is the Next.js app, DB and OAuth routes.

---

## 0. Scale warning: OmniRoute is huge. Port behavior, not structure

OmniRoute at the researched commit is about 23,800 files, 256 provider registry entries (`open-sse/config/providers/registry/`), 19 combo strategies, about 190 SQLite migrations, 110 MCP tools, A2A, skills, memory, MITM, Electron and more. Several core files are "god files": `open-sse/executors/base.ts`, `open-sse/services/accountFallback.ts` (2,516 lines), `open-sse/executors/cursor.ts` (1,846), `open-sse/executors/antigravity.ts` (1,693) and `open-sse/executors/codex.ts` (1,583). Much of that bulk is **anti-detection fingerprint mimicry** and fixes for issues found at 500+ connections.

rsrouter should port the **wire contracts** (URLs, headers, body fields, OAuth parameters, quota endpoints, error signals). It should not port the code shape. Where OmniRoute has several layers of the same idea (connection cooldown + model lockout + provider circuit breaker + combo breaker), rsrouter should keep **one cooldown model keyed by (upstream key, model)** plus a coarse per-key state.

---

## 1. Feature Landscape

### 1.1 Table Stakes (users expect these; missing = product feels broken)

| # | Feature | Why Expected | Complexity | Notes |
|---|---------|--------------|------------|-------|
| T1 | **OpenAI-compatible `/v1/chat/completions`** (stream + non-stream) | Every router (LiteLLM, OmniRoute, 9router, new-api, OpenRouter) has it. It is what opencode, Continue, Cline and aider speak | MEDIUM | SSE with `data: [DONE]`. Honor `stream_options.include_usage` |
| T2 | **Anthropic-compatible `/v1/messages`** (+ `/v1/messages/count_tokens`) | Claude Code only speaks this. OmniRoute, CLIProxyAPI, claude-code-router and OpenRouter ("Anthropic skin") all expose it | MEDIUM | Accept `?beta=true` query. Accept auth via `x-api-key` **and** `Authorization: Bearer` (Claude Code sends Bearer when `ANTHROPIC_AUTH_TOKEN` is set). `count_tokens` can proxy to Anthropic upstreams or fall back to an estimate (OmniRoute: `src/app/api/v1/messages/count_tokens/route.ts`) |
| T3 | **`/v1/models` listing** with `provider-prefix/model` and combo names | Clients (opencode) populate model pickers from it | LOW | Return OpenAI shape `{object:"list",data:[{id,object:"model",owned_by}]}`. When the request carries `anthropic-version`/`x-api-key`, return the Anthropic shape `{data:[{type:"model",id,display_name,created_at}],has_more,first_id,last_id}` |
| T4 | **Chat translation OpenAI⇄Anthropic** incl. streaming, tools, images-in, thinking | The core promise of "any model in both formats". OmniRoute, 9router, CLIProxyAPI and LiteLLM all do this | HIGH | See Section 8. This is the largest correctness surface. Build it as pure functions with golden-file tests |
| T5 | **Providers + multiple upstream credentials per provider** | Multi-account is why people run these routers (CLIProxyAPI and 9router round-robin accounts) | MEDIUM | OmniRoute: `provider_connections` table (`src/lib/db/migrations/001_initial_schema.sql`) |
| T6 | **Custom OpenAI-like / Anthropic-like providers** (base URL + key) | Every competitor supports "any OpenAI-compatible endpoint" | LOW | OmniRoute: `provider_nodes` table (prefix, api_type, base_url) |
| T7 | **Model discovery via upstream `/models`**, per credential | Users do not want to type model IDs. Entitlements differ per account (Copilot, Codex, Antigravity return per-account catalogs) | MEDIUM | Needs a per-provider discovery descriptor (URL, method, headers, parser). OmniRoute: `src/app/api/providers/[id]/models/discovery/providerModelsConfig.ts` |
| T8 | **Combos with sequential fallback** | Called "combos" in OmniRoute and 9router, "fallbacks" in LiteLLM, the `models[]` array in OpenRouter. Universal | MEDIUM | Hold the client socket until the first target commits. See the "commit point" rule in 6.2 |
| T9 | **Per-credential cooldown on 429/5xx, honoring `Retry-After` / reset headers** | LiteLLM `allowed_fails` + cooldown. OmniRoute connection cooldown. Without it, fallback thrashes | MEDIUM | See 6.5 for signal lists worth porting |
| T10 | **Client API keys** (named, generated, revocable, last-used) | LiteLLM virtual keys, new-api tokens, OmniRoute `api_keys` | LOW | Store a hash, show the key once. PROJECT wants first/last use |
| T11 | **OAuth subscription providers with auto-refresh** (Claude Code, Codex, Copilot) | This is why OmniRoute, 9router and CLIProxyAPI exist. Users come for subscription reuse | HIGH | Refresh must be single-flight per credential (rotating refresh tokens; see Pitfalls) |
| T12 | **Request log with route taken** (client key, combo, attempts, final upstream) | OmniRoute call_logs + combo decision trace. LiteLLM spend logs. new-api logs | MEDIUM | Bodies optional and togglable. Retention pruning |
| T13 | **Admin UI** to manage providers/keys/combos/logs | OmniRoute, 9router, new-api and LiteLLM all ship dashboards | HIGH | Server-rendered + htmx per PROJECT |
| T14 | **Test a credential/model** ("ping" button) | OmniRoute `test`, new-api "test channel". Needed to validate keys before routing | LOW | Tiny chat request with `max_tokens` about 1. For OAuth providers use a cheap model |
| T15 | **Usage accounting** (input/output/cache tokens per request) | Every competitor shows tokens. Needed to verify caching works | LOW | Normalize OpenAI `prompt_tokens_details.cached_tokens` vs Anthropic `cache_read_input_tokens`/`cache_creation_input_tokens` (see 8.4) |
| T16 | **Embeddings passthrough** (`/v1/embeddings`) | LiteLLM, OmniRoute and new-api all proxy it. RAG tools need it | LOW | OpenAI shape only (Anthropic has no embeddings API) |
| T17 | **Docker image + single binary, multi-arch** | Self-host audience | LOW | Per PROJECT |

### 1.2 Differentiators (competitive advantage, aligned with the Core Value)

| # | Feature | Value Proposition | Complexity | Notes |
|---|---------|-------------------|------------|-------|
| D1 | **Transparent passthrough-first routing** (translate only when formats differ, never "cloak" by default) | OmniRoute rewrites heavily: it injects cache_control, `proxy_` tool prefixes, thinking defaults and system transforms. A router that keeps bodies byte-stable preserves client behavior **and** upstream prompt caching. This is rsrouter's thesis | MEDIUM | Parse to `serde_json::Value` (or `RawValue` for untouched subtrees). Change only `model`, auth and required quirks. Test with snapshot diffs of in/out bodies |
| D2 | **Cache affinity** (client key + stable-prefix fingerprint → pinned upstream key/model) | Keeps follow-up turns on the same account, so cached prefixes hit. OmniRoute has this, but its prefix analyzer ignores Anthropic top-level `system`/`tools` (see 6.4). rsrouter can do it right | MEDIUM | Prefer explicit session keys when present: Claude Code `metadata.user_id` → `session_id`, `X-Claude-Code-Session-Id`; OpenAI/Codex `prompt_cache_key`/`session_id`. Fall back to a hash of (tools, system, first user msg) **after** stripping the dynamic `x-anthropic-billing-header:` line |
| D3 | **Format-aware endpoint selection per model** (prefer the upstream endpoint that matches the client's format) | DeepSeek, Z.ai, OpenRouter, Copilot (Claude models) and FreeInference expose **both** OpenAI and Anthropic endpoints. Choosing the matching one turns translation into passthrough | MEDIUM | OmniRoute: `alternateFormats` on the registry entry and per-model `targetFormat` (`open-sse/config/providers/shared.ts`, `open-sse/config/providers/alternateFormats.ts`). Model this as `endpoints: Vec<(Format, Url)>` per provider/model |
| D4 | **Quota probing + quota-aware cooldown** for subscription providers | Shows "5h: 13% left, resets 14:20". Pre-emptively skips nearly exhausted keys instead of eating a 429. OmniRoute and 9router have it. LiteLLM/new-api do not (for subscriptions) | MEDIUM | Endpoints in 6.6. Also parse passive signals from response headers (`anthropic-ratelimit-unified-*`, `x-ratelimit-*`) for free, with no extra calls |
| D5 | **Paste-back OAuth for headless servers** (show URL → user pastes callback URL/code) | Works on a Pi or VPS behind SSO. OmniRoute's loopback server breaks remotely (see `src/lib/oauth/utils/googleLoopbackHint.ts`) | MEDIUM | Claude's redirect page shows `code#state` to copy (ideal). Codex has a real **device flow** (better than paste). Copilot and Cursor are poll flows (no paste needed). Antigravity is the hard one (see 7.3) |
| D6 | **Route trace UI** (per request: each attempt/skip/cooldown reason, final key/model/proxy, live via SSE) | OmniRoute only added a decision trace late (`open-sse/services/combo/decisionTrace.ts`, issue #10681) because operators could not audit fallbacks. First-class traces are a clear UX win | MEDIUM | Reuse OmniRoute's skip-reason vocabulary (6.8) |
| D7 | **Lean resource footprint** (runs on 1 GB VM / Pi) | OmniRoute is Next.js + Node (hundreds of MB RSS). LiteLLM is Python. A small Rust binary is a real differentiator for the homelab audience | MEDIUM | Streaming must be zero-copy where passthrough. Avoid buffering whole responses |
| D8 | **Proxy pool with health checks and per-key assignment** | Needed for multi-account subscription use from one IP. OmniRoute has registry + assignments (global/provider/combo/account scopes) + health sweeps | MEDIUM | HTTP/HTTPS/SOCKS5 with auth. Health probe against a neutral target. Auto-disable only after N consecutive **conclusive** failures (OmniRoute policy, 6.7) |
| D9 | **Proactive limits per key** (RPM/TPM/concurrency) | Avoids hitting provider 429s at all. LiteLLM has rpm/tpm per deployment. OmniRoute has `max_concurrent` per connection | MEDIUM | In-memory token buckets. Concurrency semaphores per (key) |
| D10 | **Separate admin port** | Firewall the UI, expose only the AI API. Rare among competitors (most share one port + login) | LOW | Per PROJECT |

### 1.3 Anti-Features (seem good, create problems; deliberately NOT build)

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| **Client-fingerprint cloaking arms race** (TLS/JA3 spoofing, header-order mimicry, synthesized device IDs, "ZWJ obfuscation of sensitive words") | Keeps subscription accounts from being flagged | OmniRoute spends thousands of lines on this (`open-sse/executors/base.ts` ~L950-1290, `open-sse/executors/claudeIdentity.ts`, `docs/security/STEALTH_GUIDE.md`). It is a permanent maintenance treadmill, pinned versions rot (see Codex model-gating by `Version`), and it is ToS-adjacent | Implement only the **minimum wire contract** each upstream requires (Section 7). Forward the real client's own identity headers when the client *is* the official CLI. Make versions configurable. Document account-ban risk in the README |
| **19 combo strategies / auto-scoring / fusion / judge models** | "Smart routing" | Combinatorial testing burden. OmniRoute's `open-sse/services/combo/` has 60+ files. v1 value comes from correct fallback, not cleverness | Sequential fallback behind a `Strategy` trait (PROJECT decision). Add round-robin later |
| **Semantic response cache** | Save tokens | Wrong answers served for "similar" prompts. Breaks agent loops. Upstream prompt caching already gives most of the cost win | Maximize upstream prompt-cache hits (D1, D2) |
| **Prompt compression / tool-output filtering (9router "RTK", OmniRoute compression combos)** | Save 20-40% tokens | Violates the transparency constraint. Changes model behavior. Invalidates cached prefixes | Out of scope; conflicts with PROJECT's "minimal modification" |
| **Mid-stream failover** (switch upstream after bytes reached the client) | "Never fail" | You cannot splice two models' outputs into one SSE stream without corrupting tool calls/thinking. OmniRoute keeps its holdback recovery **off by default** (`open-sse/services/streamRecovery.ts`) | Fail over only **before commit** (before the first content byte is forwarded). After commit, surface an error event |
| **Billing, credits, redemption codes, multi-tenant users** (new-api/one-api) | Share with friends | PROJECT out of scope. Big auth/billing surface | Named client keys + optional per-key limits only |
| **Built-in admin auth/SSO** | Security | PROJECT decision: separate port + Pangolin SSO | Bind UI to its own port. Document a reverse-proxy SSO setup |
| **Web-session "providers"** (chatgpt-web, claude-web, deepseek-web via cookies/browser automation) | Free access | Fragile, heavy (headless browser), clearly against ToS. OmniRoute has dozens (`*-web.ts` executors) | Not planned |
| **MCP server, A2A, agent memory, skills, playground** | Feature parity with OmniRoute | Unrelated to routing. Bloat | Not planned |
| **Translating between incompatible modalities** (Anthropic-format image/video/TTS; chat→video) | "Everything in both formats" | Anthropic has no image-gen/embeddings/audio/video APIs, so no Anthropic client can call them (see 8.1) | OpenAI-shape passthrough only for non-chat modalities |
| **Opening OAuth URLs from the server browser / local loopback callback server** | Convenience | Headless servers have no browser. Loopback is on the wrong machine | Paste-back / device / poll flows (D5) |
| **Auto-injecting `response_format` as system-prompt text** (OmniRoute `openai-to-claude.ts` ~L474) | JSON mode on Claude | Modifies the prompt and breaks caching. Anthropic now has native `output_config.format` (GA, no beta header) | Map `response_format.json_schema` → `output_config.format` (8.3) |
| **`proxy_` tool-name prefixing** (OmniRoute `CLAUDE_OAUTH_TOOL_PREFIX`) | Evade Claude OAuth tool filtering | Changes tool names (clients see them), breaks caching and exact-match tool routing | Do not rename tools. If an upstream rejects a name, fail that target and fall back |

---

## 2. Feature Dependencies

```
[SQLite + migrations]
    └──> [Providers / Credentials / Models (per credential)] ──> [Model discovery]
             │                                                   └──> [/v1/models listing]
             ├──> [Credential secrets store + OAuth token refresh (single-flight)]
             │         └──> [OAuth flows: paste-back / device / poll]
             │                   └──> [Claude Code, Codex, Copilot, Antigravity, Cursor providers]
             └──> [Proxy pool] ──> [Per-credential HTTP client (reqwest per proxy)]

[Canonical request model + Format enum] ──> [Passthrough path (same format)]
    └──> [Translators OpenAI⇄Anthropic (req, resp, SSE)] 
             └──> [Upstream-only codecs: OpenAI Responses (Codex), Gemini/v1internal (Antigravity), ConnectRPC protobuf (Cursor)]

[Client API keys] ──> [Auth middleware on AI port] ──> [Router]
[Router] = [Model/combo resolver] + [Candidate list] + [Cooldown/limits filter] + [Strategy (sequential)] + [Attempt loop w/ commit point]
    ├──requires──> [Error classifier (status + body signals → cooldown/terminal/advance/stop)]
    ├──requires──> [Cooldown store (key, model)]  <──enhances── [Quota probing]  <──enhances── [Passive rate-limit header parsing]
    ├──enhanced by──> [Cache affinity (fingerprint → pinned key/model)]
    └──emits──> [Route trace events] ──> [Request log store + retention] ──> [Live log UI (SSE)]

[Admin UI] ──requires──> [Admin API on separate port] ──> everything above

[Embeddings / Rerank / Images / Speech / Video passthrough] ──requires──> [Router + credentials] (NOT translators)

[Mid-stream failover] ──conflicts──> [Streaming transparency/low TTFT]
[Fingerprint cloaking] ──conflicts──> [Minimal body modification]
[Prompt compression] ──conflicts──> [Upstream prompt caching]
```

### Dependency Notes

- **Router requires the error classifier.** Sequential fallback is only as good as the "advance vs stop vs cooldown vs terminal" decision. A 400 caused by a malformed client body must **stop** (do not burn every target). A 400 "model not supported" / context-overflow must **advance**. OmniRoute's table: `open-sse/services/combo/statusDecisionTable.ts` (35 lines, worth porting as-is).
- **Codex, Antigravity and Cursor require upstream-only codecs.** They do not speak OpenAI chat or Anthropic messages: Codex = OpenAI **Responses API** (SSE-only), Antigravity = **Gemini `generateContent` inside a `v1internal` envelope**, Cursor = **ConnectRPC protobuf**. PROJECT's "upstream is OpenAI-like or Anthropic-like" is not enough for day-1 OAuth providers. The architecture needs an `UpstreamCodec` trait beyond the two client formats.
- **Copilot needs three endpoints.** It routes Claude models to its Anthropic-native `/v1/messages` (passthrough for Anthropic clients) and Codex-family models to `/responses`.
- **Quota probing enhances cooldowns.** It is not required for v1 routing, since reactive 429 handling works alone. Passive header parsing is almost free and should come first.
- **Cache affinity requires stable canonicalization.** The fingerprint must be computed on a format-independent view (system + tools + first user turn), or the same conversation gets different keys via OpenAI vs Anthropic clients.
- **OAuth refresh must exist before any OAuth provider ships.** Rotating refresh tokens (Codex, Claude) punish concurrent refresh with **family revocation**. See OmniRoute `open-sse/services/tokenRefresh/rotationMap.ts` and `casGuard.ts`.
- **Mid-stream failover conflicts with transparency.** See the anti-features table.

---

## 3. MVP Definition

### Launch With (v1)

- [ ] OpenAI `/v1/chat/completions` + Anthropic `/v1/messages` + `/v1/messages/count_tokens` + `/v1/models`: the core promise
- [ ] Passthrough path for same-format routes (no translation): the transparency thesis
- [ ] OpenAI⇄Anthropic chat translation (req/resp/SSE, tools, images-in, thinking, usage, errors): "every model in both formats"
- [ ] Providers + multiple credentials + per-credential models/discovery/test/enable: the PROJECT data model
- [ ] API-key providers: OpenRouter, DeepSeek, Z.ai (both endpoints), NVIDIA, Cloudflare, FreeInference + custom OpenAI-like/Anthropic-like: cheapest to ship, exercises the whole router
- [ ] OAuth: Claude Code (paste code) and Copilot (device flow): best effort/value ratio, and Copilot exercises the multi-endpoint model
- [ ] Codex (device flow or paste URL) + Responses codec: high demand, well-documented wire
- [ ] Combos with sequential fallback + commit-point semantics: core routing
- [ ] Cooldowns per (credential, model) from 429/5xx/timeouts with Retry-After/reset parsing + terminal states (banned/credits exhausted/auth invalid): rate-limit protection
- [ ] Cache affinity (explicit session key first, prefix hash fallback): Core Value
- [ ] Client API keys (named, hashed, last-used)
- [ ] Request log with route trace + body toggle + retention pruning; live SSE tail
- [ ] Admin UI (separate port) for all of the above
- [ ] Proxy pool (HTTP/HTTPS/SOCKS5 + auth), assignment per credential, health check: PROJECT active requirement

### Add After Validation (v1.x)

- [ ] Antigravity: Gemini codec + v1internal envelope + project onboarding + OAuth loopback problem (trigger: Claude/Codex stable)
- [ ] Quota probing UI (Claude `/api/oauth/usage`, Codex `wham/usage`, Copilot `copilot_internal/user`, Antigravity `fetchAvailableModels`, OpenRouter `/key`, DeepSeek `/user/balance`, Z.ai `quota/limit`) (trigger: users ask "how much is left")
- [ ] Proactive RPM/TPM/concurrency limits per credential (trigger: 429 storms seen in logs)
- [ ] `/v1/embeddings`, `/v1/rerank`, `/v1/audio/speech`, `/v1/images/generations` passthrough (OpenAI-shape upstreams only) (trigger: RAG users)
- [ ] OpenAI **Responses API** client endpoint `/v1/responses` (Codex CLI speaks only this; the codec already exists for the Codex upstream) (trigger: Codex CLI users)
- [ ] Round-robin / weighted strategy (the trait already exists)

### Future Consideration (v2+)

- [ ] Cursor provider (ConnectRPC protobuf codec + agent tool bridge; OmniRoute needs about 3,400 lines for it): highest cost and fragility of all day-1 providers (see 7.5). **Recommend moving it out of day-1**
- [ ] `/v1/videos` passthrough (async job model: needs job-id → credential stickiness)
- [ ] Image output via chat (OpenRouter `modalities`, Codex `image_generation` tool)
- [ ] Gemini-native client endpoint (`/v1beta/models/*:generateContent`), since Gemini CLI users exist

---

## 4. Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Chat endpoints (OpenAI + Anthropic) + passthrough | HIGH | MEDIUM | P1 |
| OpenAI⇄Anthropic translation (incl. SSE) | HIGH | HIGH | P1 |
| Providers/credentials/models per credential + discovery | HIGH | MEDIUM | P1 |
| Combos: sequential fallback + commit point | HIGH | MEDIUM | P1 |
| Cooldowns + error classifier | HIGH | MEDIUM | P1 |
| Client keys | HIGH | LOW | P1 |
| API-key providers (OpenRouter, DeepSeek, Z.ai, NVIDIA, Cloudflare, FreeInference) | HIGH | LOW | P1 |
| Claude Code OAuth | HIGH | MEDIUM | P1 |
| Copilot OAuth (device) | HIGH | MEDIUM | P1 |
| Codex OAuth + Responses codec | HIGH | HIGH | P1 |
| Request log + route trace + retention | HIGH | MEDIUM | P1 |
| Admin UI | HIGH | HIGH | P1 |
| Cache affinity | HIGH | MEDIUM | P1 |
| Proxy pool + health | MEDIUM | MEDIUM | P1 (PROJECT) / could slip to P2 |
| Passive rate-limit header parsing | MEDIUM | LOW | P1 |
| Quota probing + display | MEDIUM | MEDIUM | P2 |
| Antigravity | HIGH | HIGH | P2 |
| Proactive RPM/TPM/concurrency limits | MEDIUM | MEDIUM | P2 |
| Embeddings/rerank/speech/images passthrough | MEDIUM | LOW | P2 |
| `/v1/responses` client endpoint | MEDIUM | MEDIUM | P2 |
| Cursor | MEDIUM | HIGH (very) | P3 |
| Video | LOW | MEDIUM | P3 |

---

## 5. Competitor Feature Analysis

| Feature | OmniRoute | LiteLLM proxy | 9router | CLIProxyAPI | claude-code-router | new-api / one-api | OpenRouter | **rsrouter plan** |
|---|---|---|---|---|---|---|---|---|
| Client formats | OpenAI chat, Responses, Anthropic, Gemini + many media | OpenAI (+ Anthropic `/v1/messages` passthrough) | OpenAI, Anthropic, Gemini | OpenAI, Gemini, Claude, Codex | Anthropic in, any out | OpenAI (+ Claude, Gemini relays) | OpenAI + Anthropic "skin" | OpenAI chat + Anthropic messages (v1). Responses later |
| Subscription OAuth | ~25 providers | No (API keys) | Claude, Codex, Copilot, Antigravity, Cursor, etc. | Claude, Codex, Gemini CLI, Antigravity, Qwen, Grok | No (uses keys) | No | N/A | Claude, Codex, Copilot (v1), Antigravity (v1.x), Cursor (v2) |
| Fallback | 19 strategies | Fallbacks + retries + cooldown (`allowed_fails`) | Fallback / round-robin / fusion | Round-robin + failover | Scenario routing (background/think/longContext) | Channel priority/weight, auto-disable | `models[]` + provider order | Sequential (trait for more) |
| Cooldown granularity | Connection + model lockout + provider breaker + combo breaker | Deployment | Account | Account | None | Channel auto-disable | N/A | (credential, model) + credential terminal state |
| Quota view | Yes (many providers) | Spend/budget | Yes (reset countdown) | Partial | No | Quota/billing | Credits | Yes (v1.x) |
| Cache affinity | Rendezvous hash + session stickiness | Deployment affinity (optional) | Sticky account | Session-ish | No | No | Provider sticky for caching | Client key + prefix fingerprint → pinned credential/model |
| Proxy pool | Registry + scoped assignment + health | Per-deployment `api_base` only | Per-account proxy | Global proxy | Global proxy | Per-channel proxy | N/A | Pool + per-credential assignment + health |
| Logs | call_logs + artifacts + decision trace | Spend logs, callbacks (Langfuse, etc.) | Request logs | Request logs | Log file | Per-request logs | Activity | Route trace + bodies toggle + retention |
| Footprint | Heavy (Next.js/Node) | Heavy (Python) | Node | Light (Go) | Node | Go + frontend | SaaS | Very light (Rust) |

---

## 6. OmniRoute Implementation Map (what to port, where it lives)

### 6.1 Request pipeline (overview)

`src/app/api/v1/**/route.ts` → `handleChatCore()` (`open-sse/handlers/chatCore.ts`) → combo? `handleComboChat()` (`open-sse/services/combo.ts`) → per target `handleSingleModel()` → `translateRequest()` (`open-sse/translator/index.ts`) → `getExecutor(provider)` → `executor.execute()` (`open-sse/executors/base.ts` + provider subclass) → response translation (`open-sse/translator/response/*`) → SSE/JSON.

- **Translator topology: hub-and-spoke with OpenAI chat as the hub.** Formats are in `open-sse/translator/formats.ts`: `openai`, `openai-responses`, `claude`, `gemini`, `antigravity`, `codex`, `cursor`, `kiro`, `clova`. Anthropic→Gemini goes via OpenAI. **rsrouter recommendation:** a hub is fine for upstream-only formats (Responses, Gemini, protobuf). OpenAI⇄Anthropic should be a **direct** pair, because the hub loses Anthropic-only fields (thinking signatures, cache_control, `redacted_thinking`, document blocks).
- **Executors** (`open-sse/executors/*.ts`) own URL building, headers, body quirks, refresh and error parsing per provider. `BaseExecutor` provides `buildUrl`, `buildHeaders`, `transformRequest`, `refreshCredentials`, `needsRefresh` (lead time per provider) and `countTokens`. This maps well to a Rust `trait ProviderAdapter`.
- **Registry** (`open-sse/config/providers/registry/<id>/index.ts`, type `RegistryEntry` in `open-sse/config/providers/shared.ts`): `id, alias, format, executor, baseUrl(s), urlSuffix, authType (apikey|oauth), authHeader (bearer|x-api-key), headers, oauth{clientId, tokenUrl}, models[], modelsUrl, passthroughModels, alternateFormats[], forceStream, toolNameMaxLength, naiveResetTimezone, requestDefaults, timeoutMs`. Per-model fields worth keeping: `targetFormat`, `unsupportedParams` (e.g. Opus 4.7+/Sonnet 5 reject `temperature/top_p/top_k`), `contextLength`, `maxOutputTokens`, `supportsVision`, `supportedThinkingEfforts`. **Port this as static Rust data (`const`/`phf`) + user overrides in SQLite.**

### 6.2 Combos (sequential fallback semantics worth porting)

- Data: `combos` table (`name UNIQUE`, `data` JSON with `models[]`, `strategy`, `config`) in `src/lib/db/migrations/001_initial_schema.sql`. Types: `open-sse/services/combo/types.ts`.
- Attempt loop: `open-sse/services/combo/comboAttemptLoop.ts`, `executeTargetAttempt.ts`, `executeTargetClassify.ts`, `targetExhaustion.ts`.
- **400 decision table** (`open-sse/services/combo/statusDecisionTable.ts`): non-400 → advance. 400 with context-overflow / param-validation / model-scoped text → advance. 400 matching `invalid message format|malformed|context|prompt|token` → **stop** (the client's request is bad, so do not burn the chain).
- Optional **response validation** (`open-sse/services/combo/responseValidation.ts`): treat 200-with-empty/forbidden content as failure. v1 minimum: treat an **empty 200** (no content, no tool calls) and an **SSE `error` event before the first content delta** as retryable.
- **Commit point (rsrouter rule):** buffer upstream SSE events until the first *content-bearing* event (text delta, tool_use start, thinking delta), or a small byte/time cap. Before commit, any failure advances to the next target and the client sees nothing. After commit, errors become an in-stream error event. OmniRoute's `open-sse/services/streamRecovery.ts` (`HoldbackBuffer`) is the same idea, but off by default. rsrouter should make a minimal version (first-event peek) **on by default**, since it adds almost no TTFT.
- Skip reasons for tracing (`open-sse/services/combo/decisionTrace.ts`): `circuit_open, provider_cooldown, persisted_cooldown, request_exhaustion, model_lockout, quota_cutoff, availability, model_not_in_catalog, credential_gate, concurrency_cap`. Reuse the subset relevant to rsrouter.

### 6.3 Providers registry, formats and alternate endpoints

- `alternateFormats` example (DeepSeek, `open-sse/config/providers/registry/deepseek/index.ts`): primary `openai` at `https://api.deepseek.com/chat/completions`. Alternates `openai-responses` `https://api.deepseek.com/responses` and `claude` `https://api.deepseek.com/anthropic/v1/messages` (`x-api-key` + `anthropic-version: 2023-06-01`).
- Per-model `targetFormat`: Copilot tags every `claude-*` model `targetFormat: "claude"`, so it hits `messagesUrl` `https://api.githubcopilot.com/v1/messages` (`open-sse/config/providers/registry/github/index.ts`).
- `passthroughModels: true` means accepting any model ID the upstream accepts (not just catalogued ones). Needed for OpenRouter/NVIDIA/FreeInference/Antigravity.

### 6.4 Cache affinity (and a bug to avoid)

- `open-sse/services/combo/promptCacheAffinity.ts`: key = explicit `prompt_cache_key` / `metadata.prompt_cache_key`, else a prefix hash from `src/lib/promptCache/prefixAnalyzer.ts`. The target is chosen by **rendezvous hashing** over candidate connections (75% cache score + 25% OAuth session availability).
- `open-sse/services/combo/sessionStickiness.ts`: pins the session to a target, and releases the pin on failure (`releaseStickyPinOnFailure`).
- **Bug/limitation to avoid:** `analyzePrefix()` only walks leading `messages[]` with role `system`/`assistant`/`tool`. For Anthropic-format bodies (top-level `system`, `tools`) and for a conversation that starts with a user turn, the prefix is empty, so no affinity. rsrouter should fingerprint `(client_key, requested model/combo, hash(tools), hash(system minus billing-header line), hash(first user message))`.
- Codex-specific: OmniRoute sets `prompt_cache_key` + `session_id` header to a stable per-conversation ID (`open-sse/executors/codex.ts` `getPromptCacheSessionId`). A workspace-wide ID capped cache hits at about 49% (comment in the code).

### 6.5 Rate-limit / cooldown handling

- Connection cooldown fields on `provider_connections`: `rate_limited_until, test_status, last_error, last_error_type, error_code, backoff_level` (`001_initial_schema.sql`). Eligibility check is lazy: `rate_limited_until > now` means skip.
- Defaults (`open-sse/config/constants.ts` `PROVIDER_PROFILES`): OAuth `transientCooldown 5s`, `rateLimitCooldown 60s` when there is no hint, `maxBackoffLevel 8`. API key `3s`, rate-limit `0` (= use the provider hint), `maxBackoffLevel 5`. Backoff = `base * 2^level`. Clear on success.
- **Terminal states** (not cooldowns; they stay until the operator acts): `banned`, `credits_exhausted`, `expired`/auth invalid (after N refresh retries).
- Signal lists worth porting verbatim (`open-sse/services/accountFallback.ts`): `ACCOUNT_DEACTIVATED_SIGNALS` (L213: `account_deactivated`, "verify your account to continue", "this service has been disabled in this account"...), `CREDITS_EXHAUSTED_SIGNALS` (L240: `insufficient_quota`, `billing_hard_limit_reached`, `credit_balance_too_low`, "insufficient balance"...), `OAUTH_INVALID_TOKEN_SIGNALS` (L276), `MODEL_PERMANENTLY_UNAVAILABLE_PATTERNS` (L299: "no longer available", end-of-life 404/410 → long lockout, not a short cooldown), `CONTEXT_OVERFLOW_PATTERNS` (L344), `RATE_LIMIT_TEXT_PATTERNS` (L457: some providers send 400 with rate-limit text), `PARAM_VALIDATION_PATTERNS` (L468).
- **Model lockout** (per provider+connection+model) lets one key keep serving other models. Maps directly to rsrouter's `(credential, model)` cooldown.
- Retry hints: headers `retry-after`, OpenAI-style `x-ratelimit-{limit,remaining,reset}-{requests,tokens}`, Anthropic `anthropic-ratelimit-{requests,input-tokens,...}-{limit,remaining,reset}` (RFC 3339) (`open-sse/services/rateLimitManager/headers.ts`). Body hints: `error.resets_at` unix (`open-sse/services/retryAfterJson.ts`); text "reset in N days", "reset at YYYY-MM-DD HH:MM:SS" with **naive timestamps interpreted per provider TZ** (Z.ai/GLM = `+08:00`) (`open-sse/services/quotaResetParsing.ts`).
- **Claude subscription headers** (`open-sse/services/claudeLowPriority.ts`): `anthropic-ratelimit-unified-status`, `-reset`, `-5h-reset`, `-7d-reset`, `-representative-claim`, `-overage-status`, `-overage-in-use`, and `-slow-*` (low-priority lane offered at the wall). Parse these passively for quota display and pre-emptive cooldown.

### 6.6 Quota probing (which providers, which endpoints)

| Provider | Endpoint | Auth / headers | Response fields | OmniRoute file |
|---|---|---|---|---|
| Claude (OAuth) | `GET https://api.anthropic.com/api/oauth/usage` | `Authorization: Bearer <oat>`, `anthropic-beta: oauth-2025-04-20`, UA `claude-code/<ver>` (not `claude-cli`) | `five_hour.utilization` (% **used**), `five_hour.resets_at`, `seven_day.*`, `seven_day_<codename>.*` | `open-sse/services/usage/claude.ts` (endpoint self-429s, so cool it down: `open-sse/services/claudeUsageCooldown.ts`) |
| Claude account info | `GET https://api.anthropic.com/api/claude_cli/bootstrap` | same + UA `claude-cli/<ver> (external, cli)` | `oauth_account.{account_uuid, account_email, organization_uuid, organization_rate_limit_tier}` | `src/lib/oauth/providers/claude.ts` |
| Codex | `GET https://chatgpt.com/backend-api/wham/usage` | Bearer + `chatgpt-account-id: <workspaceId>` + `originator: codex_cli_rs` + `Version` + UA | `plan_type`, `rate_limit.{limit_reached, primary_window, secondary_window}.{used_percent, reset_at, reset_after_seconds, limit_window_seconds}` | `open-sse/services/usage/codex.ts`, `open-sse/services/codexUsageQuotas.ts` |
| Copilot | `GET https://api.github.com/copilot_internal/user` | `Authorization: token <github_oauth_token>` (not the Copilot token), `X-GitHub-Api-Version` | `quota_snapshots.{premium_interactions, chat, completions}`, `quota_reset_date` | `open-sse/services/usage/github.ts` |
| Antigravity | `POST {base}/v1internal:fetchAvailableModels` body `{project}` (bases: daily-cloudcode-pa → cloudcode-pa → sandbox) | Bearer + Antigravity UA | `models[<id>].quotaInfo.{remainingFraction, resetTime}` (no fraction = unknown; fraction 1 + no reset = unlimited). Also `v1internal:retrieveUserQuota` | `open-sse/services/usage/antigravity.ts` |
| Cursor | `https://cursor.com/api/dashboard/get-current-period-usage` (cookie-based) + api2 | varies | period usage | `open-sse/services/usage/cursor.ts` |
| OpenRouter | `GET https://openrouter.ai/api/v1/key` and `/api/v1/credits` | Bearer | `limit`, `usage`, `is_free_tier`, `rate_limit`; credits `total_credits/total_usage` | `open-sse/services/openrouterQuotaFetcher.ts` |
| DeepSeek | `GET https://api.deepseek.com/user/balance` | Bearer | `is_available`, `balance_infos[].{currency,total_balance,granted_balance,topped_up_balance}` | `open-sse/services/deepseekQuotaFetcher.ts` |
| Z.ai / GLM | `GET https://api.z.ai/api/monitor/usage/quota/limit` (CN: `open.bigmodel.cn/...`) | Bearer | coding-plan windows | `open-sse/services/usage/glm.ts`, `open-sse/config/glmProvider.ts` |
| NVIDIA, Cloudflare, FreeInference | none in OmniRoute | - | - | reactive 429 only |

### 6.7 Proxy pool

- Tables (`src/lib/db/migrations/004_proxy_registry.sql`): `proxy_registry(id, name, type, host, port, username, password, region, notes, status)` + `proxy_assignments(proxy_id, scope ∈ {global, provider, combo, account}, scope_id)` UNIQUE(scope, scope_id). Resolution order: account → combo → provider → global → direct (`src/lib/db/proxies.ts`).
- Transport: `open-sse/utils/proxyDispatcher.ts` (undici `ProxyAgent` for http/https, custom SOCKS5 connector; protocols `http:`, `https:`, `socks5:`).
- Health policy (`src/lib/proxyHealth/decision.ts`): downgrade only after `removeAfter` **consecutive conclusive** failures. Our own timeout or a target error is `inconclusive` and does not count. A `blocked` result (target returned 401/403/429 to this egress IP) is neutral. By default health checks never mutate status unless auto-disable/remove is opted in. Default probe target `https://httpbin.org/ip` (`src/lib/proxyHealth/probeTarget.ts`), plus provider `/models` probes. **Port the policy.** Pick a probe target the operator can configure.
- OAuth token exchange/refresh also goes through the resolved proxy (`runWithProxyContext` in `open-sse/utils/proxyFetch.ts`). Account-IP consistency matters for subscription accounts.

### 6.8 Model discovery

- Descriptors per provider: `src/app/api/providers/[id]/models/discovery/providerModelsConfig.ts` (`url, method, headers, authHeader, authPrefix, parseResponse`, pagination up to 20 pages). Normalization and capability detection: `src/lib/providerModels/modelDiscovery.ts`.
- Specific URLs: Claude `GET https://api.anthropic.com/v1/models?limit=1000` (OAuth headers per `src/lib/providerModels/claudeModelsHeaders.ts`: Bearer + `anthropic-beta: oauth-2025-04-20` + UA `claude-cli/<ver> (external, cli)`, else `x-api-key`). Codex `https://chatgpt.com/backend-api/codex/models` (`.../discovery/codex.ts`, `?client_version=`). Antigravity `POST {base}/v1internal:models` / `:fetchAvailableModels`. Copilot `GET https://api.githubcopilot.com/models` (per-account entitlement; `open-sse/services/githubCopilotModels.ts`). OpenRouter `https://openrouter.ai/api/v1/models`. DeepSeek `https://api.deepseek.com/v1/models`. NVIDIA `https://integrate.api.nvidia.com/v1/models`. Cloudflare `GET https://api.cloudflare.com/client/v4/accounts/{accountId}/ai/models/search` (use `result[].name` = `@cf/...` slug, **not** `id` which is a UUID). Z.ai `https://api.z.ai/api/coding/paas/v4/models`. FreeInference `https://freeinference.org/v1/models`.

### 6.9 Logging

- `call_logs` columns worth copying (`001_initial_schema.sql`): `timestamp, path, status, model, requested_model, provider, account, connection_id, duration, tokens_in/out/cache_read/cache_creation/reasoning, source_format, target_format, api_key_id/name, combo_name, combo_step_id, error_summary, has_request_body, has_response_body, artifact_relpath/size/sha256`. Bodies are stored as **artifacts on disk**, not in the row (keeps SQLite small). rsrouter can store bodies as zstd blobs in a separate table/file with a size cap.
- `usage_history` records per-request token accounting. `proxy_logs` records per-proxy outcome and latency.
- Retention: `src/lib/logRotation.ts` (days/size/file-count). rsrouter: a periodic prune by `created_at` and total bytes.
- Trace safety contract (`decisionTrace.ts`): traces never contain bodies, headers or credentials, so they are safe to keep longer than bodies.

---

## 7. Per-Provider Porting Notes

> Public OAuth client IDs below are values OmniRoute embeds (XOR-masked in `open-sse/utils/publicCreds.ts` to dodge secret scanners; env override via `resolvePublicCred(key, ENV)`). They are public desktop/CLI PKCE clients. rsrouter should embed them the same way, with env/config overrides. **Pinned client versions rot.** Make every version string configurable.

### 7.1 Claude Code (Anthropic subscription OAuth): confidence HIGH

- **Config:** `src/lib/oauth/constants/oauth.ts` `CLAUDE_CONFIG`. **Flow:** `src/lib/oauth/providers/claude.ts`. **Refresh:** `open-sse/services/tokenRefresh/providers/claudeOAuth.ts`. **Registry:** `open-sse/config/providers/registry/claude/index.ts`. **Wire identity:** `open-sse/executors/base.ts` (~L950-1290), `open-sse/executors/claudeIdentity.ts`, `open-sse/config/anthropicHeaders.ts`, `src/shared/constants/claudeCodeClient.ts`.
- **Authorize (PKCE S256):** `https://claude.ai/oauth/authorize?code=true&client_id=9d1c250a-e61b-44d9-88ed-5944d1962f5e&response_type=code&redirect_uri=https://platform.claude.com/oauth/code/callback&scope=org:create_api_key user:profile user:inference user:sessions:claude_code user:mcp_servers&code_challenge=…&code_challenge_method=S256&state=…` (OmniRoute adds `prompt=login` for multi-account safety).
- **Paste flow fits naturally:** the redirect page displays `code#state`. The user pastes it. Split on `#`.
- **Token exchange:** `POST https://api.anthropic.com/v1/oauth/token` **JSON** `{code, state, grant_type:"authorization_code", client_id, redirect_uri, code_verifier}`.
- **Refresh:** same URL, form-encoded `{grant_type:"refresh_token", refresh_token, client_id}`, header `anthropic-beta: oauth-2025-04-20`. `invalid_grant`/`invalid_request` → terminal. Keep the old refresh token if none is returned.
- **Inference:** `POST https://api.anthropic.com/v1/messages?beta=true`, `Authorization: Bearer sk-ant-oat…` (**not** `x-api-key`), `anthropic-version: 2023-06-01`, `anthropic-beta` must include `oauth-2025-04-20` (+ `claude-code-20250219` for agent-shaped requests; OmniRoute picks betas by request shape in `selectBetaFlags`), `user-agent: claude-cli/<ver> (external, cli)`, `x-app: cli`, `anthropic-dangerous-direct-browser-access: true`, `X-Claude-Code-Session-Id`.
- **Body requirements for OAuth tokens:** `system` must begin with blocks `x-anthropic-billing-header: cc_version=<ver>; cc_entrypoint=cli; cch=00000;` and `You are Claude Code, Anthropic's official CLI for Claude.`. These two blocks must **not** carry `cache_control`. `metadata.user_id` = JSON string `{"device_id":<64 hex>,"account_uuid":<uuid>,"session_id":<uuid>}` (key order matters per OmniRoute). **When the client is real Claude Code, all of this already arrives from the client. Pass it through and swap only auth.** Inject only for non-Claude-Code clients (e.g. opencode via OpenAI format), idempotently (`stripClaudeSystemPrefixBlocks`).
- **Model quirks:** Opus 4.7+/Sonnet 5/Fable 5 reject non-default `temperature/top_p/top_k` (registry `unsupportedParams`). Adaptive thinking only on newest models (`thinking.type="enabled"` → 400; map to `{type:"adaptive"}` + `output_config.effort`). Haiku rejects `output_config.effort` under OAuth.
- **Quota:** 6.6 (`/api/oauth/usage` + unified headers).

### 7.2 Codex (ChatGPT subscription OAuth): confidence HIGH

- **Config:** `CODEX_CONFIG` in `src/lib/oauth/constants/oauth.ts`. **Flow:** `src/lib/oauth/providers/codex.ts`. **Device flow:** `src/lib/oauth/codexDeviceFlow.ts`. **Refresh:** `open-sse/services/tokenRefresh/providers/codex.ts`. **Executor:** `open-sse/executors/codex.ts` (+ `open-sse/executors/codex/*`). **Identity:** `open-sse/config/codexClient.ts`, `src/shared/constants/codexClient.ts`. **Registry:** `open-sse/config/providers/registry/codex/index.ts`.
- **Browser PKCE:** `https://auth.openai.com/oauth/authorize?response_type=code&client_id=app_EMoamEEZ73f0CkXaXp7hrann&redirect_uri=http://localhost:1455/auth/callback&scope=openid profile email offline_access&code_challenge=…&code_challenge_method=S256&id_token_add_organizations=true&codex_cli_simplified_flow=true&originator=codex_cli_rs&prompt=login&state=…`. Paste-back works: the browser lands on a dead `localhost:1455` URL, and the user copies it from the address bar.
- **Device flow (better for headless; recommended):** `POST https://auth.openai.com/api/accounts/deviceauth/usercode` `{client_id}` → `{device_auth_id, user_code, interval}`. The user opens `https://auth.openai.com/codex/device`. Poll `POST .../deviceauth/token` `{device_auth_id, user_code}` → `{authorization_code, code_verifier}` (PKCE generated **server-side**). Then exchange at the token URL with `redirect_uri=https://auth.openai.com/deviceauth/callback`.
- **Token exchange/refresh:** `POST https://auth.openai.com/oauth/token` form: `grant_type=authorization_code|refresh_token`, `client_id`, ... Refresh errors `refresh_token_reused|invalid_grant|token_expired|invalid_token` or 401 → terminal (re-auth).
- **id_token claims** (`https://api.openai.com/auth`): `chatgpt_account_id` (→ `workspaceId`, sent as the `chatgpt-account-id` header), `chatgpt_plan_type`, `organizations[]`.
- **Inference:** `POST https://chatgpt.com/backend-api/codex/responses` (**OpenAI Responses API**, SSE always: `forceStream`; aggregate for non-stream clients). Headers: `Authorization: Bearer`, `chatgpt-account-id`, `originator: codex_cli_rs`, `Version: <codex version>` (**the backend gates new models on this**: forward the caller's version from its UA when the client is Codex CLI), `User-Agent: codex-cli/<ver> (<os>; <arch>)`, `OpenAI-Beta: responses_websockets=2026-02-06`, `session_id: <stable conversation id>`.
- **Body rules:** `instructions` required (placeholder if empty), `store: false`, `stream: true`, `include: ["reasoning.encrypted_content"]`, `prompt_cache_key` = stable conversation ID. **Strip:** `max_tokens`, `max_output_tokens`, `temperature`, `top_p`, `user`, `truncation`, `background`, `prompt_cache_retention`, `safety_identifier`, `session_id`, `conversation_id` (`open-sse/executors/codex/stripPassthroughRejectedParams.ts`). On the translated path, keep only an allowlist: `model, input, instructions, tools, tool_choice, stream, store, reasoning, service_tier, include, previous_response_id, prompt_cache_key, client_metadata, text, parallel_tool_calls` (codex.ts ~L1550). Replayed reasoning with foreign `encrypted_content` → 400 `invalid_encrypted_content` (`open-sse/executors/codex/reasoningReplayRejection.ts`): strip it on account switch.
- **Needs:** a Chat⇄Responses translator (`open-sse/translator/request/openai-responses.ts`, `open-sse/translator/response/openai-responses.ts`).

### 7.3 Antigravity (Google Cloud Code "v1internal"): confidence HIGH (wire), MEDIUM (paste flow viability)

- **Config:** `ANTIGRAVITY_CONFIG` (`src/lib/oauth/constants/oauth.ts`). **Flow:** `src/lib/oauth/providers/antigravity.ts`. **Refresh:** `open-sse/services/tokenRefresh/providers/google.ts`. **Endpoints:** `open-sse/config/antigravityUpstream.ts`. **Headers:** `open-sse/services/antigravityHeaders.ts`, `open-sse/services/antigravityVersion.ts`. **Executor:** `open-sse/executors/antigravity.ts` (+ `open-sse/executors/antigravity/*`). **Registry:** `open-sse/config/providers/registry/antigravity/index.ts`.
- **OAuth (Google, no PKCE required, has a public client secret):** `https://accounts.google.com/o/oauth2/v2/auth?client_id=1071006060591-tmhssin2h21lcre235vtolojh4g403ep.apps.googleusercontent.com&response_type=code&redirect_uri=<loopback>&scope=https://www.googleapis.com/auth/cloud-platform https://www.googleapis.com/auth/userinfo.email https://www.googleapis.com/auth/userinfo.profile https://www.googleapis.com/auth/cclog https://www.googleapis.com/auth/experimentsandconfigs&access_type=offline&prompt=consent&state=…`. Client secret = `antigravity_alt` in `open-sse/utils/publicCreds.ts`. Do **not** request `openid` (OmniRoute notes it routes into a hanging consent). Token: `POST https://oauth2.googleapis.com/token` form with `client_secret`.
- **Paste-flow risk:** OmniRoute documents that when the loopback redirect is unreachable from the approving browser, Google's native-app consent **may never redirect** (`src/lib/oauth/utils/googleLoopbackHint.ts`). Their workarounds are an SSH tunnel or a local helper CLI that mints a credential blob to paste (`src/lib/oauth/pasteCredentials.ts`). **Needs a spike before committing to paste-back for Antigravity.**
- **Post-exchange project discovery:** `POST https://cloudcode-pa.googleapis.com/v1internal:loadCodeAssist` `{metadata}` → `cloudaicompanionProject` (string or `{id}`) + tier. If there is no project: `POST .../v1internal:onboardUser` `{tier_id, metadata}`, then reload. The project ID is **required** on every request.
- **Inference:** `POST {base}/v1internal:streamGenerateContent?alt=sse` (or `:generateContent`), bases tried in order `https://daily-cloudcode-pa.googleapis.com` → `https://cloudcode-pa.googleapis.com`. Envelope: `{project, requestId, model, userAgent:"antigravity", requestType:"agent"|"image_gen", request:<Gemini generateContent body>}`. Tools force `toolConfig.functionCallingConfig.mode = "VALIDATED"`. Strip OpenAI/Claude thinking params from the envelope. UA `antigravity/ide/<ver> darwin/arm64` (version fetched from the auto-updater feed, fallback `2.1.1`).
- **Models:** Gemini + Claude (served via Gemini wire; thought signatures required on replay: `open-sse/config/defaultThinkingSignature.ts` holds fallback signatures).
- **Ban signals:** "verify your account to continue", "this service has been disabled in this account for violation" → terminal.
- **Needs:** OpenAI/Anthropic ⇄ Gemini translator (`open-sse/translator/request/openai-to-gemini.ts`, `open-sse/translator/response/gemini-to-openai.ts`, `gemini-to-claude.ts`).

### 7.4 GitHub Copilot: confidence HIGH

- **Config:** `GITHUB_CONFIG`. **Flow:** `src/lib/oauth/providers/github.ts`. **Refresh:** `open-sse/services/tokenRefresh/providers/github.ts` + `copilot.ts`. **Executor:** `open-sse/executors/github.ts`. **Headers:** `open-sse/config/providerHeaderProfiles.ts`. **Registry:** `open-sse/config/providers/registry/github/index.ts`.
- **Device flow (no paste needed):** `POST https://github.com/login/device/code` form `{client_id: Iv1.b507a08c87ecfe98, scope: "read:user"}`. The user enters the code at github.com/login/device. Poll `POST https://github.com/login/oauth/access_token` `{client_id, device_code, grant_type: urn:ietf:params:oauth:grant-type:device_code}`.
- **Two-token model:** the GitHub OAuth token (long-lived) is exchanged for a short-lived **Copilot token**: `GET https://api.github.com/copilot_internal/v2/token` with `Authorization: token <gh_token>` → `{token, expires_at}`. Refresh the Copilot token near `expires_at` (about 30 min). A 401 on inference → re-mint.
- **Inference endpoints:** `https://api.githubcopilot.com/chat/completions` (OpenAI), `/responses` (Codex/GPT-5-family models that advertise only `/responses`), `/v1/messages` (**Anthropic-native for Claude models**, so passthrough for Anthropic clients and the only path that reports cached tokens).
- **Headers:** `copilot-integration-id: copilot-developer-cli`, `editor-version: copilot/<ver>`, `user-agent: copilot/<ver> (<platform>) term/unknown`, `openai-intent: conversation-agent`, `x-interaction-type: conversation-user`, `copilot-harness-id: copilot-sdk`, `x-github-api-version: 2026-08-01`, `x-client-machine-id: <stable uuid per install>`, **`X-Initiator: user|agent`** (forward the client's value: "agent" turns don't consume premium requests; default "user"), `copilot-vision-request: true` when images are present (else empty image responses).
- **Models:** `GET https://api.githubcopilot.com/models` (entitlement-filtered, per account). **Quota:** 6.6.

### 7.5 Cursor: confidence HIGH (wire), complexity VERY HIGH

- **Config:** `CURSOR_CONFIG`. **Login:** `src/lib/oauth/services/cursorLogin.ts`. **Refresh:** `open-sse/services/tokenRefresh/providers/cursor.ts`. **Executor:** `open-sse/executors/cursor.ts` (1,846 lines) + `open-sse/executors/cursor/*`. **Codec:** `open-sse/utils/cursorAgentProtobuf.ts` (1,532 lines).
- **Login (poll flow; works headless):** generate PKCE. The user opens `https://cursor.com/loginDeepControl?challenge=<S256>&uuid=<uuid>&mode=login&redirectTarget=cli`. Server polls `GET https://api2.cursor.sh/auth/poll?uuid=…&verifier=…` (404 = pending, 200 = `{accessToken, refreshToken}`). Expiry comes from JWT `exp` (minus 5 min skew).
- **Refresh:** `POST https://api2.cursor.sh/auth/exchange_user_api_key`, `Authorization: Bearer <refreshToken>`, body `{}`. 401/403 → terminal.
- **Inference:** ConnectRPC over HTTP/2, `content-type: application/connect+proto`, `connect-protocol-version: 1`, `connect-accept-encoding: gzip`, `user-agent: connect-es/1.6.1`, `x-cursor-client-type: cli`, `x-cursor-client-version`, `x-ghost-mode` (privacy), traceparent. RPC `agent.v1.AgentService/Run` on `https://agent.api5.cursor.sh` (privacy) / `agentn.api5.cursor.sh`. It is **bidirectional**: the server sends exec/KV events (read/write/shell requests) that the client must answer (OmniRoute rejects them) and tool-call bridging. The system prompt is folded into the user message.
- **Recommendation:** move to v2 or a dedicated late phase with its own research. It is not an OpenAI/Anthropic-like upstream at all.

### 7.6 FreeInference: confidence MEDIUM

- Registry `open-sse/config/providers/registry/freeinference/index.ts`: OpenAI-compatible via `buildOpenAiCompatibleRegistryEntry`. Chat `https://freeinference.org/v1/chat/completions`, models `https://freeinference.org/v1/models`, Bearer key, `passthroughModels: true`, no static models.
- A free research-oriented inference API (email signup, API key from dashboard). Its site says it can serve OpenAI- **and Anthropic-compatible** endpoints (LOW confidence; not in OmniRoute's registry, so verify the path before exposing an Anthropic endpoint). No quota endpoint known.

### 7.7 NVIDIA (build.nvidia.com / NIM API catalog): confidence HIGH (wire), LOW (limits)

- Registry `open-sse/config/providers/registry/nvidia/index.ts`: `https://integrate.api.nvidia.com/v1/chat/completions`, Bearer (`nvapi-…`), `passthroughModels: true`, **`toolNameMaxLength: 64`** (truncate/hash long tool names and map them back in responses, or fail over).
- Models `https://integrate.api.nvidia.com/v1/models`. Embeddings `https://integrate.api.nvidia.com/v1/embeddings` (`open-sse/config/embeddingRegistry.ts`). Rerank `https://integrate.api.nvidia.com/v1/ranking` with an **NVIDIA-specific shape** (`query:{text}`, `passages:[{text}]`, response `rankings[{index, logit}]`; adapter in `open-sse/handlers/rerank.ts` `format:"nvidia"`). TTS `https://integrate.api.nvidia.com/v1/audio/speech` and ASR are marked `nvidia-tts`/`nvidia-asr` formats (not plain OpenAI) in `open-sse/config/audioRegistry.ts`. Image gen at `https://ai.api.nvidia.com/v1/genai/<model>` (`nvidia-nim` format).
- The free tier is rate-limited (commonly cited around 40 RPM; LOW confidence), so it relies on reactive 429 handling.

### 7.8 Cloudflare Workers AI: confidence HIGH

- Registry `open-sse/config/providers/registry/cloudflare-ai/index.ts` + executor `open-sse/executors/cloudflare-ai.ts`.
- **Per-credential Account ID is required.** URL `https://api.cloudflare.com/client/v4/accounts/{accountId}/ai/v1/chat/completions` (also `/ai/v1/embeddings`). Bearer API token.
- **Quirk:** the executor flattens array-of-text content parts into a single string (some Workers AI models reject content arrays).
- Discovery: `GET /client/v4/accounts/{accountId}/ai/models/search` → use `result[].name` (`@cf/...`).
- Image gen is non-OpenAI: `/ai/run/<model>` returns base64 in JSON (`cloudflare-ai-image` format, `open-sse/config/imageRegistry.ts`).
- Free allowance is 10,000 Neurons/day, reset 00:00 UTC (MEDIUM).

### 7.9 OpenRouter: confidence HIGH

- Registry `open-sse/config/providers/registry/openrouter/index.ts`: `https://openrouter.ai/api/v1/chat/completions`, Bearer, optional attribution headers `HTTP-Referer` + `X-Title`, `passthroughModels: true`, model `auto`. Key validation `https://openrouter.ai/api/v1/auth/key`.
- Models `https://openrouter.ai/api/v1/models`. Embeddings `https://openrouter.ai/api/v1/embeddings`. **Anthropic-compatible "skin"** at `https://openrouter.ai/api/v1/messages` (base `https://openrouter.ai/api`; Claude Code appends `/v1/messages`), so Anthropic clients get passthrough (MEDIUM: from OpenRouter docs/blog via search).
- Honors Anthropic-style `cache_control` on content parts for Anthropic/Gemini models on the OpenAI endpoint. Reasoning appears as `reasoning`/`reasoning_details` in responses (not `reasoning_content`).
- `:free` models have daily caps. Quota: `/api/v1/key` + `/api/v1/credits` (6.6; `open-sse/services/openrouterFreeWindow.ts` for the free-tier window).

### 7.10 DeepSeek: confidence HIGH

- Registry `open-sse/config/providers/registry/deepseek/index.ts`: OpenAI `https://api.deepseek.com/chat/completions` (Bearer), alternates Responses `https://api.deepseek.com/responses` and **Anthropic `https://api.deepseek.com/anthropic/v1/messages`** (`x-api-key`, `anthropic-version: 2023-06-01`). Models `https://api.deepseek.com/v1/models`. Balance `https://api.deepseek.com/user/balance`.
- **Quirk (critical for OpenAI clients):** in thinking mode, the assistant's prior `reasoning_content` **must be passed back** on follow-up turns with tool calls, else 400 "The reasoning_content in the thinking mode must be passed back to the API". Most OpenAI clients drop it. OmniRoute caches reasoning by assistant-message key and re-injects it (`open-sse/services/reasoningCache.ts`, `REASONING_REPLAY_PROVIDERS`). Using the Anthropic endpoint from Anthropic clients avoids this (thinking blocks round-trip natively).
- Out-of-credit bodies ("Insufficient balance", 402) → terminal `credits_exhausted`.

### 7.11 Z.ai (GLM): confidence HIGH

- Two registry entries. **`zai`** (`open-sse/config/providers/registry/zai/index.ts`): Anthropic-format `https://api.z.ai/api/anthropic/v1/messages?beta=true`, `x-api-key`, `anthropic-version`. **`glm`** (`open-sse/config/providers/registry/glm/index.ts` + `open-sse/executors/glm.ts` + `open-sse/config/glmProvider.ts`): OpenAI-format **Coding Plan** endpoint `https://api.z.ai/api/coding/paas/v4/chat/completions`, Bearer. China mirrors `https://open.bigmodel.cn/api/coding/paas/v4/...` and `https://open.bigmodel.cn/api/anthropic/v1/messages`.
- In rsrouter, model this as **one provider with two endpoints** (D3). The same key works on both (Coding Plan key).
- Quirks: GLM-5.3+ **rejects `thinking.type="disabled"`** (force enabled). Long timeouts (OmniRoute uses 50 min, per Z.ai Coding Plan FAQ `API_TIMEOUT_MS=3000000`). Default `max_tokens` 16,384. **429 bodies contain naive reset timestamps in Asia/Shanghai time (+08:00)** (`naiveResetTimezone`). Weekly/monthly limit text "[1310] ... will reset at ...".
- Models `https://api.z.ai/api/coding/paas/v4/models`. Quota `https://api.z.ai/api/monitor/usage/quota/limit`. (The general, non-coding PaaS API lives at `https://api.z.ai/api/paas/v4`: MEDIUM, from prior knowledge, not OmniRoute.)

### 7.12 Auth summary table

| Provider | Flow type | Paste needed? | Refresh | Token rotation risk |
|---|---|---|---|---|
| Claude Code | Auth code + PKCE | Yes: paste `code#state` from callback page (clean) | refresh_token (form) | Yes: single-flight + CAS |
| Codex | Device flow (preferred) or PKCE paste URL | No (device) / Yes (PKCE) | refresh_token (form) | **High**: `refresh_token_reused` revokes the family |
| Copilot | GitHub device flow + Copilot token mint | No | Re-mint Copilot token from GH token; GH token long-lived | Low |
| Antigravity | Google auth code (+secret) | Maybe: consent may hang without loopback (spike) | Google refresh (form + secret) | Low |
| Cursor | Deep-control PKCE + server poll | No | `exchange_user_api_key` | Medium |
| API-key providers | Static key (+ Cloudflare account ID) | - | - | - |

---

## 8. Endpoint / Format Compatibility Matrix

### 8.1 Which API families exist on each side

| Modality | OpenAI-like client endpoint | Anthropic-like client endpoint | Verdict |
|---|---|---|---|
| **Chat** | `POST /v1/chat/completions` (+ `/v1/responses` later) | `POST /v1/messages`, `POST /v1/messages/count_tokens` | **Both. Full translation matrix (8.2)** |
| **Model list** | `GET /v1/models` (OpenAI shape) | `GET /v1/models` (Anthropic shape) | Both: pick the shape by request headers |
| **Embeddings** | `POST /v1/embeddings` | *(Anthropic has no embeddings API; it recommends Voyage)* | **OpenAI only.** Passthrough to OpenAI-shape upstreams (NVIDIA, Cloudflare `/ai/v1/embeddings`, OpenRouter, custom). Out of scope for Anthropic |
| **Rerank** | *(OpenAI has no rerank API)*. De-facto standard is **Cohere/Jina `POST /v1/rerank`** `{model, query, documents, top_n, return_documents}` → `{results:[{index, relevance_score, document?}]}` (what LiteLLM and OmniRoute expose) | none | **Cohere-shape only.** Passthrough to Cohere/Jina/Voyage-shape upstreams. Small adapter for NVIDIA `/v1/ranking` (`query.text`, `passages[].text` → `rankings[].logit`). Out of scope for Anthropic |
| **Images** | `POST /v1/images/generations` (+ `/edits`) | none (Claude cannot output images) | **OpenAI only.** Passthrough to OpenAI-shape image upstreams. NVIDIA (`/v1/genai/<model>`) and Cloudflare (`/ai/run/<model>`, base64 JSON) need per-provider adapters (v1.x+). Codex image gen = Responses `image_generation` hosted tool (v2). Anthropic-chat→image: **out of scope** (no Anthropic response block can carry a generated image) |
| **Video** | `POST /v1/videos` (async job: create → `GET /v1/videos/{id}` poll → `GET /v1/videos/{id}/content`) | none | **OpenAI only, passthrough, v2.** Needs job-id → credential stickiness. No day-1 provider offers it. Out of scope for Anthropic |
| **Audio speech (TTS)** | `POST /v1/audio/speech` `{model, input, voice, response_format, speed}` → binary audio | none | **OpenAI only.** Passthrough to OpenAI-shape TTS upstreams. NVIDIA TTS is marked non-standard in OmniRoute (adapter later). Out of scope for Anthropic |
| Audio input to chat | `input_audio` content part | not supported by Claude | Drop/reject when translating to Anthropic (Anthropic's own compat layer strips it) |

**Conclusion:** the "Anthropic-like" surface is **chat only** (messages + count_tokens + models). Every non-chat modality is **OpenAI-shape passthrough** (Cohere-shape for rerank). No cross-format translation. This matches PROJECT's "completely incompatible combos are out of scope".

### 8.2 Chat routing matrix (client format × upstream format)

| Client → / Upstream ↓ | OpenAI chat upstream (NVIDIA, Cloudflare, OpenRouter, DeepSeek, Z.ai-glm, FreeInference, Copilot chat, custom) | Anthropic messages upstream (Claude key/OAuth, Z.ai-zai, DeepSeek-anthropic, OpenRouter skin, Copilot `/v1/messages`, custom) | Responses upstream (Codex, Copilot `/responses`) | Gemini v1internal (Antigravity) | ConnectRPC (Cursor) |
|---|---|---|---|---|---|
| **OpenAI chat client** | **1:1 passthrough** (rewrite `model`, auth; apply provider quirks) | **Translate** (8.3). Good fidelity except thinking-signature replay | Translate chat→responses (HIGH complexity; allowlist) | Translate chat→Gemini + envelope | Translate → protobuf (v2) |
| **Anthropic client** | **Translate** (8.3). Loses: server tools, citations, document blocks (partial), cache_control (unless upstream honors it), thinking signatures | **1:1 passthrough** (rewrite `model`, auth, filter `anthropic-beta`) | Translate via chat hub (lossy on thinking) | Translate via chat hub | v2 |

**Rule:** when a provider/model offers multiple endpoints, **pick the one matching the client format** (D3). Translate only when there is no match.

### 8.3 OpenAI chat ⇄ Anthropic messages field mapping (request)

| Concept | OpenAI chat | Anthropic messages | Mapping notes / fidelity |
|---|---|---|---|
| System | `messages[]` role `system`/`developer`, anywhere | top-level `system` (string or `[{type:"text", text, cache_control?}]`) | O→A: hoist and concatenate leading system/developer messages (Anthropic's own compat layer does this). Mid-conversation system messages: convert to user text or hoist (document the choice). A→O: one `system` message. Keep blocks as a text-part array when the upstream accepts arrays, else join with `\n` (Cloudflare needs strings). **Strip the `x-anthropic-billing-header:` line when going A→non-Anthropic** (OmniRoute `claude-to-openai.ts` L10-19: it rotates per request and destroys caching) |
| Max tokens | `max_tokens` / `max_completion_tokens` (optional) | `max_tokens` (**required**) | O→A: default when missing (per-model `maxOutputTokens`, or a sane default like 8192/16384). Must exceed `thinking.budget_tokens` (OmniRoute `openai-to-claude/thinkingBudget.ts`) |
| Sampling | `temperature` 0-2, `top_p`, `presence/frequency_penalty`, `seed`, `logit_bias`, `n`, `logprobs` | `temperature` 0-1, `top_p`, `top_k` | Clamp temperature to ≤1. Drop penalties/seed/logit_bias/logprobs. `n>1` → reject 400. Drop temperature/top_p when thinking is on or the model rejects them (`unsupportedParams`) |
| Stop | `stop` (string or array) | `stop_sequences` (array) | Direct. Anthropic ignores whitespace-only stops |
| User id | `user` | `metadata.user_id` | Direct (but for Claude OAuth, `metadata.user_id` has a fixed JSON format; see 7.1) |
| Text content | string or `[{type:"text"}]` | string or `[{type:"text"}]` | Direct |
| Images in | `{type:"image_url", image_url:{url: data:…;base64 | https://…, detail}}` | `{type:"image", source:{type:"base64", media_type, data} | {type:"url", url}}` | Direct both ways. `detail` dropped. Some upstreams do not fetch URLs, so optionally download and base64 (off by default for transparency) |
| Files/PDF in | `{type:"file", file:{file_data, filename}}` | `{type:"document", source:{type:"base64", media_type:"application/pdf", data}}` | Partial. Map PDFs only. Otherwise reject/drop |
| Audio in | `input_audio` | not supported | Reject or drop (flag in trace) |
| Tool definitions | `tools:[{type:"function", function:{name, description, parameters, strict}}]` | `tools:[{name, description, input_schema, strict?}]` | Direct. `strict` maps both ways (Anthropic strict tool use is GA). Normalize `input_schema` (object with `properties:{}` when missing; OmniRoute `claude-to-openai.ts` `normalizeToolSchema`). **Anthropic server tools** (`web_search_*`, `code_execution_*`, `bash_*`, `text_editor_*`, `computer_*`, `tool_search`, MCP connector) have **no OpenAI equivalent**: drop with a trace warning or fail over (do not fake them) |
| Tool choice | `"auto"`, `"none"`, `"required"`, `{type:"function",function:{name}}`, `parallel_tool_calls:false` | `{type:"auto"}`, `{type:"none"}`, `{type:"any"}`, `{type:"tool",name}`, `disable_parallel_tool_use:true` (inside tool_choice) | Direct. Anthropic rejects forced tool choice (`any`/`tool`) combined with thinking, so drop thinking in that case (OmniRoute base.ts ~L1040) |
| Assistant tool calls | `message.tool_calls[{id, type:"function", function:{name, arguments:<JSON string>}}]` | `content:[{type:"tool_use", id, name, input:<object>}]` | Parse/serialize `arguments`. On invalid JSON, pass `{}` + warn. **IDs:** Anthropic requires `^[a-zA-Z0-9_-]+$`, so sanitize deterministically (OmniRoute `sanitizeToolId`, `sanitizeToolResultId`) |
| Tool results | `{role:"tool", tool_call_id, content}` (one message per call) | user message with `[{type:"tool_result", tool_use_id, content, is_error?}]` | O→A: merge consecutive tool messages (+ a following user text) into **one** user message. Drop orphaned results (OmniRoute `toolCallHelper.ts`: `fixMissingToolResponses`, `stripOrphanedToolResults`). A→O: split into N tool messages. Images inside `tool_result` get lifted to a following user message (OpenAI tool messages cannot carry images: `claude-to-openai.ts` ~L487). `is_error` has no OpenAI field (prefix text) |
| JSON output | `response_format: {type:"json_schema", json_schema:{schema, strict}}` / `{type:"json_object"}` | `output_config: {format: {type:"json_schema", schema}}` (GA, no beta header) | json_schema maps directly. `json_object`: no exact equivalent (either drop, or use a permissive schema; do **not** inject prompt text by default) |
| Reasoning request | `reasoning_effort: none|minimal|low|medium|high|xhigh` | `thinking:{type:"enabled", budget_tokens}` (legacy/manual) or `{type:"adaptive"}` + `output_config.effort: low|medium|high|xhigh|max` (current models) | Per-model: adaptive-only models get effort mapping. Older models get budget buckets (OmniRoute `claude-to-openai.ts` L289+ has reverse buckets). Anthropic's own compat layer **ignores** `reasoning_effort`, so rsrouter doing this mapping is a value-add |
| Prompt caching | automatic (OpenAI). Some OpenAI-compatible upstreams honor `cache_control` on parts (OpenRouter, DashScope) | block-level `cache_control:{type:"ephemeral", ttl?:"5m"|"1h"}` (≤4 breakpoints) **or top-level `cache_control`** (automatic caching; the breakpoint moves to the last cacheable block) | **O→A: add top-level `cache_control: {type:"ephemeral"}`** when the client sent no breakpoints and fewer than 4 exist. That is one field, no content rewrite, near-100% cache reads after turn 1. **A→O: drop `cache_control`** unless the upstream is known to honor it (OmniRoute `providerHonorsOpenAIFormatCacheControl`). A→A: **pass through untouched** |
| Anthropic-only extras | - | `anthropic-beta` header, `context_management`, `container`, `mcp_servers`, `service_tier`, `metadata` | A→A: pass through (filter betas the target cannot take; e.g. `context-1m` only on qualifying models). A→O: drop |
| OpenAI-only extras | `store`, `prediction`, `modalities`, `audio`, `web_search_options`, `service_tier` | - | O→A: drop |

### 8.4 Response / usage / stop-reason mapping

| Concept | OpenAI | Anthropic | Mapping |
|---|---|---|---|
| Text | `choices[0].message.content` | `content[{type:"text"}]` | Concatenate/split |
| Thinking out | non-standard: `reasoning_content` (DeepSeek convention; opencode/ai-sdk read it) or `reasoning`/`reasoning_details` (OpenRouter) | `content[{type:"thinking", thinking, signature}]`, `{type:"redacted_thinking", data}` | A→O: emit `reasoning_content`. **The signature is lost** for standard OpenAI clients. O→A: `reasoning_content` → thinking block **without a valid signature**, so only usable when the upstream does not verify it (non-Anthropic). For real Anthropic upstreams, drop prior-turn thinking or replay from a signature cache |
| Tool calls | `tool_calls` | `tool_use` blocks | Inverse of 8.3 |
| Finish | `stop`, `length`, `tool_calls`, `content_filter` | `end_turn`, `stop_sequence`, `max_tokens`, `tool_use`, `refusal`, `pause_turn`, `model_context_window_exceeded` | end_turn/stop_sequence→stop. max_tokens→length. tool_use→tool_calls. refusal→content_filter. pause_turn (server tools)→stop (+warn). Reverse: stop→end_turn, length→max_tokens, tool_calls→tool_use, content_filter→refusal |
| Usage | `prompt_tokens` (**includes** cached), `completion_tokens`, `prompt_tokens_details.cached_tokens`, `completion_tokens_details.reasoning_tokens` | `input_tokens` (**excludes** cache), `cache_read_input_tokens`, `cache_creation_input_tokens`, `output_tokens` | A→O: `prompt_tokens = input + cache_read + cache_creation`, `cached_tokens = cache_read`. O→A: `input_tokens = prompt - cached`, `cache_read_input_tokens = cached`, `cache_creation = 0` |
| Errors | `{error:{message, type, code, param}}` | `{type:"error", error:{type, message}}` (+ `request_id`); 529 `overloaded_error` | Map types: invalid_request_error↔invalid_request_error, authentication_error, permission_error, not_found_error, rate_limit_error (429), api_error (500), overloaded_error (529 → 503 for OpenAI clients) |
| IDs | `chatcmpl-…` | `msg_…` | Synthesize. Keep the upstream ID in the trace |

### 8.5 Streaming translation (SSE state machine)

- **Anthropic events:** `message_start{message{id,model,usage}}` → per block `content_block_start{index, content_block{type:text|thinking|redacted_thinking|tool_use{id,name}}}` → `content_block_delta{delta: text_delta | thinking_delta | signature_delta | input_json_delta{partial_json}}` → `content_block_stop` → `message_delta{delta{stop_reason}, usage{output_tokens,...}}` → `message_stop`. Plus `ping` and `error`.
- **OpenAI chunks:** `chat.completion.chunk` with `choices[0].delta.{role, content, reasoning_content, tool_calls[{index, id?, type?, function{name?, arguments}}]}`, `finish_reason` on the last choice chunk, optional usage-only chunk (`choices:[]`) when `stream_options.include_usage`, then `data: [DONE]`.
- **A→O:** track block index → tool_calls index. `tool_use` start emits `{id, type, function{name, arguments:""}}`, then `input_json_delta` → `arguments` fragments. `thinking_delta` → `reasoning_content`. Drop `signature_delta` (or buffer for the signature cache). `message_delta.stop_reason` → `finish_reason`. Usage from `message_start` + `message_delta` → final usage chunk. `ping` dropped. `error` → OpenAI error chunk/close.
- **O→A:** emit `message_start` on the first chunk. Open a text block on the first content, a thinking block on the first `reasoning_content`, and a tool_use block per new `tool_calls[i].id` (close the previous open block first; Anthropic blocks cannot interleave). Concatenate `arguments` fragments as `input_json_delta`. On `finish_reason`: close the open block, `message_delta` with mapped stop_reason + usage (from the usage chunk if present; else estimate), `message_stop`.
- **Commit point** (6.2) sits *after* translation of the first content-bearing event. `message_start`/role-only chunks are not content.
- **Upstream-only codecs:** Responses SSE (`response.output_text.delta`, `response.function_call_arguments.delta`, `response.reasoning_summary_text.delta`, `response.completed` with usage) and Gemini `streamGenerateContent?alt=sse` (`candidates[0].content.parts[{text|functionCall|thought|thoughtSignature}]`, `usageMetadata`). OmniRoute: `open-sse/translator/response/openai-responses.ts`, `gemini-to-openai.ts`, `gemini-to-claude.ts`.

### 8.6 Known lossy/incompatible cases (explicitly out of scope or degraded)

| Case | Behavior |
|---|---|
| Anthropic server tools (web search, code execution, computer use, bash/text-editor server variants, MCP connector) routed to non-Anthropic upstreams | Out of scope: drop the tool (trace warning) or skip the target in combos. Never emulate |
| Thinking signature round-trip (Anthropic upstream ← OpenAI client with tool loops + thinking) | Degraded: replay from a signature cache keyed by assistant-message hash (OmniRoute `reasoningCache.ts`), else omit prior thinking (Anthropic accepts missing thinking on non-final turns in most cases; when it rejects, disable thinking for that request) |
| DeepSeek thinking + tools from OpenAI clients that drop `reasoning_content` | Needs reasoning replay cache, or prefer DeepSeek's Anthropic endpoint for Anthropic clients |
| `n > 1`, logprobs, audio in/out, `prediction` | Unsupported on Anthropic targets → 400 or drop per field (Section 8.3) |
| Citations, `document` blocks with citations, `search_result` blocks | Anthropic-only: pass through on A→A, text-only on A→O |
| Non-chat modalities in Anthropic format | Do not exist; out of scope |
| Image/video/audio output through chat endpoints | Out of scope for v1 |

---

## Sources

- **OmniRoute source** (primary, HIGH): https://github.com/diegosouzapw/OmniRoute at commit `8e3b486e5a036c6d2259c6c57fe5ef5609dfb697` (2026-09-28). Cloned and read locally. All file paths above were verified to exist.
- Anthropic OpenAI SDK compatibility (HIGH, official): https://platform.claude.com/docs/en/api/openai-sdk (redirects to `/docs/en/cli-sdks-libraries/libraries/openai-sdk`)
- Anthropic prompt caching incl. top-level automatic caching (HIGH, official): https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- Anthropic structured outputs, `output_config.format`, strict tools (HIGH, official): https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- LiteLLM proxy (MEDIUM): https://docs.litellm.ai/docs/proxy/reliability, https://docs.litellm.ai/docs/routing, https://github.com/BerriAI/litellm
- CLIProxyAPI (MEDIUM): https://github.com/router-for-me/CLIProxyAPI
- claude-code-router (MEDIUM): https://github.com/musistudio/claude-code-router
- 9router (MEDIUM): https://github.com/decolua/9router, https://github.com/decolua/9router/blob/master/gitbook/content/en/features/combos.md
- OpenRouter Anthropic skin / Claude Code (MEDIUM): https://openrouter.ai/docs/cookbook/coding-agents/claude-code-integration
- Cloudflare Workers AI OpenAI compatibility (MEDIUM): https://developers.cloudflare.com/workers-ai/configuration/open-ai-compatibility
- FreeInference (LOW-MEDIUM): https://freeinference.org/
- new-api/one-api, OpenRouter provider routing: prior knowledge (LOW-MEDIUM; verify during phase research if needed)

---
*Feature research for: self-hosted AI API router/gateway (rsrouter)*
*Researched: 2026-09-28*
