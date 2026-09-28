# Architecture Research

**Domain:** Self-hosted AI API router/gateway (OpenAI + Anthropic compatible ingress, multi-provider/multi-key egress, combos with fallback), written in Rust
**Researched:** 2026-09-28
**Confidence:** HIGH for component boundaries and the stream-commit protocol (both cross-checked against three production gateways' source). MEDIUM for the IR shape and the memory numbers (derived from reasoning, not measured).

**How this was researched:** I read the source of four reference gateways on 2026-09-28 (shallow clones of each default branch):
- **OmniRoute** (TS, commit `8e3b486`): `open-sse/translator`, `open-sse/executors`, `open-sse/services/combo/*`, `open-sse/services/tokenRefresh`, `AGENTS.md`.
- **CLIProxyAPI** (Go): `sdk/translator`, `sdk/cliproxy/auth/conductor*.go`, `sdk/cliproxy/executor`.
- **new-api** (Go): `relay/channel/adapter.go`, `controller/relay.go`.
- **Helicone ai-gateway** (Rust, last commit 2025-11): workspace layout, `endpoints/`, `middleware/mapper`.

I used web search only for LiteLLM, TensorZero and Anthropic's prompt-caching guidance.

---

## Standard Architecture

Every mature gateway in this space has the same six parts, even when they name them differently:

| Concept | OmniRoute | CLIProxyAPI | new-api | Helicone (Rust) | rsrouter name |
|---|---|---|---|---|---|
| Ingress handlers per API format | `src/app/api/v1/*` + `handlers/chatCore.ts` | `sdk/api/handlers` | `relay/*_handler.go` | `endpoints/*` | **api** |
| Format conversion | `translator/` (OpenAI **hub** + a few direct pairs) | `internal/translator/<from>/<to>` (**pairwise**, raw JSON bytes) | `Adaptor.Convert*Request` (typed DTOs, per endpoint) | `TryConvert<Src,Tgt>` (**pairwise**, typed) | **translate** (IR + passthrough) |
| Provider-specific HTTP dispatch | `executors/*` extends `BaseExecutor` | `ProviderExecutor` interface | `Adaptor.DoRequest/DoResponse` | `dispatcher/*_client.rs` | **providers** (`Adapter` trait) |
| Account/key selection, cooldown, refresh | `services/accountFallback`, `tokenRefresh`, circuit breaker | `auth.Manager` ("conductor") | channel retry policy | `router/`, balance crates | **engine** (runtime state) |
| Combo/fallback orchestration | `services/combo*` (19 strategies, ~60 files) | conductor selection + retry | `controller/relay.go` retry loop | dynamic/latency router crates | **engine** (strategy + attempt loop) |
| Persistence + logs | SQLite, 190 migrations | files/Redis | SQL DB | external | **store** |

What I learned from reading these (these findings drive the design choices below):

1. **OmniRoute's pain points are architectural.** `handlers/chatCore.ts` is 6,438 lines and `executors/base.ts` is 1,719. Combo logic is spread across about 60 files. Translation uses OpenAI Chat as the hub, but that hub drops Anthropic-only data (thinking signatures, `cache_control`, redacted thinking). OmniRoute then had to add "direct translators" that bypass the hub for exactly those cases (`translator/index.ts` lines 473-560). Provider quirks such as reasoning-effort rewrites, `max_tokens` renames and tool-schema coercion live inside the shared translator, so every provider pays for every other provider's quirks. rsrouter's design has to avoid all of this.
2. **Pre-first-byte failover is solved the same way everywhere.** CLIProxyAPI's `readStreamBootstrap` reads the upstream chunk channel until it sees the first non-empty payload, and buffers what it has read. Only then does it wrap the rest of the stream for the client. If it hits an error first, it moves on to the next credential (`StreamingBootstrapRetries`). OmniRoute's `validateQuality.ts` does a "bounded peek": parse SSE until a content verdict, then replay the buffered prefix. It caps the peek at 1 MiB because an uncapped peek once grew its heap without bound (#11804). **This prime-then-commit protocol is the core of rsrouter's pipeline.**
3. **Failures need separate scopes.** OmniRoute separates *provider circuit breaker*, *connection cooldown* and *model lockout*. CLIProxyAPI separates auth-level from model-level cooldowns. LiteLLM separates per-deployment cooldown (`allowed_fails`/`cooldown_time`) from group fallbacks. Two lessons from their bug trackers: never count caller-caused failures (client timeouts, client aborts) toward cooldowns (LiteLLM PR #41230), and never let a transient cooldown overwrite a terminal state.
4. **Rotating OAuth refresh tokens need single-flight refresh and atomic persistence.** OmniRoute added a compare-and-swap guard and a rotation map after "refresh_token_reused" events revoked whole token families (`tokenRefresh/casGuard.ts`, `rotationMap.ts`). In a single Rust process a per-credential async mutex plus immediate persistence solves this. It must still be designed in from day 1.
5. **Day-1 providers speak five upstream dialects**, based on OmniRoute's registry:

   | Dialect | Providers |
   |---|---|
   | OpenAI Chat | NVIDIA, OpenRouter, DeepSeek, FreeInference, Cloudflare, Copilot |
   | Anthropic Messages | Claude Code OAuth; Z.ai via `api.z.ai/api/anthropic/v1/messages` |
   | OpenAI Responses | Codex (`chatgpt.com/backend-api/codex/responses`, forces `stream=true`) |
   | Gemini-cloudcode envelope | Antigravity (`v1internal:streamGenerateContent?alt=sse`) |
   | Connect-RPC protobuf over HTTP/2 | Cursor |

   There are two client dialects (OpenAI Chat, Anthropic Messages). This matrix is why a canonical IR beats pairwise translation for rsrouter (see Pattern 2).

### System Overview

```
                 AI API listener (:8080, public)                Admin listener (:8081, firewalled/SSO)
┌──────────────────────────────────────────────────────┐  ┌──────────────────────────────────────────┐
│ rsr-api                                              │  │ rsr-admin                                │
│  /v1/chat/completions  /v1/messages  /v1/models      │  │  askama/maud templates + htmx            │
│  /v1/messages/count_tokens  /v1/embeddings  /rerank  │  │  SSE live logs (broadcast subscriber)    │
│  /v1/images/*  /v1/audio/speech  /v1/videos          │  │  OAuth paste-back flows, model discovery │
│  ClientKey auth → Ingress{dialect, endpoint, body}   │  │  → ControlPlane API only (no raw SQL)    │
└───────────────┬──────────────────────────────────────┘  └───────────────┬──────────────────────────┘
                │ Engine::dispatch(ingress) -> Response                    │ ControlPlane::{create_*, test, discover, oauth_*}
┌───────────────▼──────────────────────────────────────────────────────────▼──────────────────────────┐
│ rsr-engine                                                                                          │
│  Resolver ─► RoutingStrategy ─► CacheAffinity(reorder) ─► AttemptLoop ─► StreamPrimer ─► Commit     │
│     │              │                    │                     │               │                     │
│  Snapshot      Candidates          AffinityMap           Admission         Tap (usage/log)          │
│  (ArcSwap)                                           (Cooldowns, Limits,                            │
│                                                       Semaphores)                                   │
│  Runtime hot state: CooldownTracker · RateLimiter · AffinityMap · TokenCache · UsageCounters        │
│  Background: Scheduler (token pre-refresh, quota probes, proxy health, usage flush, log prune)      │
└──────┬──────────────────────┬────────────────────────────┬───────────────────────────┬──────────────┘
       │ uses                 │ uses                       │ uses                      │ LogSink (try_send)
┌──────▼───────────┐  ┌───────▼───────────────────┐  ┌─────▼──────────────────┐  ┌────▼─────────────────┐
│ rsr-translate    │  │ rsr-providers             │  │ rsr-transport          │  │ rsr-store            │
│ Dialect codecs   │  │ Adapter impls (generic    │  │ reqwest Client per     │  │ SQLite (WAL)         │
│ IR + stream IR   │  │ OpenAI/Anthropic-compat,  │  │ proxy (direct/HTTP/    │  │ migrations           │
│ RawBody surgical │  │ claude-code, codex,       │  │ SOCKS5), health state, │  │ repositories         │
│ edits, SSE codec │  │ copilot, antigravity,     │  │ timeouts, SSE framing  │  │ write-behind log     │
│ (pure, no IO)    │  │ cursor), OAuth flows,     │  │                        │  │ writer + pruner      │
│                  │  │ quota probers, error      │  │                        │  │                      │
│                  │  │ classifiers, model disc.  │  │                        │  │                      │
└──────────────────┘  └───────────────────────────┘  └────────────────────────┘  └──────────────────────┘
                              all depend on ▼
                     ┌───────────────────────────────────────────────┐
                     │ rsr-core: ids, Dialect, Endpoint, domain      │
                     │ types, errors, traits (Adapter, Credential-   │
                     │ Source, RoutingStrategy, QuotaProber, Store   │
                     │ repos, LogSink)                               │
                     └───────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Communicates With | Typical Implementation |
|---|---|---|---|
| **api** (AI listener) | Parse HTTP, authenticate the client key, enforce body limits, map route → `Ingress { dialect, endpoint, raw body }`, render `/v1/models` in the client's dialect, and turn engine errors into the client's error format | engine (calls `dispatch`), store (only through the engine's key cache) | axum `Router`, handlers under 30 lines, no business logic |
| **admin** (UI listener) | Server-rendered pages, htmx partials, SSE live logs, OAuth paste-back forms, "test model", "discover models" | engine `ControlPlane`, log broadcast channel, store read-only query repos (logs UI) | axum + askama/maud + htmx + embedded assets |
| **engine / Resolver** | Turn a requested `model` string into a target: `provider-prefix/model` → a single `ModelRef`; `combo-name` → an ordered `Vec<ModelRef>`. Expand each `ModelRef` into the credentials that have that model **enabled** (per-key model scoping) | Snapshot | Pure function over `Arc<Snapshot>` |
| **engine / RoutingStrategy** | Order candidates. v1: `Sequential` (combo order, then credential priority) | Resolver output, RuntimeView (read-only) | `trait RoutingStrategy`, one impl in v1 |
| **engine / CacheAffinity** | Fingerprint `(client_key_id, target, system+tools+first N messages)`. If the fingerprint maps to a still-healthy `(credential, upstream_model)`, move that candidate to the front. Record the winner after a successful commit | AffinityMap (bounded TTL cache) | Hash raw byte slices (xxh3-128/blake3), no JSON re-serialization |
| **engine / AttemptLoop** | For each candidate: admission (cooldown, limits, concurrency permit), auth material, build upstream request, send, classify, prime the stream, commit or fall through. Owns the "hold the client until something succeeds" behavior | CooldownTracker, RateLimiter, CredentialSource, Adapter, Transport, Translator, LogSink | A plain `async fn` loop in the request's own task |
| **engine / CooldownTracker** | Per-credential and per-(credential, model) availability: `until`, reason, backoff level, terminal flag. Expiry is lazy (checked at read time), so there are no timers | AttemptLoop (write), Strategy/UI (read), store (persist long or terminal states) | Sharded `HashMap` behind `parking_lot::RwLock`, or DashMap |
| **engine / RateLimiter** | Proactive RPM/TPM/concurrency per credential (and optionally per model). Concurrency is a `tokio::Semaphore` with `try_acquire_owned` (skip to the next candidate, never queue in v1). RPM/TPM is a GCRA or sliding window with post-hoc token debit | AttemptLoop | `governor`-style GCRA + semaphores, created when the snapshot is built |
| **engine / TokenCache (CredentialSource)** | Return `AuthMaterial` (headers, optional base-URL override, extra ids such as Copilot endpoint or Antigravity project). OAuth: refresh ahead of expiry, single-flight per credential, persist rotated tokens *before* returning them | Adapter (provider-specific refresh call), store | `tokio::sync::Mutex` per credential id + double-checked expiry |
| **engine / Scheduler** | A few long-lived tasks: token pre-refresh, quota probes, proxy health checks, usage-counter flush, log retention prune | QuotaProber, Transport, store, CooldownTracker | `tokio::time::interval` loops with jitter, cancelled via `CancellationToken` |
| **engine / ControlPlane** | Application-service API for all mutations: write to store in a transaction → rebuild snapshot → `ArcSwap::store`. Model discovery, model tests and OAuth start/complete live here | store, Adapter registry, Snapshot | Plain struct with async methods; the admin UI is a thin view over it |
| **translate** | Pure conversion. Dialect wire types, canonical IR (request, response, stream events), request/stream/response codecs per dialect, the `RawBody` top-level surgical editor, and the SSE frame encoder/decoder | nothing (pure); used by engine and providers | serde structs + `#[serde(flatten)] extra`, `Box<RawValue>`, incremental state machines |
| **providers** | One `Adapter` per upstream family. Declarative `ProviderSpec` for simple OpenAI/Anthropic-compatible providers (including user-registered custom ones). Code hooks for quirky ones (Claude Code identity headers/system prefix, Codex Responses + forced stream, Copilot token exchange, Antigravity envelope, Cursor protobuf). Also OAuth flows, quota probers, error classifiers, model discovery | core traits, translate (dialect codecs), transport | `Arc<dyn Adapter>` registry keyed by provider kind |
| **transport** | Outbound HTTP: one `reqwest::Client` (connection pool) for direct traffic, plus one per proxy (reqwest configures proxies per client). Proxy health state, timeouts (connect, first-byte), HTTP/2, rustls | Adapter (send), Scheduler (health) | `ArcSwap<HashMap<ProxyId, ProxyEntry>>` |
| **store** | SQLite schema + embedded migrations run at startup. Typed repositories (providers, credentials, credential_models, combos, client_keys, proxies, settings, cooldowns, oauth_tokens, quota_snapshots, request_logs, attempt_logs). Write-behind log writer. Retention pruner | engine (via core repo traits), admin (read queries) | Single writer connection plus a small read pool in WAL mode (sqlx or rusqlite: STACK.md decides) |
| **core** | Shared vocabulary and every trait boundary. Depends on no other internal crate | everyone | serde, http, bytes, async-trait, thiserror |

---

## Recommended Project Structure

A cargo workspace with **9 crates**. The dependency arrows point only downward. The engine depends on store *traits* in `core`, never on the `rsr-store` crate. The binary is the only composition root.

```
rsrouter/
├── Cargo.toml                    # [workspace] members, [workspace.dependencies] pinned once, release profile (lto, strip, codegen-units=1)
├── migrations/                   # SQL migrations, embedded into rsr-store at compile time
├── crates/
│   ├── rsr-core/                 # NO internal deps
│   │   └── src/
│   │       ├── ids.rs            # ProviderId, CredentialId, ModelId, ComboId, ClientKeyId, ProxyId (newtypes)
│   │       ├── dialect.rs        # enum Dialect { OpenAiChat, AnthropicMessages, OpenAiResponses, GeminiCloudCode, CursorRpc }
│   │       ├── endpoint.rs       # enum Endpoint { Chat, CountTokens, Embeddings, Rerank, Images, AudioSpeech, Video, Models }
│   │       ├── domain.rs         # Provider, Credential, CredentialModel, Combo, ClientKey, Proxy (config entities)
│   │       ├── snapshot.rs       # Snapshot: immutable indexed view built from config entities
│   │       ├── failure.rs        # Failure / FailureKind / Disposition (shared error taxonomy)
│   │       ├── log.rs            # RequestRecord, AttemptRecord, LogSink trait
│   │       └── traits/           # adapter.rs, credential.rs, strategy.rs, quota.rs, repo.rs
│   ├── rsr-translate/            # deps: core. Pure, heavily golden-tested
│   │   └── src/
│   │       ├── raw.rs            # RawBody: top-level IndexMap<String, Box<RawValue>> surgical editor
│   │       ├── sse.rs            # incremental SSE frame decoder/encoder (zero-copy over Bytes)
│   │       ├── ir/               # request.rs, response.rs, stream.rs (StreamEvent), usage.rs
│   │       └── dialects/
│   │           ├── openai_chat/  # wire types + decode/encode for req, resp, stream
│   │           ├── anthropic/    # wire types + codecs
│   │           ├── responses/    # OpenAI Responses (Codex) codecs, upstream side only in v1
│   │           └── gemini/       # Gemini/cloudcode codecs, upstream side only
│   ├── rsr-transport/            # deps: core
│   │   └── src/ {pool.rs, proxy.rs, health.rs, timeouts.rs}
│   ├── rsr-providers/            # deps: core, translate, transport
│   │   └── src/
│   │       ├── registry.rs       # AdapterRegistry: kind -> Arc<dyn Adapter>
│   │       ├── generic/          # openai_compat.rs, anthropic_compat.rs (data-driven via ProviderSpec)
│   │       ├── claude_code/      # adapter.rs, oauth.rs (PKCE paste-code), quota.rs, classify.rs
│   │       ├── codex/            # Responses dialect, forced stream, oauth (paste callback URL), quota
│   │       ├── copilot/          # GitHub device flow, copilot token exchange, quota
│   │       ├── antigravity/      # Google OAuth, cloudcode envelope, project discovery
│   │       ├── cursor/           # feature = "cursor": prost/connect framing (last, riskiest)
│   │       └── simple/           # nvidia, cloudflare, openrouter, deepseek, zai, freeinference specs + small hooks
│   ├── rsr-engine/               # deps: core, translate, providers, transport
│   │   └── src/
│   │       ├── state.rs          # AppState, Runtime (hot state), snapshot swap
│   │       ├── resolve.rs        # model/combo resolution + credential expansion
│   │       ├── strategy/         # mod.rs (trait), sequential.rs
│   │       ├── affinity.rs       # fingerprint + AffinityMap
│   │       ├── admission.rs      # cooldown.rs, limits.rs, concurrency
│   │       ├── attempt.rs        # the attempt loop
│   │       ├── prime.rs          # stream priming / commit
│   │       ├── tap.rs            # usage extraction + body capture for logs
│   │       ├── auth.rs           # TokenCache (CredentialSource impl, single-flight refresh)
│   │       ├── control.rs        # ControlPlane (mutations, discovery, tests, oauth)
│   │       └── scheduler.rs      # background maintenance loops
│   ├── rsr-store/                # deps: core
│   │   └── src/ {db.rs, migrate.rs, repos/*.rs, log_writer.rs, retention.rs}
│   ├── rsr-api/                  # deps: core, engine
│   │   └── src/ {router.rs, openai.rs, anthropic.rs, models.rs, media.rs, errors.rs, client_auth.rs}
│   ├── rsr-admin/                # deps: core, engine (+ store read-query traits)
│   │   ├── templates/            # askama/maud templates
│   │   ├── assets/               # css, htmx.min.js, embedded via rust-embed / include_bytes
│   │   └── src/ {router.rs, pages/*.rs, sse.rs, forms.rs}
│   └── rsrouter/                 # bin: deps: all. Composition root
│       └── src/ {main.rs, config.rs, wiring.rs, shutdown.rs}
└── tests/                        # workspace-level e2e: wiremock upstreams, real axum server
```

### Structure Rationale

- **rsr-core has no dependencies on other internal crates, so traits can be swapped (a stated constraint).** A future Postgres store, a WASM provider plugin or a different UI only needs to implement core traits. The compiler enforces the boundaries. In OmniRoute the dependencies run in every direction (`open-sse` imports `src/lib/db` through "dynamic-import seams").
- **rsr-translate is pure (no tokio, no reqwest).** Translation carries the most correctness risk: tools, thinking, streaming state machines. It needs hundreds of fixture tests, and a pure crate makes those fast and deterministic. Keeping IO out also stops it from becoming the dumping ground for provider quirks that OmniRoute's translator became.
- **rsr-providers is the biggest crate and the one most likely to be ported file-by-file from OmniRoute.** Each provider is one folder containing everything provider-specific: adapter, OAuth, quota probe, error classifier. Porting a provider touches one folder. Cursor sits behind a cargo feature because it pulls in protobuf tooling and is the riskiest port.
- **rsr-engine owns all mutable runtime state**, so concurrency reasoning happens in one place.
- **rsr-api and rsr-admin are thin and separate.** The two listeners map to two crates, and admin code can never leak into the public listener's router.
- **Nine crates is the upper limit.** Resist splitting further (e.g. one crate per provider). Every crate costs compile time and link time on a Raspberry Pi, and more crates add ceremony without real decoupling.

---

## Architectural Patterns

### Pattern 1: Declarative ProviderSpec + behavioral Adapter hooks

**What:** Split provider knowledge into *data* and *code*:
- **Data** (`ProviderSpec`, stored in SQLite for custom providers and compiled in for built-ins): base URLs, dialect per endpoint, auth scheme (`Bearer`, `x-api-key`, OAuth kind), static headers, models-list path.
- **Code** (`Adapter` trait): default methods cover every OpenAI/Anthropic-compatible provider. Quirky providers override only the hooks they need.

**When to use:** Always. Most day-1 providers (NVIDIA, OpenRouter, DeepSeek, FreeInference, Z.ai) are pure data, and user-registered custom providers *must* be pure data.

**Trade-offs:** Hooks must be designed well up front. Too few, and quirky providers fork the whole pipeline (OmniRoute's `opencode*.ts` family of about 20 files). Too many, and the trait becomes unreadable. The hook list below is based on what OmniRoute's executors actually override (`buildUrl`, `buildHeaders`, `transformRequest`, `shouldRetry`, `refreshCredentials`, `parseError`, `countTokens`).

**Example:**
```rust
// rsr-core/src/traits/adapter.rs
#[async_trait::async_trait]           // native AFIT is not dyn-compatible; we need Arc<dyn Adapter>
pub trait Adapter: Send + Sync + 'static {
    fn kind(&self) -> &'static str;

    /// Dialect the upstream speaks for this endpoint/model (may differ per custom model).
    fn dialect(&self, ep: Endpoint, m: &CredentialModel) -> Dialect;

    /// Build the upstream request. `body` is already in `self.dialect(..)`:
    /// either a RawBody (passthrough) or freshly encoded IR.
    /// Default: spec.base_url + path, auth headers, spec static headers.
    fn build(&self, ctx: &AttemptCtx, auth: &AuthMaterial, body: UpstreamBody)
        -> Result<http::Request<reqwest::Body>, Failure>;

    /// Surgical same-dialect patches (e.g. Claude Code system identity prefix,
    /// max_tokens -> max_completion_tokens, strip unsupported params). Default: no-op.
    fn patch(&self, _ctx: &AttemptCtx, _body: &mut RawBody) -> Result<(), Failure> { Ok(()) }

    /// Map an upstream failure to a disposition. Default: generic HTTP table.
    fn classify(&self, status: u16, headers: &http::HeaderMap, body: &[u8]) -> Failure;

    /// Decoder for this upstream's stream wire format (SSE default; Cursor = connect frames).
    fn stream_decoder(&self, ctx: &AttemptCtx) -> Box<dyn StreamDecoder>;

    async fn list_models(&self, t: &Transport, cred: &Credential, auth: &AuthMaterial)
        -> Result<Vec<DiscoveredModel>, Failure>;

    fn oauth(&self) -> Option<&dyn OAuthFlow> { None }
    fn quota(&self) -> Option<&dyn QuotaProber> { None }
    fn token_refresher(&self) -> Option<&dyn TokenRefresher> { None }
}
```

### Pattern 2: Hybrid translation, raw passthrough when dialects match and a canonical IR when they differ

**What:** Every chat request takes one of two paths:
- **Passthrough (client dialect == upstream dialect).** Parse only the *top level* of the JSON into an ordered map of raw values (`IndexMap<String, Box<RawValue>>`). Rewrite `model`, apply the adapter's `patch` hook, re-serialize. Nested content (messages, tools, `cache_control`, thinking signatures) is forwarded **as the exact bytes the client sent**. Cost is one shallow parse. Nothing is lost.
- **Translate (dialects differ).** Decode fully into the source dialect's serde types, convert to the canonical **IR**, then encode to the target dialect. Responses and streams go the other way: an upstream decoder produces IR `StreamEvent`s and a client encoder emits them.

**Why IR over pairwise:** rsrouter has 2 client dialects and 5 upstream dialects. Pairwise needs up to 2×5 request translators plus 2×5 stream translators, each a separate state machine. With an IR it is 2 client codecs + 5 upstream codecs, and adding a client dialect later (e.g. the Responses API for Codex CLI) costs one codec, not five.

**Why not OpenAI Chat as the hub (OmniRoute's choice):** OpenAI Chat cannot represent thinking signatures, redacted thinking, per-block `cache_control`, or document blocks. OmniRoute had to bolt on direct translators to route around its own hub. rsrouter's IR should be a **block-based superset shaped like Anthropic Messages**, because Anthropic's model is the richer of the two client dialects. Converting from OpenAI to the IR loses nothing. Converting from the IR to OpenAI drops, deterministically, what OpenAI cannot express.

**Unknown fields:** each dialect type carries `#[serde(flatten)] extra`. The IR keeps them tagged with their source dialect and re-emits them only to the same dialect. They are dropped (and logged at debug level) when crossing dialects.

**Determinism rule:** translation must be a pure function with stable output, because upstream prompt caches match on byte-identical prefixes. Anthropic's docs explicitly warn that key-order randomization breaks caching. Use serde_json's `preserve_order` feature, never iterate a `HashMap` when serializing, and serialize tool-call arguments with preserved key order.

**When to use:** Chat and Messages endpoints. Embeddings, rerank, images, audio and video are OpenAI-shaped only (Anthropic has no equivalents), so they are always passthrough plus `patch`.

**Trade-offs:** The IR is a real design task. It must cover tools, tool results with images, thinking/reasoning, stop reasons, usage (including cache read/write tokens) and multimodal input. Budget a dedicated phase and fixture corpus for it. The passthrough path ships first and is independent of the IR, so rsrouter is useful before the IR is finished.

**Example:**
```rust
// rsr-translate/src/ir/stream.rs — Anthropic-shaped event model, superset of OpenAI chunks
pub enum StreamEvent {
    Start { id: String, model: String },
    BlockStart { index: u32, block: BlockKind },          // Text | Thinking | ToolUse{id,name} | RedactedThinking
    TextDelta { index: u32, text: Bytes },
    ThinkingDelta { index: u32, text: Bytes },
    SignatureDelta { index: u32, signature: String },
    ToolArgsDelta { index: u32, partial_json: Bytes },
    BlockStop { index: u32 },
    Usage(Usage),                                        // input/output/cache_read/cache_write/reasoning
    Stop { reason: StopReason },
    Error(UpstreamStreamError),                          // in-band error after 200
}

pub trait StreamDecoder: Send { fn push(&mut self, chunk: &[u8], out: &mut Vec<StreamEvent>) -> Result<(), Failure>;
                                fn finish(&mut self, out: &mut Vec<StreamEvent>) -> Result<(), Failure>; }
pub trait StreamEncoder: Send { fn push(&mut self, ev: &StreamEvent, out: &mut BytesMut);
                                fn finish(&mut self, out: &mut BytesMut); }

// rsr-translate/src/raw.rs — passthrough surgical edit
pub struct RawBody(indexmap::IndexMap<String, Box<serde_json::value::RawValue>>);
impl RawBody {
    pub fn parse(b: &[u8]) -> Result<Self, Failure>;
    pub fn set_str(&mut self, key: &str, v: &str);      // e.g. "model"
    pub fn get<T: DeserializeOwned>(&self, key: &str) -> Option<T>;  // e.g. stream, max_tokens
    pub fn raw(&self, key: &str) -> Option<&RawValue>;  // for affinity hashing (system, messages)
    pub fn remove(&mut self, key: &str);
    pub fn to_bytes(&self) -> Bytes;
}
```

A **stream passthrough** also needs a lightweight *tap* that parses only the SSE events carrying usage or errors (`message_delta`, `event: error`, the final OpenAI chunk). It extracts accounting and failure data without re-encoding what goes to the client.

A **non-stream client on a stream-only upstream** (Codex forces `stream=true`): upstream decoder → IR events → `Aggregator` → IR response → client response encoder. The same aggregator serves the "stream to upstream, JSON to client" case for any provider.

### Pattern 3: Prime-then-commit streaming, holding the client until an upstream proves healthy

**What:** The client's HTTP response (status + headers) is **not created** until an upstream attempt has *primed*:
1. The upstream returned 2xx headers.
2. The upstream decoder produced its first *meaningful* event (text, thinking, tool-use or signature delta; or a complete non-stream body).
3. No in-band error event appeared before that.

Until then, every failure (non-2xx, connect error, first-byte timeout, in-band `event: error` such as Anthropic's `overloaded_error` after a 200, an empty stream that ends without content) is a *pre-commit failure*. It updates cooldowns and the loop moves to the next candidate. The client socket stays open and silent the whole time.

**Commit:** build `Response` with status 200 and the client dialect's content type. The body is `prefix (buffered primed events, re-encoded for the client) ++ rest of the upstream stream (decoded → IR → encoded, or raw passthrough)`, all poll-driven.

**After commit, fallback is impossible.** A mid-stream failure (idle timeout, upstream reset, in-band error) is emitted as a client-dialect error event, the stream is terminated, and the failure is recorded against the credential. This matches what CLIProxyAPI (`readStreamBootstrap`) and OmniRoute (`validateQuality` bounded peek) do.

**Bounds (learned from OmniRoute #11804):**
- Cap the priming buffer at about 64 KiB of upstream bytes. If the cap is reached before a content event, commit anyway.
- Cap priming *time* with a first-token timeout that is configurable per provider (reasoning models can think for a while before the first token). Choose defaults with care, e.g. 60–120 s for reasoning-heavy upstreams. Thinking deltas count as content.
- Watch for client disconnects during priming (the handler future is dropped, which drops the upstream request; see Pattern 5).

**Status fidelity vs. keepalive:** holding headers means that when *all* candidates fail, the client gets a true HTTP status (429/503 with Retry-After). Clients such as Claude Code and the OpenAI SDKs build their retry logic on that. Default to holding.

For very long fallback chains, offer an opt-in per combo, `early_commit_keepalive`: commit 200 + SSE immediately, send `: keepalive` comments (Anthropic-dialect clients also accept `event: ping`) while trying candidates, and report total failure as an in-band error event. Not in v1 unless a client is found to time out on silent sockets.

**Example:**
```rust
// rsr-engine/src/prime.rs
pub enum Primed {
    Ready { prefix: Vec<StreamEvent>, raw_prefix: Bytes, rest: UpstreamStream, decoder: Box<dyn StreamDecoder> },
    Failed(Failure),                       // → next candidate
}

pub async fn prime(mut up: UpstreamStream, mut dec: Box<dyn StreamDecoder>,
                   first_token: Duration, cap: usize) -> Primed {
    let deadline = Instant::now() + first_token;
    let (mut events, mut raw, mut bytes) = (Vec::new(), BytesMut::new(), 0usize);
    loop {
        let chunk = match tokio::time::timeout_at(deadline.into(), up.next()).await {
            Err(_) => return Primed::Failed(Failure::first_token_timeout()),
            Ok(None) => { dec.finish(&mut events).ok();
                          return if has_content(&events) { ready(events, raw, up, dec) }
                                 else { Primed::Failed(Failure::empty_stream()) } }
            Ok(Some(Err(e))) => return Primed::Failed(Failure::from_transport(e)),
            Ok(Some(Ok(c))) => c,
        };
        bytes += chunk.len(); raw.extend_from_slice(&chunk);
        if let Err(f) = dec.push(&chunk, &mut events) { return Primed::Failed(f) }
        if let Some(err) = first_error_before_content(&events) { return Primed::Failed(err) }
        if has_content(&events) || bytes >= cap { return ready(events, raw, up, dec) }
    }
}
```

### Pattern 4: Immutable config snapshot (ArcSwap) + small mutable runtime state + write-behind persistence

**What:** Three kinds of state, each with its own home:

| State | Examples | Source of truth | Hot-path access | Persistence |
|---|---|---|---|---|
| **Config** | providers, credentials, credential_models (enabled/tested per key), combos, client keys, proxies, settings | SQLite | `Arc<Snapshot>` loaded via `ArcSwap::load()`: lock-free, no DB on the request path | ControlPlane: DB txn → rebuild snapshot → swap. Snapshot rebuild is O(config size), which is tiny |
| **Runtime** | cooldowns, backoff levels, rate windows, semaphores, affinity map, OAuth access tokens, proxy health, quota snapshots | memory | sharded locks, held briefly and never across `.await` | **write-through** only for rare, important transitions: terminal credential states, cooldowns longer than ~5 min (quota resets), rotated OAuth tokens, quota snapshots. Everything else is lost on restart by design |
| **Records** | request/attempt logs, bodies, client key first/last-use, usage counters | memory → SQLite | `LogSink::record()` = `mpsc::try_send`, never awaits | **write-behind**: one writer task batches inserts in a single transaction (flush every ~250 ms or 100 records). First/last-use timestamps and counters are coalesced in memory and flushed every ~15 s |

**When to use:** Always. The request path should only read memory and write to a channel.

**Trade-offs:** Up to a few seconds of logs can be lost on a crash (acceptable; flush on graceful shutdown). Config changes are only visible after the snapshot swap (they are atomic, which is the point). The rule "no `.await` while holding a lock" must be enforced in review; clippy's `await_holding_lock` catches std/parking_lot guards.

**Example:**
```rust
pub struct AppState {
    pub snapshot: arc_swap::ArcSwap<Snapshot>,
    pub runtime: Runtime,                     // cooldowns, limits, affinity, tokens, proxy health
    pub adapters: Arc<AdapterRegistry>,
    pub transport: Arc<TransportPool>,
    pub logs: LogSink,                        // mpsc::Sender<LogMsg> + broadcast::Sender<LiveEvent>
    pub repos: Arc<dyn Repos>,                // core trait; SqliteRepos wired in main
}

// LogSink: never blocks the request path
impl LogSink {
    pub fn record(&self, rec: RequestRecord) {
        let _ = self.live.send(rec.summary());                 // broadcast: lagging UI subscribers just lose events
        if self.tx.try_send(LogMsg::Request(rec)).is_err() {  // bounded (e.g. 2048)
            self.dropped.fetch_add(1, Relaxed);                // surfaced in UI; bodies dropped first under pressure
        }
    }
}
```

### Pattern 5: One task per request, poll-driven streams, RAII finalization

**What:**
- The axum handler's task runs the whole attempt loop.
- After commit, the response body is a `Stream` that hyper polls directly: prefix chained with the upstream stream mapped through the tap and encoder. **No spawned task and no mpsc channel per request.** CLIProxyAPI uses a goroutine plus channel per stream. That is idiomatic Go but costs an extra task, a channel buffer and a wakeup per chunk in Rust.
- Request finalization (logging the outcome, releasing the concurrency permit, recording affinity, debiting TPM) lives in a guard struct owned by the body stream. Its `Drop` runs whether the stream finishes, errors, or is dropped because the client disconnected.

**When to use:** All proxying. Spawn tasks only for true background work (scheduler, log writer) and for optional hedged attempts in the future.

**Trade-offs:** `Drop` cannot be async. The guard must only do synchronous work: `try_send` to the log channel, drop the permit, update in-memory maps. That fits Pattern 4 exactly.

```rust
struct Finalizer { rec: Option<RequestRecord>, sink: LogSink, _permit: Option<OwnedSemaphorePermit>, tap: TapState }
impl Drop for Finalizer {
    fn drop(&mut self) {
        if let Some(mut r) = self.rec.take() {
            r.finish(self.tap.outcome_or(Outcome::ClientCancelled));  // client abort is NOT a credential failure
            self.sink.record(r);
        }
    }
}
```

### Pattern 6: Strategy orders candidates; execution mode is a separate engine concern

**What:** Split "which order" from "how many at once":
- `RoutingStrategy::order()` only orders (or filters) the candidate list. Round-robin, weighted, least-latency, cost-first, quota-headroom are all pure orderings that later drop in without touching the attempt loop.
- `ExecutionMode` (`Sequential` in v1; `Hedged { delay }` later; *maybe* `Race`) is a small enum inside the engine, because it changes the attempt loop's concurrency.

OmniRoute's `fusion`/`pipeline` strategies (fan-out plus a judge) are a third category, a whole different handler. They are explicitly out of scope.

**Example:**
```rust
pub trait RoutingStrategy: Send + Sync {
    fn name(&self) -> &'static str;
    /// Order/filter candidates. Must be cheap and non-blocking. State (RR cursors) lives inside the impl.
    fn order(&self, target: &Target, cands: Vec<Candidate>, view: &RuntimeView) -> Vec<Candidate>;
}
pub struct Candidate { pub model: ModelRef, pub credential: CredentialId, pub upstream_model: Arc<str>,
                       pub dialect: Dialect, pub proxy: Option<ProxyId>, pub priority: i32 }
```

Credential ordering *within* one model uses the same trait (v1: priority, then least-recently-failed). Cache affinity runs **after** the strategy and only promotes one candidate. That keeps affinity orthogonal to every future strategy.

### Pattern 7: Failure taxonomy with explicit disposition

**What:** Every failure is classified once, by the adapter's `classify` (default HTTP table, with provider overrides), into a `Failure { kind, scope, cooldown, disposition }`. The attempt loop never inspects status codes itself.

| Signal | Scope | Cooldown | Disposition |
|---|---|---|---|
| 429 + `Retry-After` / `x-ratelimit-reset-*` / `anthropic-ratelimit-*-reset` / body reset hint | credential or credential+model (provider-specific) | until reset (honor hint; else exponential from 3–5 s) | next candidate |
| Quota exhausted (402, provider-specific 429 bodies, low probed quota) | credential (or +model) | until quota reset time if known, else long (1 h); persisted | next candidate |
| 401 on OAuth credential | credential | none on first failure: force refresh and retry **same** credential once | retry once, then terminal `auth_invalid`, next |
| 401/403 on API key (not classified transient) | credential | terminal until operator action; persisted | next candidate |
| 404 model not found / model not allowed | credential+model | long (1 h), persisted | next candidate |
| 408 upstream, 5xx, connect error, first-token timeout, empty stream, pre-commit in-band error | credential+model | exponential backoff after K consecutive failures (K≈2) | next candidate |
| 400 context-length-exceeded | none | none | next candidate (a later target may have a bigger window) |
| 400 other invalid request, 413 payload | none | none | **stop**: return the upstream error (translated into the client dialect). Retrying the same bad request down a combo just multiplies latency |
| Client disconnect / client-set timeout | none | **none** | stop (LiteLLM PR #41230 lesson) |
| Proxy dead (assigned proxy unhealthy) | credential (effective) | until proxy recovers | next candidate. **Fail closed**: never silently fall back to a direct connection for a credential pinned to a proxy |

Terminal states are never overwritten by transient cooldowns (OmniRoute's rule). Cooldown writes are idempotent under concurrency: extend `until` only if the new value is later, and increment backoff at most once per window (anti-thundering-herd).

---

## Data Flow

### Request Flow (chat, streaming, combo with fallback)

```
Client (opencode / Claude Code)
  │ POST /v1/messages  Authorization|x-api-key: <client key>
  ▼
rsr-api
  ├─ ClientKey lookup (Snapshot, hashed key → ClientKeyId); mark last-use (in-memory, flushed later)
  ├─ Read body (limit), Ingress{dialect: AnthropicMessages, endpoint: Chat, body: Bytes}
  ▼
rsr-engine::dispatch
  ├─ RawBody::parse (top-level only) → requested model, stream flag
  ├─ Resolver: "my-combo" → [ModelRef A, ModelRef B, ModelRef C]
  │            each → credentials with that model enabled → Vec<Candidate>
  ├─ RoutingStrategy::order (Sequential)
  ├─ CacheAffinity: fp = hash(client_key, target, raw(system), raw(tools), raw(messages[..N]))
  │                 hit & healthy → promote candidate
  ├─ RequestRecord opened (Finalizer guard)
  ▼
AttemptLoop  ── for cand in candidates ─────────────────────────────────────────────┐
  ├─ Admission: CooldownTracker.available(cand)? RateLimiter.check? Semaphore.try_acquire?
  │     no → AttemptRecord{skipped, reason} → continue                               │
  ├─ Auth: TokenCache.material(cred)  (single-flight refresh if near expiry)         │
  ├─ Body: dialect match? RawBody.set("model") + adapter.patch()                     │
  │        else IR = decode(client) → encode(upstream)   (IR decoded once, cached)   │
  ├─ Transport(cand.proxy).send(adapter.build(...)) with connect timeout             │
  ├─ non-2xx → read ≤64 KiB error body → adapter.classify → Cooldown update         │
  │            disposition=Next → continue │ Stop → return translated error ────────┤
  ├─ 2xx → prime(stream, decoder, first_token_timeout, 64 KiB cap)                   │
  │        Failed → classify → cooldown → continue ──────────────────────────────────┘
  │        Ready  → COMMIT
  ▼
Commit
  ├─ Affinity.record(fp → cand), cooldown.clear_success(cand)
  ├─ Response 200 text/event-stream, body = prefix ++ rest
  │      rest: passthrough bytes + tap   OR  decode → IR → client encoder (+ tap)
  └─ Finalizer (dropped at end/abort): usage, TPM debit, permit release, LogSink.record
All candidates exhausted → aggregate: 429 w/ min Retry-After if all rate-limited, else 503/502,
                           error body in client dialect, full attempt trail in the log.
```

### State Management

```
             ControlPlane (admin actions, OAuth complete, discovery, tests)
                     │ 1. SQLite txn (single writer)
                     │ 2. Snapshot::build(repos) → ArcSwap::store
                     ▼
 ┌──────────── Arc<Snapshot> (immutable) ─────────────┐
 │ read lock-free by every request                    │
 └────────────────────────────────────────────────────┘
 ┌──────────── Runtime (mutable, sharded) ────────────┐       write-through (rare):
 │ Cooldowns · Limits · Semaphores · Affinity ·       │ ────► terminal states, long cooldowns,
 │ TokenCache · ProxyHealth · QuotaSnapshots          │       rotated tokens, quota snapshots
 └────────────────────────────────────────────────────┘
 Request path ──try_send──► mpsc(2048) ──► LogWriter task ──batch txn──► SQLite
              ──send─────► broadcast(256) ──► Admin SSE subscribers (lag = drop)
 Scheduler tasks ──► probes/refresh/health ──► Runtime (+ write-through)
```

### Key Data Flows

1. **Config change:** the admin form posts to `ControlPlane` → SQLite transaction → snapshot rebuild → swap. In-flight requests keep their old `Arc<Snapshot>` until they finish. New requests see the new config immediately. Runtime state keyed by ids survives the swap; entries for deleted ids are garbage-collected during the rebuild.
2. **OAuth login (paste-back):** the admin clicks "Connect Claude Code" → `adapter.oauth().start()` returns `{authorize_url, pending{state, pkce_verifier}}`. The pending state is kept in memory with a TTL. The user opens the URL in their own browser and pastes back the callback URL or code. For device flows (Copilot) the UI instead shows the user code and polls. `complete(pasted, pending)` → tokens → persist the credential → snapshot swap.
3. **Token refresh:** `TokenCache.material(cred)` checks expiry with skew (for example 5 min). If refresh is needed it takes the per-credential async mutex and re-checks (another request may have refreshed already). It then calls the adapter's refresher through the credential's proxy, **persists the rotated refresh token synchronously before returning**, and updates the cache. A background pre-refresh loop keeps idle credentials warm, so request latency rarely includes a refresh.
4. **Quota probing:** the Scheduler runs `QuotaProber::probe(cred)` for each supporting credential on a jittered interval (and on demand from the UI). It stores a `QuotaSnapshot` (windows, used %, reset times). If headroom falls below a user threshold it writes a cooldown until reset. The UI reads snapshots directly from Runtime.
5. **Model discovery/test:** the admin picks a credential → `adapter.list_models(cred)` → a diff against `credential_models` → the user enables a selection. "Test" sends a tiny prompt through the *normal* attempt path, pinned to that credential with no fallback. The result is written to `credential_models.test_status`.
6. **Log viewing:** the admin logs page queries SQLite through read-only query repos (paginated, never a full scan). The live tail subscribes to the broadcast channel. Bodies are stored only if enabled, truncated at a configurable cap, and optionally compressed. The retention pruner deletes by age/size in small batches to avoid long write locks.

---

## Two Listeners

```rust
// rsrouter/src/main.rs (sketch)
let state = wiring::build(cfg).await?;                  // migrations → repos → snapshot → runtime → adapters
let shutdown = CancellationToken::new();
let ai    = TcpListener::bind(cfg.api_addr).await?;     // e.g. 0.0.0.0:8080
let admin = TcpListener::bind(cfg.admin_addr).await?;   // e.g. 127.0.0.1:8081 by default
let bg    = scheduler::spawn_all(state.clone(), shutdown.clone());
tokio::try_join!(
    axum::serve(ai, rsr_api::router(state.clone()))
        .with_graceful_shutdown(shutdown.clone().cancelled_owned()).into_future(),
    axum::serve(admin, rsr_admin::router(state.clone()))
        .with_graceful_shutdown(shutdown.clone().cancelled_owned()).into_future(),
)?;
state.logs.flush_and_close().await;                     // drain write-behind queue
```

- Both listeners share one `Arc<AppState>` and one tokio runtime. Two runtimes would double thread and memory overhead for no isolation benefit.
- The admin router is never merged into the AI router. Even the `/healthz` routes are separate per listener.
- Default the admin bind to loopback, so exposing it is an explicit choice (compose maps the port for Pangolin/SSO).
- Streaming responses do not block graceful shutdown forever: after a grace period (30 s), cancel remaining streams via the token checked in the body stream.

---

## Scaling Considerations

The target is individuals and small groups: roughly 1–20 client keys and 5–100 concurrent streams, on 1 GB VMs and a Raspberry Pi.

| Scale | Architecture Adjustments |
|---|---|
| 1–20 concurrent streams (typical) | Everything above as-is. Idle RSS target < 30 MB. Per active stream: a few tens of KiB (socket buffers, decoder state, 64 KiB max priming buffer) **plus the retained request body** (agentic requests are 50 KB–2 MB) |
| 20–200 concurrent streams | Request bodies dominate memory. Keep a single `Bytes` per request (refcounted, shared by all attempts and the log record). Decode the IR at most once per request, not per attempt. Cap `DefaultBodyLimit` (for example 32 MB for AI API routes, since images/audio need headroom). Consider storing only a truncated body in logs |
| 200+ concurrent / multiple instances | Out of scope. SQLite plus in-memory runtime state is single-instance by design. If ever needed, the `Repos` trait lets Postgres slot in. Runtime state (cooldowns, affinity) would need Redis-like sharing. Do not design for this now |

### Scaling Priorities

1. **First bottleneck: memory from buffered bodies and log capture, not CPU.** Streaming proxying is IO-bound. Keep bodies as `Bytes`. Bound every buffer: priming cap, error-body cap (64 KiB), captured-response cap for logs (for example 1 MB, then truncate with a marker). Set tokio worker threads to the core count (1–4 on target hosts). Consider jemalloc/mimalloc or `MALLOC_ARENA_MAX=2` for long-running glibc fragmentation (STACK.md decides).
2. **Second bottleneck: SQLite write contention from logs.** Use a single writer task, batched transactions, WAL mode and `synchronous=NORMAL`, and a retention pruner that deletes in small chunks. Admin reads never block writers in WAL mode. If body logging is on for heavy agent traffic, store bodies in a separate table (or a separate DB file) so metadata queries stay fast and pruning stays cheap.

---

## Anti-Patterns

### Anti-Pattern 1: The god-handler

**What people do:** Grow one `chatCore` function that handles caching, rate limits, translation, retries, quirks and logging (OmniRoute: 6,438 lines).
**Why it's wrong:** Every provider quirk touches the hot path, nothing can be tested in isolation, and a regression in one provider breaks all of them.
**Do this instead:** An attempt loop of about 200 lines in `rsr-engine/attempt.rs` that calls trait objects. Quirks live in `Adapter::patch`/`classify`, and conversion lives in `rsr-translate`.

### Anti-Pattern 2: A lossy hub format

**What people do:** Use OpenAI Chat as the internal format, then add direct translators whenever data gets lost.
**Why it's wrong:** Thinking signatures and `cache_control` vanish on Anthropic→Anthropic-via-hub paths, which breaks multi-turn thinking and prompt caching. That defeats the project's core "transparent, cache-friendly" value.
**Do this instead:** Raw passthrough when dialects match, and an Anthropic-shaped superset IR when they don't.

### Anti-Pattern 3: Full deserialization of passthrough bodies

**What people do:** Deserialize the whole request into typed structs (or `serde_json::Value`) and re-serialize, even when nothing needs changing.
**Why it's wrong:** Unknown fields get dropped or reordered, floats can be reformatted, memory is 3–10× the raw size for `Value` trees, and it contradicts "minimal modification".
**Do this instead:** `RawBody` (top-level ordered map of `RawValue`) plus targeted edits. Only the translation path builds typed structures.

### Anti-Pattern 4: Committing the response before the upstream proves healthy

**What people do:** Forward upstream headers and bytes the instant the upstream responds.
**Why it's wrong:** A 200 followed by an in-band `overloaded_error`, or an empty stream, can no longer fall back. The client sees a failure that a healthy next candidate could have answered.
**Do this instead:** Prime-then-commit (Pattern 3), with byte and time caps.

### Anti-Pattern 5: Unbounded buffers and per-chunk task spawning

**What people do:** Peek without a cap, or spawn a task plus channel per stream, or per chunk for logging.
**Why it's wrong:** OmniRoute's heap grew without bound (#11804). On a 1 GB VM that is an OOM kill.
**Do this instead:** Put a bound on every buffer, use poll-driven body streams, and use a single write-behind log task with `try_send`.

### Anti-Pattern 6: Counting the caller's failures against upstream keys

**What people do:** Treat client aborts, client-set timeouts, and 400s caused by bad client requests as credential failures.
**Why it's wrong:** Healthy keys get cooled down, a bad request cascades through every combo target, and latency multiplies.
**Do this instead:** Use the failure taxonomy (Pattern 7): client-caused means no cooldown, and 400 invalid-request means stop, not next.

### Anti-Pattern 7: Proliferating strategies and resilience layers early

**What people do:** Ship 19 combo strategies and 3 overlapping failure layers (provider breaker, connection cooldown, model lockout) plus opt-in global provider cooldowns.
**Why it's wrong:** Debugging "why was this key skipped" requires reading multiple subsystems. OmniRoute's own AGENTS.md has a debugging section just for that.
**Do this instead:** Two scopes (credential, credential+model), one tracker, and every skip reason in the attempt log. A provider is "down" when all its credentials are cooling; no separate breaker in v1.

### Anti-Pattern 8: DB reads on the request path

**What people do:** Look up keys, providers and models in SQLite per request (OmniRoute has a `readCache` layer to compensate).
**Why it's wrong:** It adds latency and contends with the log writer.
**Do this instead:** An `ArcSwap<Snapshot>` rebuilt on every config mutation.

### Anti-Pattern 9: Silent direct-connection fallback when a proxy dies

**What people do:** When a credential's assigned proxy is unhealthy, send the request directly.
**Why it's wrong:** The user assigned the proxy on purpose (geo or IP separation of OAuth accounts). Leaking the host IP can get subscription accounts flagged.
**Do this instead:** Fail closed. Treat the credential as unavailable while its proxy is dead, and log `skipped: proxy_down`.

---

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---|---|---|
| OpenAI-compatible upstreams (NVIDIA, OpenRouter, DeepSeek, FreeInference, Cloudflare, custom) | Generic `openai_compat` adapter driven by `ProviderSpec` | Per-provider `patch` hooks for param quirks (OmniRoute's `reasoningEffort.ts` lists many). Cloudflare needs an account id in the URL path |
| Anthropic-compatible upstreams (Z.ai, DeepSeek alt endpoint, custom) | Generic `anthropic_compat` adapter | Preserve `anthropic-version` / `anthropic-beta` headers from the client where sensible (passthrough allowlist) |
| Claude Code OAuth | Anthropic Messages dialect + OAuth bearer + required beta headers + identity system prefix via `patch` | PKCE paste-code flow; `api/oauth/usage` quota probe (per OmniRoute `usage/claude.ts`). Exact headers and prefix to be ported from OmniRoute in the provider phase |
| Codex (ChatGPT subscription) | OpenAI Responses dialect, forced `stream=true`, `store=false` | Non-stream clients need the Aggregator. `backend-api/wham/usage` quota probe. Needs a Responses **upstream** codec (request encode + stream decode) |
| GitHub Copilot | OpenAI Chat dialect; GitHub device-flow token exchanged for a short-lived Copilot token | Two-level token cache (GitHub token → Copilot token). `copilot_internal/user` quota |
| Antigravity | Gemini-cloudcode envelope dialect, Google OAuth | Needs a Gemini upstream codec + envelope wrapper. Multiple base URLs with fallback inside the adapter |
| Cursor | Connect-RPC protobuf frames over HTTP/2, bidirectional agent protocol | Highest risk: custom `StreamDecoder` + request encoder behind a cargo feature. Port last, and flag it for its own research phase |
| Proxies (HTTP/HTTPS/SOCKS5 with auth) | One `reqwest::Client` per proxy, built lazily and rebuilt on edit | SOCKS5 needs reqwest's `socks` feature. Health check = periodic request through the proxy to a configurable URL |

### Internal Boundaries

| Boundary | Communication | Notes |
|---|---|---|
| api ↔ engine | Direct async call `Engine::dispatch(Ingress) -> Result<Response, ApiError>` | The engine returns an axum-agnostic `http::Response<Body>`. The api crate only maps errors to client-dialect JSON |
| admin ↔ engine | `ControlPlane` methods + `broadcast::Receiver<LiveEvent>` | Admin never writes SQL. It may read logs via a `LogQuery` repo trait |
| engine ↔ providers | `Arc<dyn Adapter>`, `dyn OAuthFlow`, `dyn QuotaProber`, `dyn TokenRefresher` | Adapters are stateless; all state is passed in via `AttemptCtx` or Runtime |
| engine ↔ translate | Plain function calls (codec traits), pure | IR decoded at most once per request |
| engine ↔ transport | `TransportPool::client(proxy) -> reqwest::Client` (cheap clone) | Adapter builds the request; engine sends it (uniform timeouts and cancellation) |
| engine ↔ store | `core::Repos` traits (async) for config and persistence; `LogSink` (sync `try_send`) for records | The store crate is wired only in the binary |
| scheduler ↔ runtime | Direct method calls on Runtime | All loops observe the shutdown `CancellationToken` |

---

## Suggested Build Order (dependencies → phases)

Each step depends only on earlier ones. Steps 1–3 already give a working single-dialect router.

1. **Skeleton and persistence:** `rsr-core` types and traits, `rsr-store` (schema v1, migrations, repos), binary with two listeners and graceful shutdown, `Snapshot` + ArcSwap, client keys. *No dependencies.*
2. **Passthrough proxy (single model, same dialect):** `rsr-transport` (direct client), generic `openai_compat` and `anthropic_compat` adapters, `RawBody`, SSE framing, non-stream + stream passthrough with tap, `/v1/models`, write-behind `LogSink`. *Depends on 1.* This validates the Pattern 5 streaming mechanics early.
3. **Multi-credential routing core:** per-credential model scoping, `Resolver`, `RoutingStrategy` (Sequential), combos, failure taxonomy + `CooldownTracker`, **prime-then-commit**, attempt logging with skip reasons. *Depends on 2.* This is the project's core value; test it hard with wiremock upstreams that 429, 500, error in-band and stall.
4. **Minimal admin UI:** ControlPlane + pages for providers/keys/models/combos/client keys + live log tail. *Depends on 3.* The UI can grow in later phases, but needs to exist early so the router can be configured without SQL.
5. **Translation IR (OpenAI Chat ↔ Anthropic Messages, both directions, streaming, tools, thinking):** `rsr-translate` IR + codecs + Aggregator + fixture corpus. *Depends on 2 for integration only;* development can start in parallel with 3–4 because the crate is pure.
6. **OAuth infrastructure + API-key providers set:** `TokenCache` (single-flight refresh, rotation persistence), `OAuthFlow` paste-back UI, model discovery, then the simple providers (NVIDIA, OpenRouter, DeepSeek, Z.ai, Cloudflare, FreeInference). *Depends on 3–4.*
7. **OAuth providers, in increasing difficulty:** Claude Code (Anthropic dialect: no new codec) → Copilot (OpenAI dialect, token exchange) → Codex (needs the Responses upstream codec + Aggregator) → Antigravity (needs the Gemini codec) → Cursor (protobuf; own research spike). *Depends on 5–6.*
8. **Protection and affinity:** proactive RPM/TPM/concurrency limits, cache affinity, QuotaProbers + quota UI, proxy pool + health checks + fail-closed assignment. *Depends on 3 (runtime state) and 6/7 (probers are per provider).* Affinity could move earlier (right after 3), since it is small and central to the caching value.
9. **Other modalities:** embeddings, rerank, images, audio speech, video (passthrough + patch), Anthropic `count_tokens` (passthrough to Anthropic-dialect upstreams, local estimate otherwise). *Depends on 2–3.*
10. **Operations polish:** retention/pruning UI, body-logging toggles, multi-arch Docker, embedded assets, sample compose. Can overlap with anything after 4.

**Research flags for the roadmap:**
- Step 3 (priming) needs careful timeout defaults per provider type.
- Step 5 (IR) needs a design spike and a fixture corpus captured from real clients (opencode, Claude Code).
- Step 7 needs per-provider porting research from OmniRoute (Codex, Antigravity and especially Cursor).
- Steps 1, 2, 4, 9 and 10 are standard patterns.

---

## Sources

- OmniRoute source, commit `8e3b486` (2026-09-28): `AGENTS.md` (request pipeline, resilience layers), `open-sse/translator/{index,registry,formats}.ts`, `open-sse/executors/base.ts`, `open-sse/services/combo/{validateQuality,promptCacheAffinity,sessionStickiness}.ts`, `open-sse/services/tokenRefresh/{casGuard,rotationMap}.ts`, `open-sse/config/providers/registry/*` (dialects per provider). https://github.com/diegosouzapw/OmniRoute (HIGH, primary source)
- CLIProxyAPI source (2026-09-28): `sdk/translator/types.go` (pairwise raw-JSON translators), `sdk/cliproxy/auth/conductor.go` (`ProviderExecutor`), `conductor_stream.go` (`readStreamBootstrap`, pre-first-byte failover), `sdk/api/handlers/handlers.go` (`StreamingBootstrapRetries`). https://github.com/router-for-me/CLIProxyAPI (HIGH, primary source)
- new-api source (2026-09-28): `relay/channel/adapter.go` (Adaptor interface), `controller/relay.go` (retry loop). https://github.com/QuantumNous/new-api (HIGH, primary source)
- Helicone ai-gateway source (last commit 2025-11): Rust workspace layout, `endpoints/mod.rs` (`Endpoint` trait), `middleware/mapper/mod.rs` (`TryConvert<Source,Target>`). https://github.com/Helicone/ai-gateway (HIGH for structure; project appears inactive)
- LiteLLM routing/cooldowns: https://docs.litellm.ai/docs/routing, https://docs.litellm.ai/docs/proxy/reliability, https://github.com/BerriAI/litellm/pull/41230 (MEDIUM)
- TensorZero gateway (Rust, normalized internal types, async persistence after response): https://github.com/tensorzero/tensorzero, https://deepwiki.com/tensorzero/tensorzero (MEDIUM)
- Anthropic prompt caching (byte-exact prefix, key-order warning, in-stream error events): https://platform.claude.com/docs/en/build-with-claude/prompt-caching (MEDIUM via search summary; verify the exact wording in the IR phase)
- Rust-specific claims (reqwest proxies configured per `Client`, axum `serve` + graceful shutdown, hyper dropping the body stream on client disconnect, `async fn` in traits not being dyn-compatible without `async-trait`/boxing, serde_json `preserve_order`/`raw_value` features): from training knowledge. MEDIUM; confirm against current crate docs in STACK.md and in phases 1–2.

---
*Architecture research for: self-hosted AI API router/gateway in Rust (rsrouter)*
*Researched: 2026-09-28*
