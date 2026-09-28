# Phase 1: Foundation, Admin Shell & Client Keys - Research

**Researched:** 2026-09-28
**Domain:** Rust service skeleton: two axum listeners, SQLite (sqlx) with embedded migrations, ArcSwap config snapshot, server-rendered admin UI (askama + htmx + Tailwind/daisyUI), client API key auth
**Confidence:** HIGH. Every core API in this document was compiled and run in a throwaway spike on this machine (rustc 1.98.1) against the exact crate versions prescribed in CLAUDE.md.

## Summary

Phase 1 is a greenfield walking skeleton. One binary starts two axum listeners (AI API and admin UI), opens SQLite through sqlx 0.9 (one writer pool, one read pool, WAL), applies embedded migrations, and loads an immutable `Snapshot` into an `ArcSwap`. The admin UI creates and manages client keys, and each mutation is committed, then the snapshot is rebuilt and swapped. The AI API authenticates `GET /v1/models` purely from the in-memory snapshot. For this phase, `/v1/models` returns an empty list, because providers only arrive in Phase 2.

I built and ran a spike with the prescribed stack: tokio 1.53.1, axum 0.8.9, tower-http 0.7.1 `CsrfLayer`, sqlx 0.9.0, askama 0.16.1 + askama_web 0.16.0, rust-embed 8.12.0, arc-swap 1.9.2, clap 4.6.7, mimalloc 0.1.52, rand 0.10.3, sha2 0.11.0 and base64 0.23.1. It passed all of these checks:
- Bearer and `x-api-key` auth.
- Rejection of an unknown key.
- Host-allowlist rejection.
- CSRF rejection of a cross-site POST.
- htmx partial rendering of a single askama block.
- Embedded assets served with the correct MIME type.
- Data surviving a restart.
- Graceful SIGINT shutdown.

The spike also found a real bug in the project's own STACK.md recipe. Building the read pool as `opts.read_only(true)` from the writer's options fails at startup with `attempt to write a readonly database`. The reader needs its own options, without `journal_mode` and `auto_vacuum` (details in Pitfall 1).

The remaining risks are process-level rather than library-level:
- sqlx compile-time macros need `sqlx-cli` (not installed) and a committed `.sqlx/` cache.
- Mutations must be serialized so an older snapshot rebuild can't overwrite a newer one.
- `tower-http`'s CsrfLayer lets non-browser requests through by design, and it does no Host allowlisting. Host allowlisting needs a small custom middleware.
- A clickjacking defense (`frame-ancestors`/`X-Frame-Options`) is needed, because CSRF checks don't stop a framed same-origin page.
- Copy-to-clipboard buttons fail on a plain-HTTP LAN admin origin.

**Primary recommendation:** Build the walking skeleton first: `rsrouter` binary → clap/TOML config → sqlx writer + reader pools → `migrate!` → Snapshot in ArcSwap → two axum routers. The first real slice is one htmx "create key" form that writes a row, rebuilds the snapshot, and makes `GET /v1/models` accept that key. Then add rename/disable/delete, usage timestamps, snippets, the polished shell, hardening layers, and FOUND-07 trait seams with stub-swap tests.

## User Constraints

No CONTEXT.md exists for this phase. The user chose to plan straight from research and requirements (`skip_discuss: true`). There are no locked discuss-phase decisions. The binding constraints are CLAUDE.md (below), the PROJECT.md Key Decisions and ROADMAP.md.

## Project Constraints (from CLAUDE.md)

These carry the same authority as locked decisions:

- **Rust for everything, including the UI (server-rendered).** No WASM UI (Leptos/Dioxus/Yew), no Node build chain.
- **SQLite with automated migrations.** sqlx 0.9, `sqlite-bundled`, `sqlx::migrate!()` embedded. Use one writer pool plus one read pool, in WAL mode. Do not mix rusqlite with sqlx. Use `default-features = false`, and do not enable the `sqlite` umbrella feature.
- **Prefer the compile-time macros** (`query!`/`query_as!`). Commit the `.sqlx/` offline cache. CI and Docker build with `SQLX_OFFLINE=true`. Dynamic SQL must be wrapped in `AssertSqlSafe(...)`.
- **axum 0.8.9 + tower 0.5 + tower-http 0.7.1.** Admin router gets `CsrfLayer`, a Host-header allowlist and compression. The AI router gets no `CompressionLayer`.
- **askama 0.16 + askama_web 0.16 (`axum-0.8`)**, `#[derive(Template, WebTemplate)]`. No maud or minijinja.
- **htmx 2.0.x (not 4.x)**, htmx-ext-sse 2.2.x, the Tailwind CSS 4 standalone CLI + daisyUI 5 (`daisyui.mjs` plugin, no npm), all embedded with **rust-embed 8**. Commit the generated `app.css`. CI fails if a rebuild changes it.
- **arc-swap `ArcSwap<Snapshot>`.** The request path never touches SQLite for config.
- **clap 4.6 (`derive`, `env`)** for bootstrap config with `RSR_*` env fallbacks. Skip figment and config-rs. All other settings live in SQLite.
- **mimalloc 0.1.52** as the `#[global_allocator]`. No jemalloc.
- **tracing 0.1 + tracing-subscriber 0.3** (env-filter, fmt, json). The domain request log is a separate `LogSink` (mpsc → batch writer), never a tracing layer.
- **Background jobs** are plain `tokio::spawn` loops with `CancellationToken`/`TaskTracker`.
- **Client API keys:** 32 random bytes → `rsr-…` base64url. Store the **SHA-256** digest for O(1) lookup. Never argon2/bcrypt.
- **thiserror in library crates, anyhow only in `main`.**
- **`panic = "unwind"`** in release (not abort). Use `CatchPanicLayer` on routers.
- **Rust toolchain** stable 1.98.1, edition 2024, `rust-version = "1.94"`, pinned in `rust-toolchain.toml`.
- **Architecture:** loosely coupled, with swappable components behind traits. Portability: amd64 + arm64.
- **GSD workflow:** file changes go through GSD commands (execute-phase).
- **Provider fidelity (MANDATORY)** does not apply in Phase 1 (no upstream providers yet), but the trait seams defined here must not preclude it. For example, the adapter trait must be able to set arbitrary headers and body markers.

<phase_requirements>
## Phase Requirements

Requirement text quoted verbatim from `.planning/REQUIREMENTS.md:12-25` and `:138,142` [VERIFIED: .planning/REQUIREMENTS.md, read this session].

| ID | Description | Research Support |
|----|-------------|------------------|
| FOUND-01 | Operator can run rsrouter as a single binary that serves the AI API and the admin UI/API on two separately configurable ports | Pattern 1 (two `axum::serve` + shared `CancellationToken`, verified in spike) |
| FOUND-02 | On startup, the SQLite database is created if missing and all pending schema migrations are applied automatically (embedded in the binary) | Pattern 2 (`create_if_missing`, `sqlx::migrate!`, `build.rs` rerun, downgrade guard via `VersionMissing`) |
| FOUND-03 | Operator can configure listen addresses, data path and log/retention defaults via environment variables or config file | Pattern 3 (clap `env` + `--config` TOML, precedence CLI > env > file > default, verified) |
| FOUND-04 | Admin port rejects cross-origin state-changing requests (CSRF protection) and requests with non-allowlisted Host headers | Pattern 6 (`CsrfLayer` rules + custom Host guard, both verified at runtime) |
| FOUND-05 | Configuration changes made via the admin UI take effect on the AI API without restart and without per-request database reads | Pattern 4 (ControlPlane: txn → rebuild → `ArcSwap::store`, serialized by a mutex) |
| FOUND-07 | Architecture exposes swappable components behind traits (provider adapter, upstream codec, routing strategy, cooldown tracker, quota prober, proxy transport, log sink, storage) | Pattern 9 (`#[async_trait]` dyn traits in `rsr-core`, `Components` registry, stub-swap tests. Native async-fn-in-trait was verified NOT dyn-compatible) |
| KEYS-01 | User can create a named client API key (e.g. `opencode`) and sees the generated secret exactly once | Pattern 5 (key gen/hash verified, one-time reveal partial with `Cache-Control: no-store`) |
| KEYS-02 | User can list, rename, disable and delete client API keys | Pattern 7 (htmx CRUD with askama `blocks`, verified partial render) |
| KEYS-03 | rsrouter records first-used and last-used timestamps per client key (including `/models` calls) | Pattern 8 (in-memory usage map + periodic flush, `COALESCE`/`MAX` UPDATE verified by `query!`) |
| KEYS-04 | AI API accepts client keys via `Authorization: Bearer` and `x-api-key` headers and rejects unknown/disabled keys with a format-appropriate error | Pattern 5 + error envelopes captured verbatim from the live OpenAI and Anthropic APIs |
| UI-01 | Server-rendered Rust admin UI (no auth, separate port) with a polished, responsive look for managing providers, credentials, models, combos, client keys, proxies, quota and logs | Pattern 7 (askama layout + daisyUI drawer/navbar, CSS pipeline verified). Placeholder pages for later areas |
| UI-05 | UI shows copy-paste setup snippets for opencode and Claude Code (base URL + key) | Snippet formats verified against the opencode and Claude Code docs. `api_public_url` config. Clipboard pitfall |
</phase_requirements>

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Bootstrap config (listen addrs, data dir, log/retention defaults) | Binary (composition root, `rsrouter` crate) | — | Read once at startup. Everything else lives in SQLite |
| DB open, pragmas, migrations | Database / Storage (`rsr-store`) | Binary (wiring) | Only the store crate knows sqlx |
| Config snapshot + mutations (ControlPlane) | API / Backend (`rsr-engine`) | Storage via `rsr-core` traits | Engine owns runtime state and the swap. Admin never writes SQL |
| Client key authentication | API / Backend (`rsr-api` middleware) | Engine snapshot (read-only) | Lock-free `ArcSwap::load()`, no DB |
| Key usage timestamps | API / Backend (engine runtime map) | Storage (periodic batch flush) | Hot path writes memory only |
| Admin pages, htmx partials | Frontend Server (SSR, `rsr-admin`) | Browser (htmx swaps, copy buttons) | Server-rendered by constraint. Browser only does swaps and clipboard |
| CSRF / Host allowlist / security headers | Frontend Server (`rsr-admin` tower layers) | — | Must wrap every admin route |
| Static assets (htmx, CSS, icons) | CDN / Static, embedded in the binary (rust-embed) | — | Single-binary requirement |
| Swappable component traits | Shared core (`rsr-core`) | — | No internal deps, so stubs can be swapped in tests |

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| tokio | 1.53.1 | Async runtime | CLAUDE.md prescription. Everything else runs on it [VERIFIED: crates.io API] |
| axum | 0.8.9 | Both HTTP listeners | CLAUDE.md. Compiled and ran in spike [VERIFIED: crates.io + spike] |
| tower | 0.5.3 (+ `util` feature in dev-deps for `ServiceExt::oneshot`) | Middleware and test harness | Spike test needed `features = ["util"]` [VERIFIED: spike] |
| tower-http | 0.7.1 (`csrf`, `trace`, `catch-panic`, `set-header`, `sensitive-headers`, `compression-gzip`, `limit`) | CSRF, headers, tracing | `CsrfLayer` API read in source [VERIFIED: crate source] |
| sqlx | 0.9.0 (`runtime-tokio`, `sqlite-bundled`, `migrate`, `macros`) | SQLite + migrations | Bundled SQLite supports `STRICT` tables (the spike migration used them) [VERIFIED: spike] |
| askama | 0.16.1 (default features) | Templates | `path=` works with default features. HTML auto-escaping verified [VERIFIED: spike] |
| askama_web | 0.16.0 (`axum-0.8`) | `IntoResponse` for templates | `#[derive(Template, WebTemplate)]` compiled [VERIFIED: spike] |
| rust-embed | 8.12.0 (`mime-guess`, `include-exclude`) | Embed `assets/` | `#[derive(rust_embed::Embed)]`, `metadata.mimetype()`, `metadata.sha256_hash()` [VERIFIED: rust-embed-utils-8.12.0/src/lib.rs:121,139] |
| arc-swap | 1.9.2 | Config snapshot | `ArcSwap::from_pointee`, `.load()`, `.store(Arc)` [VERIFIED: spike] |
| clap | 4.6.7 (`derive`, `env`) | CLI + env config | [VERIFIED: spike] |
| toml | 1.1.6 | Config file parsing | `toml::from_str` + `#[serde(deny_unknown_fields)]` [VERIFIED: spike] |
| mimalloc | 0.1.52 (`default-features = false`) | Global allocator | Compiled with the host `cc` [VERIFIED: spike] |
| tracing / tracing-subscriber | 0.1.44 / 0.3.23 | Diagnostics | CLAUDE.md [VERIFIED: crates.io] |
| tokio-util | 0.7.19 (`rt`) | `CancellationToken`, `TaskTracker` | [VERIFIED: spike] |
| async-trait | 0.1.92 | dyn-compatible async traits | Native async fn in traits is **not** dyn-compatible on rustc 1.98.1 (error E0038 reproduced) [VERIFIED: spike] |
| sha2 / base64 / rand | 0.11.0 / 0.23.1 / 0.10.3 | Key hash / encoding / CSPRNG | `rand::fill(&mut [u8;32])` uses `ThreadRng` (ChaCha12, `TryCryptoRng`) [VERIFIED: rand-0.10.3/src/lib.rs:314, rngs/thread.rs:92,241] |
| serde / serde_json | 1.0.229 / 1.0.151 | JSON bodies, config | [VERIFIED: crates.io] |
| thiserror / anyhow | 2.0.21 / 1.0.104 | Errors | [VERIFIED: crates.io] |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| jiff | 0.2.37 | Relative/absolute time formatting in the UI ("3 min ago") | Admin templates. Storage stays INTEGER unix-ms |
| http-body-util | 0.1.5 (dev) | `BodyExt::collect()` in router tests | Tests [VERIFIED: spike] |
| tempfile | 3.27.0 (dev) | Real temp DB files (WAL does not work in `:memory:`) | Store/migration tests |
| dashmap | 6.2.1 (optional) | Key-usage runtime map | Optional. A `std::sync::Mutex<HashMap>` held without `.await` is enough for ≤ ~100 keys |

**Not needed in Phase 1:** reqwest, rustls, rustls-platform-verifier, aws-lc-rs. No outbound HTTP happens in this phase, so keep them out and they arrive with `rsr-transport` in Phase 2. That keeps the first build fast and avoids aws-lc-sys. `subtle` is also unnecessary: lookup is by SHA-256 digest in a `HashMap`, so timing reveals nothing usable about the secret itself [ASSUMED: standard hashed-token reasoning]. Also skip moka, governor, xxhash, zstd, prost, secrecy (Phase 2+) and indexmap (Phase 2).

**Front-end assets (vendored, committed):**

| Asset | Version | Source | sha256 (downloaded this session) |
|-------|---------|--------|------------------------------|
| htmx.min.js | 2.0.11 (npm `latest`; `next` = 4.0.0) | `https://unpkg.com/htmx.org@2.0.11/dist/htmx.min.js` (52,182 B) | `d6fdc75f204e6bdefa99b69bf1e6d4ac69b8a364f77929f45c13476b4000f717` |
| daisyui.mjs | 5.7.46 (released 2026-09-24) | `https://github.com/saadeghi/daisyui/releases/download/v5.7.46/daisyui.mjs` (350,546 B) | `12d17fb603c8a19a3268e268d9037429cf199574b2deb4201c998979ade0dfdc` |
| tailwindcss standalone CLI (tool, **not committed**) | 4.3.3 (released 2026-07-16) | `https://github.com/tailwindlabs/tailwindcss/releases/download/v4.3.3/tailwindcss-linux-x64` (also `-linux-arm64`, `-musl` variants; ~110 MB) | x64: `dc61b3ac6b8c9ca874c0cc4c57b2409791a64c5540404ca5f5367360babc313a` |

htmx-ext-sse 2.2.4 is not needed until Phase 5 (live logs).

**Installation (workspace `Cargo.toml` excerpt, Phase 1 subset):**
```toml
[workspace.dependencies]
tokio         = { version = "1.53", features = ["rt-multi-thread", "macros", "net", "time", "sync", "signal"] }
tokio-util    = { version = "0.7.19", features = ["rt"] }
axum          = { version = "0.8.9", default-features = false, features = ["http1", "http2", "json", "query", "form", "tokio", "matched-path", "original-uri", "tracing"] }
tower         = "0.5.3"
tower-http    = { version = "0.7.1", features = ["trace", "limit", "csrf", "catch-panic", "set-header", "sensitive-headers", "compression-gzip"] }
sqlx          = { version = "0.9", default-features = false, features = ["runtime-tokio", "sqlite-bundled", "migrate", "macros"] }
askama        = "0.16.1"
askama_web    = { version = "0.16", features = ["axum-0.8"] }
rust-embed    = { version = "8.12", features = ["mime-guess", "include-exclude"] }
arc-swap      = "1.9"
clap          = { version = "4.6", features = ["derive", "env"] }
toml          = "1.1"
mimalloc      = { version = "0.1.52", default-features = false }
serde         = { version = "1.0.229", features = ["derive"] }
serde_json    = "1.0.151"
sha2 = "0.11"
base64 = "0.23"
rand = "0.10"
async-trait = "0.1.92"
thiserror = "2.0"
anyhow = "1.0"
jiff = "0.2.37"
tracing = "0.1.44"
tracing-subscriber = { version = "0.3.23", features = ["env-filter", "fmt", "json"] }
# dev
http-body-util = "0.1.5"
tempfile = "3.27"
```
This exact dependency set (plus `subtle`, since dropped) compiled cleanly in 50 s on this host [VERIFIED: spike].

## Package Legitimacy Audit

`gsd-tools query package-legitimacy check --ecosystem crates …` returned **OK** for all 27 crates checked this session: tokio, tokio-util, axum, tower, tower-http, sqlx, askama, askama_web, rust-embed, arc-swap, clap, mimalloc, serde, serde_json, toml, sha2, base64, rand, subtle, async-trait, thiserror, anyhow, tracing, tracing-subscriber, jiff, http-body-util, tempfile. Every one is also prescribed by CLAUDE.md, which was compiled from crates.io registry reads.

| Package | Registry | Age / Last release | Downloads (all-time) | Source Repo | Verdict | Disposition |
|---------|----------|-----|-----------|-------------|---------|-------------|
| tokio | crates.io | 1.53.1, 2026-07-20 | 1.0 B | github.com/tokio-rs/tokio | OK | Approved |
| axum | crates.io | 0.8.9, 2026-04-14 | 491 M | github.com/tokio-rs/axum | OK | Approved |
| tower-http | crates.io | 0.7.1, 2026-08-31 | 482 M | github.com/tower-rs/tower-http | OK | Approved |
| sqlx | crates.io | 0.9.0, 2026-05-21 | 154 M | github.com/launchbadge/sqlx | OK | Approved |
| askama | crates.io | 0.16.1, 2026-09-04 | 51 M | github.com/askama-rs/askama | OK | Approved |
| askama_web | crates.io | 0.16.0, 2026-04-29 | 414 K | github.com/askama-rs/askama_web | OK | Approved (low downloads but owned by askama maintainers; verdict OK) |
| rust-embed | crates.io | 8.12.0, 2026-07-08 | 55 M | pyrossh.dev/repos/rust-embed | OK | Approved |
| arc-swap | crates.io | 1.9.2, 2026-06-28 | 339 M | github.com/vorner/arc-swap | OK | Approved |
| clap | crates.io | 4.6.7, 2026-09-14 | 1.17 B | github.com/clap-rs/clap | OK | Approved |
| toml | crates.io | 1.1.6, 2026-09-10 | 948 M | github.com/toml-rs/toml | OK | Approved |
| mimalloc | crates.io | 0.1.52, 2026-05-22 | 58 M | github.com/purpleprotocol/mimalloc_rust | OK | Approved |
| rand / sha2 / base64 | crates.io | 0.10.3 / 0.11.0 / 0.23.1 | >990 M each | rust-random, RustCrypto, marshallpierce | OK | Approved |
| async-trait / thiserror / anyhow | crates.io | 0.1.92 / 2.0.21 / 1.0.104 | >680 M each | dtolnay | OK | Approved |
| jiff / tempfile / http-body-util | crates.io | 0.2.37 / 3.27.0 / 0.1.5 | 204 M / 842 M / 485 M | BurntSushi, Stebalien, hyperium | OK | Approved |

**Packages removed due to [SLOP] verdict:** none
**Packages flagged as suspicious [SUS]:** none
**npm artifacts** (htmx.org 2.0.11, daisyui 5.7.46) are vendored files, not installed packages. Both are CLAUDE.md-prescribed, and their checksums are recorded above.

## Architecture Patterns

### System Architecture Diagram

```
 operator shell / Docker env / rsrouter.toml
            │ (clap: CLI > RSR_* env > TOML file > defaults)
            ▼
 ┌──────────────────────── rsrouter (bin, composition root) ─────────────────────────┐
 │ tracing init → open data dir → writer pool(1) ─► migrate!() ─► reader pool(2)     │
 │                         │                                                          │
 │                         ▼                                                          │
 │           Storage trait (SqliteStore) ──load──► Snapshot::build ──► ArcSwap<Snapshot>
 │                                                                  ▲        │        │
 │   Components { storage, log_sink, adapter?, codec?, strategy?,   │        │ load() │
 │                cooldown?, quota?, transport? }  (Arc<dyn …>)     │        ▼        │
 └──────────────────────────────────────────────────────────────────┼────────────────┘
                                                                    │
  Admin browser ──HTTP──► :8081 admin router                        │
    Host guard ─(bad Host → 421)                                    │
      └► CsrfLayer ─(cross-site non-GET → 403)                      │
          └► security headers (frame-ancestors none, no-store on reveal)
              └► handlers (askama pages / htmx block partials)      │
                    │ create/rename/disable/delete key              │
                    ▼                                               │
              ControlPlane (tokio Mutex: txn on writer → rebuild ───┘ → store)
                    │ list keys (reader pool) + usage overlay (runtime map)
                    ▼
               HTML page or partial (secret revealed once, snippets)

  AI client (opencode / Claude Code / curl) ──HTTP──► :8080 AI router
    client-auth middleware: x-api-key | Authorization: Bearer → sha256 → snapshot.keys.get()
      ├─ missing/unknown/disabled → 401 in client dialect (Anthropic if anthropic-version or x-api-key present, else OpenAI)
      └─ ok → usage.touch(key_id, now) [memory only] → GET /v1/models → {"object":"list","data":[]} or Anthropic shape
                                              │
  usage flusher task (every ~5 s + on shutdown) ─batch UPDATE COALESCE/MAX─► SQLite (writer)
  SIGINT/SIGTERM → CancellationToken → both servers drain → final usage flush → pools close (WAL checkpoint)
```

### Recommended Project Structure (Phase 1 subset of ARCHITECTURE.md's 9 crates)

```
rsrouter/
├── Cargo.toml                 # [workspace], resolver = "3", [workspace.dependencies], release profile
├── rust-toolchain.toml        # channel = "1.98.1" (ignored locally: no rustup on this host)
├── .sqlx/                     # committed offline query cache (cargo sqlx prepare --workspace)
├── migrations/                # 0001_client_keys.sql (never edit once applied)
├── rsrouter.example.toml      # documented config file
├── tools/                     # gitignored: tailwindcss binary (downloaded by script)
├── scripts/build-css.sh       # downloads pinned tailwind (sha256-checked), builds assets/app.css
├── crates/
│   ├── rsr-core/              # NO internal deps: ids, Dialect, domain (ClientKey), traits/*, errors
│   ├── rsr-store/             # SqliteStore: open(), migrate, repos (impl core::traits::Storage) + build.rs
│   ├── rsr-engine/            # AppState, Snapshot, ControlPlane, KeyUsage runtime + flusher, Components
│   ├── rsr-api/               # AI router, client-auth middleware, /v1/models, dialect error envelopes
│   ├── rsr-admin/             # admin router, layers, pages, templates/, assets/ (rust-embed)
│   └── rsrouter/              # bin: config.rs, wiring.rs, shutdown.rs, main.rs (#[global_allocator])
└── tests/ (or crates/rsrouter/tests/)  # e2e: spawn both routers in-process
```

Do not create rsr-translate, rsr-transport or rsr-providers in Phase 1. Their traits live in `rsr-core`, and the crates arrive with their first real implementation.

### Walking Skeleton (MVP thinnest end-to-end slice)

1. `cargo run -p rsrouter -- --data-dir ./data` starts, logs both listen addresses, creates `data/rsrouter.db` and applies `0001`.
2. `GET http://127.0.0.1:8081/keys` renders a daisyUI page (from the askama layout) listing keys, and includes a "Create key" form.
3. Submitting the form (`hx-post="/keys"`) inserts a row through the ControlPlane, rebuilds and swaps the snapshot, and swaps in a partial that shows the secret once.
4. `curl -H "Authorization: Bearer rsr-…" http://127.0.0.1:8080/v1/models` returns `200 {"object":"list","data":[]}`. The same key via `x-api-key` returns the Anthropic-shaped empty list, and a bad key returns 401 in the matching dialect.
5. Ctrl-C drains, and a restart keeps the key working.

The spike proved every step of this slice.

### Pattern 1: Two listeners, one runtime, one shutdown token
**What:** Bind both listeners before serving. Share `Arc<AppState>`. `try_join!` both servers.
```rust
// Source: spike (compiled + ran), mirrors STACK.md §2
let token = CancellationToken::new();
let api   = axum::serve(TcpListener::bind(cfg.api_listen).await?, api_router(state.clone()))
              .with_graceful_shutdown(token.clone().cancelled_owned());
let admin = axum::serve(TcpListener::bind(cfg.admin_listen).await?, admin_router(state.clone()))
              .with_graceful_shutdown(token.clone().cancelled_owned());
tokio::try_join!(api.into_future(), admin.into_future())?;   // IntoFuture is in the 2024 prelude
// after: usage.flush().await; writer.close().await; reader.close().await;
```
Handle both `ctrl_c()` and `tokio::signal::unix::signal(SignalKind::terminate())`, since Docker sends SIGTERM. Closing the pools checkpoints the WAL. Without that, the spike left a 65 KB `-wal` file after exit [VERIFIED: spike `ls data`]. The OPS-03 hard drain deadline is Phase 11.

Defaults: API `0.0.0.0:8080` (authenticated by client keys), admin `127.0.0.1:8081` (loopback, so exposing it is an explicit choice, as in ARCHITECTURE.md) [ASSUMED: port numbers from the STACK.md Dockerfile `EXPOSE 8080 8081`].

### Pattern 2: SQLite open + migrations (the corrected recipe)
```rust
// Source: spike (ran successfully after fix). Writer FIRST (creates file, sets pragmas, migrates), reader SECOND.
let url = format!("sqlite://{}", data_dir.join("rsrouter.db").display());
let wopts = SqliteConnectOptions::from_str(&url)?
    .create_if_missing(true)
    .journal_mode(SqliteJournalMode::Wal)
    .synchronous(SqliteSynchronous::Normal)
    .busy_timeout(Duration::from_secs(5))
    .foreign_keys(true)
    .auto_vacuum(SqliteAutoVacuum::Incremental);      // applied on connect, before migrate creates tables
let writer = SqlitePoolOptions::new().max_connections(1).connect_with(wopts).await?;
sqlx::migrate!("../../migrations").run(&writer).await?;  // path relative to the crate's CARGO_MANIFEST_DIR
// Reader: SEPARATE options. NO journal_mode / auto_vacuum (they write). WAL persists in the file.
let ropts = SqliteConnectOptions::from_str(&url)?
    .read_only(true).busy_timeout(Duration::from_secs(5)).foreign_keys(true);
let reader = SqlitePoolOptions::new().max_connections(2).connect_with(ropts).await?;
// verified on reader: PRAGMA journal_mode = "wal", PRAGMA auto_vacuum = 2 (INCREMENTAL)
```
- **`build.rs` in the crate that calls `migrate!`** must print `cargo:rerun-if-changed=../../migrations` (the migrations path relative to that crate). Otherwise, a *new* migration file doesn't trigger a rebuild. sqlx docs: "The only solution on stable Rust right now is to create a Cargo build script in your project and have it print `cargo:rerun-if-changed=migrations`" [VERIFIED: sqlx-0.9.0/src/macros/mod.rs:818-826].
- **Downgrade guard for free.** `Migrator` defaults to `ignore_missing: false` [VERIFIED: sqlx-core-0.9.0/src/migrate/migrator.rs:37]. A DB that has a migration the binary lacks fails with `VersionMissing`: `#[error("migration {0} was previously applied but is missing in the resolved migrations")]` [VERIFIED: sqlx-core-0.9.0/src/migrate/error.rs:15-16]. Surface this as a clear startup error ("database is newer than this binary").
- **Never edit an applied migration.** sqlx fails with `VersionMismatch`: `"migration {0} was previously applied but has been modified"` [VERIFIED: error.rs:18-19].
- Optional (Pitfall 12 in PITFALLS.md): before applying pending migrations to a *non-empty* DB, `VACUUM INTO 'rsrouter.db.pre-<version>.bak'`.
- Phase 1 schema = **only** the `client_keys` table. Don't pre-create tables for later phases.

```sql
-- migrations/0001_client_keys.sql  (STRICT verified to work with bundled SQLite)
CREATE TABLE client_keys (
  id            INTEGER PRIMARY KEY,
  name          TEXT    NOT NULL UNIQUE COLLATE NOCASE,
  key_hash      BLOB    NOT NULL UNIQUE,   -- sha256(secret), 32 bytes
  key_prefix    TEXT    NOT NULL,          -- e.g. "rsr-612y" for display
  enabled       INTEGER NOT NULL DEFAULT 1,
  created_at    INTEGER NOT NULL,          -- unix ms
  updated_at    INTEGER NOT NULL,
  first_used_at INTEGER,
  last_used_at  INTEGER
) STRICT;
```
(`COLLATE NOCASE` and `updated_at` are recommendations [ASSUMED]. The rest of the shape ran in the spike.)

### Pattern 3: Config precedence (CLI > env > TOML file > defaults)
```rust
// Source: spike (compiled + ran)
#[derive(clap::Parser)]
struct Cli {
    #[arg(long, env = "RSR_CONFIG")]        config: Option<PathBuf>,
    #[arg(long, env = "RSR_API_LISTEN")]    api_listen: Option<SocketAddr>,
    #[arg(long, env = "RSR_ADMIN_LISTEN")]  admin_listen: Option<SocketAddr>,
    #[arg(long, env = "RSR_DATA_DIR")]      data_dir: Option<PathBuf>,
    #[arg(long, env = "RSR_ADMIN_ALLOWED_HOSTS", value_delimiter = ',')] admin_allowed_hosts: Option<Vec<String>>,
    // + log_level, log_format (pretty|json), api_public_url, log_retention_days, log_max_db_mb, log_store_bodies
}
#[derive(serde::Deserialize, Default)] #[serde(deny_unknown_fields)]
struct FileCfg { /* same fields, all Option<_> */ }
let file: FileCfg = match &cli.config { Some(p) => toml::from_str(&std::fs::read_to_string(p)?)?, None => FileCfg::default() };
let data_dir = cli.data_dir.or(file.data_dir).unwrap_or_else(|| PathBuf::from("./data"));
```
- Make the merge a **pure function** `resolve(cli: Cli, file: FileCfg) -> Config` and unit-test it with `Cli::try_parse_from([...])`. Do not test by mutating env. In edition 2024, `std::env::set_var` is `unsafe` (error E0133 reproduced), and it is racy under multithreaded `cargo test` [VERIFIED: spike].
- Log/retention defaults (FOUND-03) are parsed and validated now, shown read-only on a Settings page, and consumed in Phase 5.
- `api_public_url` (optional) is the base URL clients use (it may sit behind a reverse proxy). The snippet generator uses it. If unset, derive `http://<admin request Host's hostname>:<api port>`.

### Pattern 4: Snapshot + ControlPlane (FOUND-05)
```rust
pub struct Snapshot { pub client_keys: HashMap<[u8; 32], Arc<ClientKeyEntry>> /* id, name, enabled */ }
pub struct AppState { pub snapshot: ArcSwap<Snapshot>, pub usage: KeyUsage, pub components: Components, pub control: ControlPlane }

impl ControlPlane {
    pub async fn create_key(&self, name: &str) -> Result<CreatedKey, ControlError> {
        let _g = self.mutation_lock.lock().await;          // tokio::sync::Mutex<()>: serialize write+rebuild+swap
        let (secret, hash) = gen_key();
        let id = self.storage.insert_client_key(name, &hash, &secret[..8], now_ms()).await?;
        self.rebuild_and_swap().await?;                    // load from storage (writer or post-commit read)
        Ok(CreatedKey { id, secret })                      // secret returned once, never stored
    }
}
```
Keep disabled keys **in** the snapshot with `enabled = false`, so the API can say "disabled" rather than "unknown". In-flight requests keep their old `Arc<Snapshot>`.

### Pattern 5: Client key generation, lookup, dialect-aware rejection (KEYS-01/04)
```rust
// Source: spike (ran: secret length 47, e.g. "rsr-612yyJ6m_lT9uknN62sPNt45GDGnBtaIsqFenXGAsPE")
fn gen_key() -> (String, [u8; 32]) {
    let mut raw = [0u8; 32];
    rand::fill(&mut raw);                                         // ThreadRng = ChaCha12 CSPRNG
    let secret = format!("rsr-{}", base64::engine::general_purpose::URL_SAFE_NO_PAD.encode(raw));
    let hash: [u8; 32] = Sha256::digest(secret.as_bytes()).into();
    (secret, hash)
}
```
Auth middleware (axum `middleware::from_fn_with_state` on the AI router, before handlers):
1. Collect up to two candidates, `x-api-key` and `Authorization: Bearer <t>` (case-insensitive scheme). Claude Code v2.1.248+ sends **both** headers on `/v1/models` discovery [CITED: code.claude.com/docs/en/llm-gateway-protocol#model-discovery]. Accept if any candidate hashes to an **enabled** entry.
2. Choose the dialect: **Anthropic** if an `anthropic-version` header or an `x-api-key` header is present, otherwise **OpenAI**. Phase 2 also uses the route (`/v1/messages` → Anthropic) [ASSUMED: heuristic].
3. On success, `usage.touch(id, now_ms)` and insert `ClientKeyCtx` into request extensions.
4. Rejection bodies, copied from real 401 responses this session [VERIFIED: live probe of api.openai.com and api.anthropic.com, 2026-09-28]:
   - OpenAI, bad key: `{"error":{"message":"Incorrect API key provided: sk-bogus**************arch. …","type":"invalid_request_error","param":null,"code":"invalid_api_key"}}`, HTTP 401
   - OpenAI, missing key: `{"error":{"message":"Missing bearer authentication in header","type":"invalid_request_error","param":null,"code":null}}`, HTTP 401
   - Anthropic, bad key: `{"type":"error","error":{"type":"authentication_error","message":"invalid x-api-key"},"request_id":"req_…"}`, HTTP 401
   - Anthropic, missing key: `{"type":"error","error":{"type":"authentication_error","message":"x-api-key header is required"},"request_id":"req_…"}`, HTTP 401

   Use these shapes with rsrouter wording (e.g. "Invalid rsrouter API key", "rsrouter API key is disabled"). Never echo the presented key beyond a masked prefix.

`GET /v1/models` in Phase 1:
- **OpenAI shape:** `{"object":"list","data":[]}`.
- **Anthropic shape:** `{"data":[],"has_more":false,"first_id":null,"last_id":null}`. Per-model fields `type:"model"`, `id`, `display_name`, `created_at` (RFC 3339) arrive in Phase 2 [CITED: platform.claude.com/docs/en/api/models/list].
- Accept and ignore `?limit=1000`. Claude Code discovery uses a **3 s timeout and treats any redirect as failure** [CITED: code.claude.com/docs/en/llm-gateway-protocol].

### Pattern 6: Admin hardening layers (FOUND-04)
```rust
// Source: spike. Order: last .layer() = outermost → Host guard runs first.
Router::new()
    .route(...)
    .layer(SetResponseHeaderLayer::overriding(header::X_FRAME_OPTIONS, HeaderValue::from_static("DENY")))
    .layer(SetResponseHeaderLayer::overriding(header::CONTENT_SECURITY_POLICY, HeaderValue::from_static(
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; frame-ancestors 'none'; base-uri 'none'; form-action 'self'")))
    .layer(CsrfLayer::new())                                   // tower_http::csrf
    .layer(middleware::from_fn_with_state(st.clone(), host_guard))
    .layer(CatchPanicLayer::new())
```
`CsrfLayer` allows a request if any of these holds [VERIFIED: tower-http-0.7.1/src/csrf/mod.rs:8-15, verbatim]:
```
1. The method is `GET`, `HEAD`, or `OPTIONS`.
2. The `Origin` header byte-for-byte matches an allow-listed trusted origin.
3. `Sec-Fetch-Site` is `same-origin` or `none`.
4. Neither `Sec-Fetch-Site` nor `Origin` is present.
5. The `Origin`'s authority (host and any port) matches the request's effective
   host byte-for-byte (the request-target authority if present, else `Host`).
```
Rejections are `403 Forbidden` with an empty body, and a `ProtectionError` sits in the response extensions. Use `.with_rejection_response(|e| …)` for a friendlier page. Runtime results from the spike:
- cross-site POST → **403**
- same-origin POST → passes CSRF (405 on the GET-only route)
- bare curl POST with neither header → **passes (rule 4, by design)**
- `Host: evil.example:18081` → **421** from the Host guard

**Host guard** (custom, about 10 lines): compare the `Host` header, falling back to `req.uri().authority()` for HTTP/2, case-insensitively against the allowlist. The default allowlist comes from the admin bind: `localhost:<port>`, `127.0.0.1:<port>`, `[::1]:<port>`. When the admin binds `0.0.0.0`/`::`, log a startup **warning** telling the operator to set `admin_allowed_hosts` (e.g. `rsrouter-admin.example.com`, `192.168.1.3:8081`). Reverse proxies (Pangolin/Traefik) pass the public Host by default, so the operator lists the public hostname [ASSUMED: Traefik `passHostHeader` default].

### Pattern 7: askama layout + htmx partials via `blocks` (UI-01, KEYS-02)
```rust
// Source: spike (ran: full page on GET, only <tr> rows when HX-Request: true)
#[derive(Template, WebTemplate)]
#[template(path = "keys.html", blocks = ["rows"])]      // generates page.as_rows()
struct KeysPage { keys: Vec<KeyView> }

async fn keys_page(State(st): State<Arc<AppState>>, headers: HeaderMap) -> Response {
    let page = KeysPage { keys: /* reader pool + usage overlay */ };
    if headers.contains_key("hx-request") { page.as_rows().into_web_template().into_response() }  // askama_web::WebTemplateExt
    else { page.into_response() }
}
```
- Templates live in `crates/rsr-admin/templates/`, which is askama's default dir relative to the crate. Use `{% extends "base.html" %}` with `{% block content %}`. `.html` templates auto-escape: `<script>` becomes `&#60;script&#62;` [VERIFIED: spike test].
- Shell layout: a daisyUI `drawer` (sidebar, collapsible on mobile) + `navbar`, with nav entries for **Dashboard, Providers, Credentials, Models, Combos, Client Keys, Proxies, Quota, Logs, Settings**. Only Client Keys and Settings (read-only config) work in Phase 1. The rest render an "arrives in Phase N" empty state. The UI-SPEC from `/gsd-ui-phase` (enabled, `ui_phase: true`) owns the visual contract.
- htmx CRUD:
  - Create: `hx-post="/keys"` → reveal partial.
  - Rename: `hx-get="/keys/{id}/edit"` → inline form row, then `hx-put="/keys/{id}"` → row partial.
  - Toggle: `hx-post="/keys/{id}/toggle"` → row partial.
  - Delete: `hx-delete="/keys/{id}"` with `hx-confirm` → empty 200 and `hx-swap="outerHTML"`. Don't return 204, because htmx does not swap it.
- **htmx 2 does not swap 4xx/5xx by default.** Default `responseHandling`: `{code:"204", swap:false}`, `{code:"[23]..", swap:true}`, `{code:"[45]..", swap:false, error:true}` [CITED: htmx.org/docs/#response-handling]. For validation errors (duplicate name), either return 200 with an error partial, or add `<meta name="htmx-config">` with a `{"code":"422","swap":true}` entry and return 422.
- Avoid `hx-on:*` and inline `<script>` so a strict CSP can hold. Put behavior (copy buttons, theme toggle) in a small embedded `assets/app.js`.

### Pattern 8: Usage timestamps without request-path DB writes (KEYS-03)
- The runtime map keyed by `ClientKeyId` holds `{ first_seen_ms, last_seen_ms, dirty }` (a `Mutex<HashMap>` without `.await` while held, or DashMap). `touch()` is O(1), memory only.
- A flusher task (`tokio::time::interval(5s)`, `CancellationToken`, final flush on shutdown) takes the dirty entries and runs one transaction on the writer:
  ```sql
  UPDATE client_keys SET first_used_at = COALESCE(first_used_at, ?1),
                         last_used_at  = MAX(COALESCE(last_used_at, 0), ?2) WHERE id = ?3
  ```
  This statement type-checked under `sqlx::query!` against a live DB [VERIFIED: spike].
- The keys page **overlays** the runtime map on DB values, so a fresh `/v1/models` call shows up instantly on the next page render or htmx poll, before the flush. A deleted key's pending update simply affects 0 rows.

### Pattern 9: Trait seams + stub swap (FOUND-07)
- `rsr-core/src/traits/` defines **dyn-compatible** traits with `#[async_trait]`: `Storage` (config repos: client keys now, grown per phase), `LogSink` (sync `record()` + async `flush()`), `ProviderAdapter`, `UpstreamCodec`, `RoutingStrategy`, `CooldownTracker`, `QuotaProber`, `ProxyTransport`.
- Signatures use only core types plus `http`/`bytes`. No reqwest in core.
- Phase 1 keeps the non-storage traits **minimal but real**, for example `RoutingStrategy::order(&self, cands: Vec<Candidate>) -> Vec<Candidate>` and `CooldownTracker::state(&self, cred, model) -> Availability`, and ships `Noop*`/default impls. Later phases extend them. That is cheap inside one workspace.
- `Components { storage: Arc<dyn Storage>, log_sink: Arc<dyn LogSink>, adapter_registry: Arc<dyn …>, … }` lives in `AppState`, and `wiring.rs` is the only place concrete types are chosen.
- **Tests:**
  - Build `AppState` with an in-memory `Storage` stub seeded with one key, and assert `/v1/models` accepts it (storage swap is observable end to end).
  - Use a recording `LogSink` stub, and for each of the other six traits a stub whose calls are counted through the trait object.
  - Native `async fn` in a trait fails `Box<dyn Trait>` with E0038 on rustc 1.98.1, so `#[async_trait]` is required [VERIFIED: spike].

### Anti-Patterns to Avoid
- **Reader pool built from the writer's options** (`opts.read_only(true)`): crashes at startup (Pitfall 1).
- **Querying SQLite in the auth path** "just for last_used": violates FOUND-05/KEYS-03 and contends with the writer.
- **Rebuilding the snapshot outside the mutation lock**: races (Pitfall 3).
- **Returning the secret from a GET or putting it in a URL/query string**: return it only in the POST response body, with `Cache-Control: no-store`.
- **Logging request headers** on the AI router: add `SetSensitiveRequestHeadersLayer` for `authorization` and `x-api-key` before `TraceLayer`.
- **Pre-creating future-phase tables** in `0001`: later changes would need migration edits, which are forbidden.
- **Merging the admin router into the AI router** (even `/healthz`): each listener gets its own.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| CSRF protection | Token tables, double-submit cookies | `tower_http::csrf::CsrfLayer` | Stateless Sec-Fetch-Site/Origin scheme (Go 1.25 parity), tested edge cases (ports, `null` origin, non-UTF8) |
| Migrations | Hand-written `schema_version` table | `sqlx::migrate!()` + `_sqlx_migrations` | Checksums (`VersionMismatch`), downgrade detection (`VersionMissing`), locking |
| Template escaping | Manual HTML string building | askama `.html` templates | Compile-time checked, auto-escape verified |
| Static file embedding + MIME | `include_bytes!` tables | rust-embed `Embed` + `mime-guess` | Debug builds read from disk (hot reload). `sha256_hash()` for ETags |
| Lock-free config reads | `RwLock<Config>` on the hot path | `arc_swap::ArcSwap` | No reader contention, atomic swaps |
| CSPRNG | Seeding your own RNG | `rand::fill` (ThreadRng ChaCha12) or `rand::rngs::SysRng` | OS-seeded, `TryCryptoRng` |
| CSS framework | Hand-written CSS for a "polished" UI | Tailwind 4 standalone + daisyUI 5 | Verified no-Node pipeline, 22 KB minified output |
| Config file parsing | Ad-hoc `key=value` | `toml` + serde `deny_unknown_fields` | Typos fail loudly |

**Key insight:** every Phase 1 capability has a small, verified library answer. The only custom middleware is the Host guard (about 10 lines) and the client-auth layer. Both are simple and must be well tested.

## Common Pitfalls

### Pitfall 1: Read-only pool inherits write pragmas → startup crash (FOUND in spike)
**What goes wrong:** `SqlitePoolOptions::new().connect_with(opts.read_only(true))` (the STACK.md recipe) fails with `Error: error returned from database: (code: 8) attempt to write a readonly database` [VERIFIED: spike run log].
**Why it happens:** sqlx applies `auto_vacuum` and `journal_mode` pragmas on every connect when they are set. sqlx-sqlite-0.9.0/src/options/mod.rs:173-183: "`auto_vacuum` needs to be executed before `journal_mode`, if set." … `pragmas.insert("auto_vacuum".into(), None);` … "Don't set `journal_mode` unless the user requested it." … `pragmas.insert("journal_mode".into(), None);` [VERIFIED: source]. Both write to the DB header.
**How to avoid:** build reader options from scratch: `read_only(true)`, `busy_timeout`, `foreign_keys`. Open the writer and migrate first. WAL mode persists in the file, and the reader reported `journal_mode=wal` [VERIFIED: spike].
**Warning signs:** a startup error mentioning readonly, only on fresh or first connections.

### Pitfall 2: sqlx macros need a DB or a `.sqlx` cache at compile time
**What goes wrong:** without `DATABASE_URL`, every `query!` fails: `set \`DATABASE_URL\` to use query macros online, or run \`cargo sqlx prepare\` to update the query cache` [VERIFIED: spike].
**Resolution logic:** if `DATABASE_URL` is set and non-empty (and not `SQLX_OFFLINE=true`), the macros go online. Otherwise they read `SQLX_OFFLINE_DIR`, then `<crate>/.sqlx`, then `<workspace>/.sqlx` [VERIFIED: sqlx-macros-core-0.9.0/src/query/mod.rs:80-120]. So **with a committed `.sqlx/`, a fresh clone builds with no DB**.
**How to avoid:**
- Wave 0: `cargo install sqlx-cli --no-default-features --features sqlite` (not installed here).
- Dev loop after changing SQL: `export DATABASE_URL=sqlite://$PWD/target/sqlx-dev.db && cargo sqlx database setup --source migrations && cargo sqlx prepare --workspace`, then commit `.sqlx/`.
- Do **not** commit a `.env` with `DATABASE_URL` (fresh clones would go online against a missing DB).
- CI sets `SQLX_OFFLINE=true`.
- Use `as "enabled: bool"` overrides for INTEGER booleans in `query_as!` [VERIFIED: spike].
**Warning signs:** CI passes locally but fails in Docker. `.sqlx` drift after SQL edits. Add a CI step `cargo sqlx prepare --workspace --check`.

### Pitfall 3: Snapshot rebuild race between concurrent admin mutations
**What goes wrong:** mutation A commits and starts rebuilding. B commits, rebuilds and swaps. A's older read then swaps last, and B's new key is missing from the snapshot until the next mutation. That is exactly the "works after restart" bug FOUND-05 forbids.
**How to avoid:** hold one `tokio::sync::Mutex<()>` across write → rebuild → `store` in the ControlPlane. Admin mutations are rare, so serializing them costs nothing. Alternatively, keep a monotonic generation and `compare_and_swap` [ASSUMED: reasoning, no incident reference].
**Warning signs:** flaky "new key rejected" tests under parallel admin requests.

### Pitfall 4: CsrfLayer is not a Host allowlist, and lets non-browser POSTs through
**What goes wrong:** assuming `CsrfLayer` covers FOUND-04. It does no DNS-rebinding defense, and rule 4 passes requests carrying neither `Sec-Fetch-Site` nor `Origin` (e.g. curl) [VERIFIED: source + spike]. Behind a proxy that rewrites `Host` or strips `Origin`, only `Sec-Fetch-Site` still protects (crate doc "Deployment caveat", csrf/mod.rs:25-32).
**How to avoid:** add the separate Host guard (Pattern 6). Test with explicit headers (`Sec-Fetch-Site: cross-site`, or a mismatched `Origin` without `Sec-Fetch-Site`), not bare curl. Document the reverse-proxy requirements.

### Pitfall 5: Clickjacking bypasses CSRF checks
**What goes wrong:** an attacker page iframes the admin UI and tricks a click on "Delete". Requests from inside the frame are genuinely same-origin (`Sec-Fetch-Site: same-origin`), so CSRF checks pass [ASSUMED: standard web-security behavior].
**How to avoid:** `X-Frame-Options: DENY` + CSP `frame-ancestors 'none'` on every admin response.

### Pitfall 6: Copy buttons silently fail on a plain-HTTP LAN admin origin
**What goes wrong:** `navigator.clipboard.writeText` "is available only in secure contexts (HTTPS)" [CITED: developer.mozilla.org/en-US/docs/Web/API/Clipboard/writeText]. `http://127.0.0.1` and `localhost` count as secure, but `http://192.168.1.3:8081` does not, and writing throws `NotAllowedError`.
**How to avoid:** show secrets and snippets in `readonly` `<textarea>`/`<code>` blocks with select-all on focus. The copy button tries `navigator.clipboard`, falls back to selecting the text (and `document.execCommand('copy')`), and shows "Press Ctrl+C" if both fail.

### Pitfall 7: One-time secret leaks through caches, logs or history
**How to avoid:**
- Return the secret only in the POST response partial, with `Cache-Control: no-store`.
- Never log it: no `tracing` field containing it, and `SetSensitiveRequestHeadersLayer` on the AI router.
- Never persist it; store only the hash plus an 8-char prefix.
- Don't place it in `hx-push-url`'d state.

### Pitfall 8: Snippet base URLs are subtly different per client
**What goes wrong:** opencode's `@ai-sdk/openai-compatible` `baseURL` **includes `/v1`** (doc example `"https://api.myprovider.com/v1"`), while Claude Code's `ANTHROPIC_BASE_URL` **excludes `/v1`**, because it calls `"$ANTHROPIC_BASE_URL/v1/messages"` [CITED: opencode.ai/docs/providers, code.claude.com/docs/en/llm-gateway-connect].
**Also:**
- `ANTHROPIC_AUTH_TOKEN` is sent as `Authorization: Bearer` and takes precedence immediately. `ANTHROPIC_API_KEY` is sent as `x-api-key` and requires a one-time interactive approval [CITED: llm-gateway-connect].
- Claude Code's `/v1/models` discovery is **off by default** (`CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1`) and only keeps ids containing `claude` or `anthropic` [CITED: llm-gateway-protocol]. This matters for Phase 2+ model naming. Snippets should mention `ANTHROPIC_MODEL` for non-Claude ids.
- opencode lists models **explicitly** in `opencode.json` (docs don't mention auto-fetching `/v1/models`) [CITED: opencode.ai/docs/providers]. In Phase 1 the snippet has an empty `models` map, and Phase 2 fills it from the snapshot.

### Pitfall 9: Edition-2024 and test-harness traps
- `std::env::set_var` is `unsafe` in edition 2024 (E0133) [VERIFIED: spike]. Test config merging as a pure function.
- WAL is not available in `:memory:`. Use `tempfile::TempDir` DBs.
- `tower::ServiceExt::oneshot` needs `tower` with `features = ["util"]` [VERIFIED: spike].
- In the spike, `pkill -f <binary>` also killed the calling shell. In e2e scripts, kill by PID.

### Pitfall 10: `auto_vacuum` must be set before the first table exists
sqlx applies it on connect, before `migrate!` runs, so the order in Pattern 2 is correct (the reader saw `auto_vacuum=2`) [VERIFIED: spike]. Never add it later expecting it to apply without a `VACUUM`.

## Code Examples

### opencode snippet (UI-05)
```jsonc
// ~/.config/opencode/opencode.json (global) or ./opencode.json (project)
// Source: opencode.ai/docs/providers + opencode.ai/docs/config
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "rsrouter": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "rsrouter",
      "options": {
        "baseURL": "{{ api_public_url }}/v1",
        "apiKey": "{env:RSROUTER_API_KEY}"
      },
      "models": {}
    }
  }
}
```
Pair it with `export RSROUTER_API_KEY=rsr-…`. On the one-time reveal, also offer a variant with the key inline. opencode supports `{env:VAR}` and `{file:path}` substitution and accepts JSONC [CITED: opencode.ai/docs/config].

### Claude Code snippet (UI-05)
```bash
# Source: code.claude.com/docs/en/llm-gateway-connect + llm-gateway-protocol
export ANTHROPIC_BASE_URL={{ api_public_url }}          # no /v1
export ANTHROPIC_AUTH_TOKEN=rsr-…                        # sent as Authorization: Bearer
export CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1      # optional: /model picker lists gateway models
```
```json
// ~/.claude/settings.json
{ "env": { "ANTHROPIC_BASE_URL": "{{ api_public_url }}", "ANTHROPIC_AUTH_TOKEN": "rsr-…" } }
```

### Embedded asset handler
```rust
// Source: spike (served text/javascript). rust-embed-utils: sha256_hash() -> [u8;32], mimetype() -> &str
#[derive(rust_embed::Embed)] #[folder = "assets/"] struct Assets;
async fn asset(Path(p): Path<String>) -> Response {
    match Assets::get(&p) {
        Some(f) => ([(header::CONTENT_TYPE, f.metadata.mimetype().to_string()),
                     (header::ETAG, format!("\"{}\"", hex8(&f.metadata.sha256_hash())))], f.data).into_response(),
        None => StatusCode::NOT_FOUND.into_response(),
    }
}
// route: .route("/assets/{*path}", get(asset))   // axum 0.8 wildcard syntax verified
```
Reference assets as `/assets/app.css?v={{ build_hash }}` with `Cache-Control: public, max-age=31536000, immutable` in release and `no-cache` in debug [ASSUMED: caching policy choice].

### Tailwind + daisyUI (no Node), verified
```css
/* crates/rsr-admin/assets-src/input.css */
@import "tailwindcss" source(none);
@source "../templates";
@plugin "./daisyui.mjs" {
  themes: light --default, dark --prefersdark;
}
```
```bash
tools/tailwindcss -i crates/rsr-admin/assets-src/input.css -o crates/rsr-admin/assets/app.css --minify
# spike output: "/*! 🌼 daisyUI 5.7.46 */ Done in 214ms", 21,999 B, contained .btn .card .navbar .drawer md:grid-cols-3
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `askama_axum` crate | `askama_web` with `axum-0.8` feature + `WebTemplate` derive | askama 0.13+ | Don't search for or add `askama_axum` |
| CSRF tokens | `Sec-Fetch-Site`/`Origin` (tower-http 0.7 `CsrfLayer`) | tower-http 0.7.0 (2026-06-15) | No token plumbing in forms/htmx |
| `sqlx::query(&String)` | `impl SqlSafeStr`: `&'static str` OK, dynamic SQL needs `AssertSqlSafe(..)` | sqlx 0.9.0 | `impl SqlSafeStr for &'static str` [VERIFIED: sqlx-core-0.9.0/src/sql_str.rs:52]; `pub struct AssertSqlSafe<T>(pub T);` [VERIFIED: :74] |
| axum `/:param`, `/*rest` | axum 0.8 `/{param}`, `/{*rest}` | axum 0.8 | Old syntax panics at startup |
| `rand::thread_rng().fill_bytes` | `rand::fill(&mut buf)` / `rand::rng()` / `RngExt` | rand 0.9–0.10 | API renamed; training-data snippets won't compile |
| Tailwind JS config + npm | Tailwind 4 CSS-first `@import`/`@plugin`, standalone binary | Tailwind 4 | No `tailwind.config.js` |

**Deprecated/outdated:** htmx 4.0.0 is on npm `next` (not `latest`). Stay on 2.0.11 per CLAUDE.md [VERIFIED: npm dist-tags].

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Default ports API `0.0.0.0:8080`, admin `127.0.0.1:8081` | Pattern 1 | Low: config values, easy to change |
| A2 | Dialect heuristic: Anthropic if an `anthropic-version` or `x-api-key` header is present | Pattern 5 | Medium: wrong error shape for an unusual client. Refined in Phase 2 by route |
| A3 | Hash-keyed `HashMap` lookup needs no constant-time compare | Standard Stack | Low: the secret is 256-bit and only its digest is compared |
| A4 | The snapshot race needs a mutation mutex (no incident reference; reasoned) | Pitfall 3 | Low: the mutex is cheap insurance either way |
| A5 | Clickjacking passes Sec-Fetch-Site checks because framed requests are same-origin | Pitfall 5 | Low: the mitigation is standard and harmless |
| A6 | Reverse proxies (Pangolin/Traefik) forward the public Host by default | Pattern 6 | Medium: operators may need to list internal hostnames. Document it |
| A7 | CSP `style-src 'unsafe-inline'` is needed for htmx indicator styles | Pattern 6 | Low: tighten later if htmx's `includeIndicatorStyles` is disabled |
| A8 | `COLLATE NOCASE` unique names and an `updated_at` column | Pattern 2 schema | Low: product choice |
| A9 | Asset caching policy (versioned URL + immutable) | Code Examples | Low |
| A10 | `sqlx-cli` 0.9.0 installs cleanly with `--features sqlite` on this host (not attempted; installing tools is outside research scope) | Pitfall 2 | Medium: without it, the macro workflow needs a workaround (live `DATABASE_URL` against a DB created by the binary) |

## Open Questions (RESOLVED)

1. **Should non-browser state-changing admin requests be allowed (curl scripts)?**
   - What we know: `CsrfLayer` allows them (rule 4), and they are not a CSRF vector.
   - **RESOLVED:** Allow them. Optionally require `HX-Request` or `Sec-Fetch-Site` on mutations as defense in depth later.
2. **Disabled key: 401 or 403?**
   - What we know: OpenAI and Anthropic both return 401 for invalid keys. Anthropic uses 403 `permission_error` for permission problems.
   - **RESOLVED:** 401 with a distinct message ("disabled"). Claude Code treats 401 as a credential problem, which is the correct UX.
3. **Where do placeholder nav areas link?**
   - **RESOLVED:** Real routes rendering an empty-state card ("Providers arrive in Phase 2"), so the shell is complete and later phases only fill pages. UI-SPEC confirms the visuals.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| cargo / rustc | everything | ✓ | 1.98.1 (2026-09-01) | — |
| clippy / rustfmt | lint gates | ✓ | 0.1.98 / 1.9.0 | — |
| rustup | `rust-toolchain.toml` enforcement | ✗ | — | Installed toolchain already equals the pinned 1.98.1. The file is harmless and is honored in CI/Docker |
| cc (gcc) | libsqlite3-sys bundled, mimalloc | ✓ | 13.3.0 | — |
| cmake / make / pkg-config | (aws-lc-sys later, not Phase 1) | ✓ | 3.28 / 4.3 / 1.8.1 | — |
| sqlx-cli | `cargo sqlx prepare`, `database setup` | ✗ | — | **Wave 0 install:** `cargo install sqlx-cli --no-default-features --features sqlite` (0.9.0 on crates.io) |
| cargo-nextest | faster tests | ✗ | — | `cargo test` (built in) |
| tailwindcss standalone | CSS build | ✗ (downloadable, verified) | 4.3.3 | Committed `app.css` means `cargo build` never needs it |
| node / npm | not required (only used here for version lookups) | ✓ | 25.6.1 / 11.9.0 | — |
| curl | e2e smoke scripts | ✓ | 8.5.0 | — |
| docker | not needed in Phase 1 (Phase 11) | ✓ | 29.8.1 | — |
| sqlite3 CLI | manual DB inspection | ✗ | — | `sqlx`-based test helpers |
| Network access to crates.io/GitHub | builds, asset download | ✓ | — | — |

**Missing dependencies with no fallback:** none.
**Missing dependencies with fallback:** sqlx-cli (install in Wave 0), nextest (use `cargo test`), tailwind (download script, with CSS committed).

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Rust built-in `cargo test` with `#[tokio::test]` (tokio `macros` + `rt-multi-thread`). Router tests via `tower::ServiceExt::oneshot` + `http_body_util::BodyExt::collect`. Real temp DBs via `tempfile` |
| Config file | none (Wave 0 creates the workspace; no test config needed) |
| Quick run command | `cargo test --workspace --lib` (unit) or `cargo test -p <crate>` |
| Full suite command | `SQLX_OFFLINE=true cargo test --workspace && cargo clippy --workspace --all-targets -- -D warnings && cargo fmt --check` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| FOUND-01 | Both listeners bind to separately configured addrs and serve their own routes. The admin route is 404 on the API port and vice versa | integration (bind `127.0.0.1:0` twice, real TCP via hyper client or `reqwest` dev-dep, or in-process `oneshot` per router) | `cargo test -p rsrouter --test listeners` | ❌ Wave 0 |
| FOUND-02 | Fresh temp dir → DB file created, `_sqlx_migrations` populated. Reopen keeps rows. A DB with an unknown higher migration version → `MigrateError::VersionMissing` | integration (tempfile) | `cargo test -p rsr-store --test migrations` | ❌ Wave 0 |
| FOUND-02 | Reader pool opens read-only on an existing WAL DB (Pitfall 1 regression) | integration | `cargo test -p rsr-store --test pools` | ❌ Wave 0 |
| FOUND-03 | Precedence CLI > env-derived value > TOML > default. Unknown TOML key rejected | unit (pure `resolve()`, `Cli::try_parse_from`) | `cargo test -p rsrouter config::` | ❌ Wave 0 |
| FOUND-04 | POST with `Sec-Fetch-Site: cross-site` → 403. POST with mismatched `Origin`, no Sec-Fetch-Site → 403. Same-origin POST → 2xx. Bad `Host` → 421. `X-Frame-Options: DENY` present | integration (`oneshot` on admin router) | `cargo test -p rsr-admin --test hardening` | ❌ Wave 0 |
| FOUND-05 | Create key via ControlPlane, then the API accepts it with no restart. Disable, then rejected immediately. **Close both DB pools, then `/v1/models` with a valid key still → 200** (proves no per-request DB read) | integration | `cargo test -p rsr-engine --test hot_config` | ❌ Wave 0 |
| FOUND-05 | Concurrent create_key ×20 → all 20 keys present in the final snapshot (race regression) | integration | `cargo test -p rsr-engine --test hot_config concurrent` | ❌ Wave 0 |
| FOUND-07 | For each of the 8 traits, a stub implementation is plugged into `Components` and invoked through `Arc<dyn …>`. The storage stub drives `/v1/models` auth end to end | unit + integration | `cargo test -p rsr-engine --test component_swap` | ❌ Wave 0 |
| KEYS-01 | Create returns a secret matching `^rsr-[A-Za-z0-9_-]{43}$`. The DB stores the 32-byte hash, not the secret (grep the DB bytes for the secret → absent). The reveal response has `Cache-Control: no-store`. The list page never contains the secret | integration | `cargo test -p rsr-admin --test keys create` | ❌ Wave 0 |
| KEYS-02 | Rename (duplicate name → error partial), disable/enable toggle, delete (then API rejects) | integration (`oneshot` with htmx headers) | `cargo test -p rsr-admin --test keys` | ❌ Wave 0 |
| KEYS-03 | `/v1/models` call → overlay shows last_used immediately. After `flush()`, DB has first_used (unchanged on a 2nd call) and last_used (advanced) | integration | `cargo test -p rsr-engine --test usage` | ❌ Wave 0 |
| KEYS-04 | Bearer ok. x-api-key ok. Both headers present (Claude Code style) ok. Unknown → 401 OpenAI shape. Unknown with `x-api-key` → 401 Anthropic shape (`type:"error"`, `error.type:"authentication_error"`). Disabled → 401. Missing → 401. Anthropic-shape list when `anthropic-version` is present | integration (`oneshot` on API router) | `cargo test -p rsr-api --test auth` | ❌ Wave 0 |
| UI-01 | Shell renders with nav links for all 10 areas. Every nav route returns 200. HTML escapes user-supplied key names | integration + snapshot of nav | `cargo test -p rsr-admin --test shell` | ❌ Wave 0 |
| UI-01 | Visual polish / responsiveness | manual (browser at desktop + mobile width) | UAT in `/gsd-verify-work` | manual-only: visual judgment |
| UI-05 | Reveal partial contains an opencode JSON with `baseURL` ending `/v1` and a Claude Code snippet with `ANTHROPIC_BASE_URL` without `/v1`, both containing the new secret. `api_public_url` override respected | integration | `cargo test -p rsr-admin --test keys snippets` | ❌ Wave 0 |
| UI-05 | Real clients: `curl` with the snippet values → `/v1/models` 200. Optionally a real opencode / Claude Code smoke | manual smoke | documented curl commands in VERIFICATION | manual |

### Sampling Rate
- **Per task commit:** `cargo test -p <touched crate>` (< 30 s after the first build).
- **Per wave merge:** `SQLX_OFFLINE=true cargo test --workspace` + clippy `-D warnings` + `cargo fmt --check` + `cargo sqlx prepare --workspace --check`.
- **Phase gate:** full suite green + manual UI walkthrough before `/gsd-verify-work`.

### Wave 0 Gaps
- [ ] Cargo workspace (`Cargo.toml`, `rust-toolchain.toml`, crate skeletons, `[workspace.dependencies]`, `[profile.release]` with `panic = "unwind"`)
- [ ] `cargo install sqlx-cli --no-default-features --features sqlite` + dev DB setup + first `cargo sqlx prepare --workspace` + committed `.sqlx/`
- [ ] `crates/rsr-store/build.rs` (`cargo:rerun-if-changed=../../migrations`)
- [ ] Shared test helpers: `test_state()` (TempDir DB + migrated pools + AppState), `oneshot_json()`, header builders for CSRF/Host cases
- [ ] `scripts/build-css.sh` (pinned tailwind download + sha256 check) + vendored `htmx.min.js` + `daisyui.mjs` + committed `app.css`
- [ ] `.gitignore`: `tools/`, `data/`, `target/`, `*.db*` (but NOT `.sqlx/`)

## Security Domain

`security_enforcement: true`, ASVS level 1, block on high.

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | yes (AI API client keys) | 256-bit random keys (`rand::fill`), SHA-256 digest storage, shown once, accepted via Bearer or `x-api-key` |
| V3 Session Management | no | No sessions: admin is unauthenticated by design (PROJECT.md Out of Scope), and the API is stateless per key |
| V4 Access Control | yes | Admin: loopback default bind + Host allowlist + `CsrfLayer` + frame denial. API: every `/v1/*` route behind the client-auth middleware (only `/healthz` exempt) |
| V5 Input Validation | yes | Key name: trimmed, 1–64 chars, allowed charset, unique (DB constraint → friendly error). Typed axum `Form`/`Path` extractors. TOML `deny_unknown_fields`. askama auto-escaping |
| V6 Cryptography | yes | CSPRNG from `rand` 0.10 (ChaCha12 ThreadRng), `sha2` 0.11. Nothing hand-rolled |
| V7 Error Handling & Logging | yes | Dialect-shaped 401s with no key echo. `SetSensitiveRequestHeadersLayer` (`authorization`, `x-api-key`) before `TraceLayer`. Secrets never in tracing fields. `CatchPanicLayer` |
| V8 Data Protection | yes | Secret never stored or logged. `Cache-Control: no-store` on the reveal response. Only an 8-char prefix is displayed afterwards |
| V14 Configuration | yes | Security headers (XFO DENY, CSP, `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`) on admin. Warning when admin binds to a non-loopback address without an explicit allowlist |

### Known Threat Patterns for axum + htmx admin on a LAN

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Cross-site form POST to the admin port (CSRF) | Tampering | `CsrfLayer` (Sec-Fetch-Site/Origin) |
| DNS rebinding → attacker JS reads admin pages | Information disclosure / Tampering | Host header allowlist → 421 |
| Clickjacking the admin UI | Tampering | `X-Frame-Options: DENY` + CSP `frame-ancestors 'none'` |
| Stored XSS via key name | Tampering / Elevation | askama auto-escape (verified) + CSP `script-src 'self'`, no inline scripts / `hx-on` |
| Client key brute force / guessing | Spoofing | 256-bit keys make guessing infeasible. Rate limiting is a Phase 10 concern |
| Secret leakage via logs, caches or history | Information disclosure | Sensitive-header layer, `no-store`, reveal only in the POST body, hash-only storage |
| Timing side channel on key comparison | Information disclosure | Digest-keyed lookup, since the attacker cannot relate digest timing to secret bytes (A3) |
| Admin exposed to the internet unintentionally | Elevation | Loopback default + startup warning + docs (Phase 11 OPS-05) |

## Sources

### Primary (HIGH confidence)
- **Spike built and run this session** (scratchpad `spike/`): the full dependency set above compiled on rustc 1.98.1. Runtime checks: Bearer/x-api-key 200, bad key 401, Host guard 421, cross-site POST 403, htmx block partial, asset MIME, restart persistence, graceful SIGINT. Test checks: `query_as!`/`query!` with a live DATABASE_URL, E0038 for native AFIT dyn, `oneshot` + CsrfLayer test, askama escaping, E0133 `set_var`.
- Crate sources read in `~/.cargo/registry/src`: `tower-http-0.7.1/src/csrf/{mod,layer,response}.rs`, `sqlx-sqlite-0.9.0/src/options/mod.rs`, `sqlx-core-0.9.0/src/{sql_str.rs,migrate/migrator.rs,migrate/error.rs}`, `sqlx-0.9.0/src/macros/mod.rs`, `sqlx-macros-core-0.9.0/src/{query/mod.rs,migrate.rs}`, `askama_derive-0.16.1/src/{input,generator}.rs`, `askama_web-0.16.0/src/lib.rs` + Cargo features, `rand-0.10.3/src/{lib.rs,rngs/thread.rs}`, `rust-embed-utils-8.12.0/src/lib.rs`.
- crates.io API (2026-09-28): versions, dates, downloads and repos for every crate listed. The GSD package-legitimacy seam returned OK for all 27.
- Live API probes (2026-09-28): api.openai.com `/v1/models` and api.anthropic.com `/v1/models` 401 bodies.
- npm registry dist-tags (htmx.org, htmx-ext-sse, daisyui, tailwindcss). GitHub release APIs (tailwindcss v4.3.3 assets, daisyui v5.7.46 assets). The Tailwind standalone + daisyUI plugin build was executed.

### Secondary (MEDIUM confidence)
- code.claude.com/docs/en/llm-gateway-connect and /llm-gateway-protocol: env vars, header mapping, model discovery (3 s timeout, no redirects, both headers, id filter).
- platform.claude.com/docs/en/api/errors and /api/models/list: error envelope, models list shape.
- opencode.ai/docs/providers and /docs/config: custom provider JSON, config locations, `{env:}` substitution.
- htmx.org/docs (response handling defaults). MDN Clipboard.writeText (secure-context requirement).
- Project research: `.planning/research/{STACK,ARCHITECTURE,PITFALLS,SUMMARY}.md` (2026-09-28).

### Tertiary (LOW confidence)
- None relied upon. Everything that is only reasoned is in the Assumptions Log.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH. Versions come from crates.io today, and the whole set compiled and ran.
- Architecture: HIGH. The skeleton, layer order, snapshot swap and auth path ran end to end in the spike. Crate split and trait shapes are MEDIUM (design choices).
- Pitfalls: HIGH for 1, 2, 4, 9 and 10 (reproduced or read in source). MEDIUM for 3 and 5 (reasoned). MEDIUM for 6 and 8 (official docs).

**Research date:** 2026-09-28
**Valid until:** 2026-10-28 (stable crates). Re-check Claude Code gateway docs sooner, since they change with CLI releases.
