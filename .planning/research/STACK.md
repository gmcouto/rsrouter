# Stack Research

**Domain:** Self-hosted AI API router/gateway (OpenAI + Anthropic compatible), Rust, single static binary, low memory
**Researched:** 2026-09-28
**Confidence:** HIGH for core crate choices and versions. MEDIUM for UI tooling and build pipeline details.

> **How confidence was assigned.** Context7 and similar doc MCPs were not available here. The `classify-confidence` seam rates every provider this session could reach (webfetch, websearch, registry) as **LOW** by default, and the research-store digests carry that tag. Every version number below was read from the **crates.io registry API** on 2026-09-28. Feature flags and API claims for reqwest, sqlx, axum, tower-http, and libmimalloc-sys were checked against the **published `.crate` source tarballs**, which are primary sources. The per-row labels mean:
> - **HIGH**: checked against registry metadata or crate source.
> - **MEDIUM**: taken from official docs, changelogs, or announcements, or it is a design judgment built on HIGH facts.
> - **LOW**: community reports only.

---

## TL;DR (prescriptive)

| Concern | Use | Not |
|---|---|---|
| Async runtime | **tokio 1.53** (multi-thread) | async-std (dead), smol |
| HTTP server | **axum 0.8.9** + tower 0.5 + **tower-http 0.7.1** | actix-web, poem, salvo |
| HTTP client | **reqwest 0.13.5** (rustls/aws-lc, http2, socks, stream), **one `Client` per proxy** in an `ArcSwap` map | raw hyper-util, isahc, ureq |
| TLS | **rustls 0.23** + aws-lc-rs provider (reqwest default), one shared `ClientConfig` | native-tls/OpenSSL |
| Database | **SQLite via sqlx 0.9** (`sqlite-bundled`), `sqlx::migrate!()` embedded, one writer pool + one read pool, WAL | sea-orm, diesel, mixing rusqlite with sqlx |
| Templates | **askama 0.16** + **askama_web 0.16** (`axum-0.8`) | maud (no Jinja-style files), minijinja (runtime errors) |
| Front-end | **htmx 2.0.x + htmx-ext-sse 2.2.x**, Tailwind CSS 4 standalone CLI + daisyUI 5, all embedded with **rust-embed 8** | React/WASM (Leptos, Dioxus), Node build chains |
| Logging/diagnostics | **tracing 0.1 + tracing-subscriber 0.3** (env-filter, json) | log/env_logger |
| Request logs (domain) | Explicit `mpsc` then a batched SQLite writer, plus `broadcast` for live SSE | Routing domain logs through a tracing layer |
| Allocator | **mimalloc 0.1.52** (v3 is the default) | jemalloc (breaks on Raspberry Pi 5 16K-page kernels), musl malloc |
| Background jobs | Plain `tokio::spawn` loops + `tokio::time::interval` + `CancellationToken`/`TaskTracker` | tokio-cron-scheduler, apalis |
| OAuth | Hand-rolled on reqwest + sha2 + base64 + rand (PKCE is about 20 lines) | `oauth2` 5.0 (pulls reqwest 0.12, rand 0.8, sha2 0.10, thiserror 1) |
| Builds | **`x86_64/aarch64-unknown-linux-musl`** static binaries via **cargo-zigbuild**, Docker `FROM gcr.io/distroless/static-debian13:nonroot` | Compiling Rust under QEMU in buildx, glibc `+crt-static` |

---

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended | Conf. |
|---|---|---|---|---|
| Rust toolchain | stable **1.98.1** (edition 2024); set `rust-version = "1.94"` | Language | 1.94 is the highest MSRV in the tree (sqlx 0.9, sea-orm 2 if ever added). Pin the toolchain in `rust-toolchain.toml` so CI, Docker, and local builds agree. | HIGH |
| tokio | **1.53.1** | Async runtime | The ecosystem standard. axum, reqwest, sqlx, and hyper all run on it natively, so one runtime and one I/O driver handle both listeners and all outbound streams. | HIGH |
| axum | **0.8.9** (no 0.9 exists yet) | AI API listener + admin listener | Built on hyper 1 + tower, the same stack reqwest uses, so the server and client share one HTTP implementation (smaller binary, one set of bugs). Tower middleware (timeouts, CSRF, tracing, body limits) composes per router, which fits two routers on two ports. Handlers returning `Body::from_stream` give zero-copy SSE passthrough. | HIGH |
| tower-http | **0.7.1** | Middleware | 0.7.0 (2026-06-15) added **`CsrfLayer`** (Sec-Fetch-Site/Origin based, no tokens). That matters here because the admin UI has **no auth by design**, so without it any web page the admin visits could POST to the admin port. Also used: `on-early-drop` (log client disconnects mid-stream), `DeadlineBody`, `TraceLayer`, `RequestBodyLimitLayer`, `CatchPanic`, `SensitiveHeaders`. The compression `DefaultPredicate` already skips `text/event-stream` (verified in source). It works with axum 0.8, whose tower-http dependency is only an optional/private one. | HIGH |
| reqwest | **0.13.5** | Upstream HTTP client | Mature pooling, HTTP/2 via ALPN, streaming bodies, `read_timeout` (per-read idle timeout, which is right for LLM streams), and HTTP/HTTPS/**SOCKS4/5/5h** proxies with auth built in (the `socks` feature is now a flag over hyper-util's connectors, with no extra C deps). 0.13 made rustls the default. **Proxies are set per `Client`, not per request (confirmed in source), so keep one Client per proxy** (pattern below). | HIGH |
| rustls | **0.23.45** (aws-lc-rs provider) | TLS for outbound | Pure-Rust TLS with no OpenSSL, so musl static builds stay simple. aws-lc-rs is reqwest 0.13's and rustls' default provider. Non-FIPS aws-lc-sys needs only a C compiler (no CMake, no Go), which zig supplies. Build **one** `rustls::ClientConfig` and pass it to every per-proxy client with `tls_backend_preconfigured` (see Pitfalls). | HIGH |
| rustls-platform-verifier | **0.7.1** (reqwest accepts `>=0.6,<0.8`) | Cert verification | reqwest's default. It reads the OS trust store (`/etc/ssl/certs`), so corporate MITM CAs work. The distroless static image ships a CA bundle. | HIGH |
| sqlx | **0.9.0** (2026-05-21, MSRV 1.94) | SQLite access + migrations | Async pool on tokio, `#[derive(FromRow)]`, **compile-time-checked `query!`/`query_as!`** (useful when much of the SQL is agent-written), and **`sqlx::migrate!()` embeds `migrations/*.sql` into the binary** so they run at startup. SQLite is bundled (`sqlite-bundled`), so there is no system libsqlite and the build is fully static. | HIGH |
| askama | **0.16.1** | Server-side templates | Templates compile to Rust at build time, so they are type-checked against their structs, render fast, and add no runtime template files. The Jinja syntax suits a large, polished UI better than a Rust DSL. `#[template(path = "...", block = "row")]` renders a single block, which is what htmx partial swaps need. | HIGH |
| askama_web | **0.16.0**, feature `axum-0.8` | axum `IntoResponse` for templates | The supported replacement for the removed `askama_axum`. Owned by the askama maintainers. Use `#[derive(Template, WebTemplate)]`. | HIGH |
| htmx | **2.0.11** (npm `latest`) + **htmx-ext-sse 2.2.4** | UI interactivity + live logs | See the htmx 2 vs 4 decision below. The 2.x line is supported indefinitely, and every tutorial and every code-generating agent writes 2.x idioms. | MEDIUM |
| mimalloc | **0.1.52** (libmimalloc-sys 0.1.49 builds **mimalloc v3** by default) | Global allocator | Required on musl, whose allocator is slow and contends under many threads. v3 returns memory to the OS better than v2 (lower steady RSS on 1 GB VMs). It detects page size **at runtime**, so it works on Raspberry Pi 5 16K-page kernels, where jemalloc aborts. | HIGH (feature facts), MEDIUM (RSS claims) |
| tracing / tracing-subscriber | **0.1.44 / 0.3.23** | Diagnostics | Standard. axum, tower-http, reqwest, and sqlx all emit tracing spans. Use `env-filter` + `fmt` (pretty in dev, `json` in Docker). | HIGH |

### Supporting Libraries

| Library | Version | Purpose | When to Use | Conf. |
|---|---|---|---|---|
| tokio-util | 0.7.19 (`rt`) | `CancellationToken`, `TaskTracker` | Graceful shutdown of both listeners and all background loops | HIGH |
| tokio-stream | 0.1.19 (`sync`) | `BroadcastStream` | Turning the live-log `broadcast::Receiver` into an SSE stream (handle `Lagged`) | HIGH |
| futures-util | 0.3.34 | `StreamExt`, `stream::once().chain()` | Prime-then-commit: re-emit buffered first chunks, then the rest of the upstream stream | HIGH |
| http / http-body-util / bytes | 1.x / 0.1.5 / 1.12.1 | Shared HTTP types | Adapter signatures (`http::Request<reqwest::Body>`) and zero-copy chunk forwarding | HIGH |
| serde / serde_json | 1.0.229 / 1.0.151 (`raw_value`) | JSON | **Passthrough:** deserialize the top level into `IndexMap<String, Box<RawValue>>`, rewrite `model`, re-serialize. Nested content stays byte-identical. Full typed structs only on translation paths. | HIGH |
| indexmap | 2.14.2 (`serde`) | Order-preserving top-level JSON map | Keeps field order across rewrites. Needed for the RawValue passthrough pattern. | HIGH |
| arc-swap | 1.9.2 | Lock-free config snapshot | `ArcSwap<Snapshot>` holds providers, credentials, combos, client keys, and the proxy → Client map. The request path never touches SQLite for config. | HIGH |
| dashmap | 6.2.1 | Concurrent maps | Cooldown tracker, per-credential state. (`parking_lot::RwLock<HashMap>` is also fine.) | MEDIUM |
| moka | 0.12.16 (`sync`) | Bounded TTL cache | Cache-affinity map (fingerprint → credential+model), with TTL and max entries so memory stays bounded | MEDIUM |
| governor | 0.10.4 | GCRA rate limiter | User-configured RPM, and TPM via `check_n(tokens)`. Pair with `tokio::sync::Semaphore` for concurrency caps. | MEDIUM |
| xxhash-rust | 0.8.19 (`xxh3`) | Fast non-crypto hashing | Prefix fingerprint (system + tools + first N messages, hashed over **raw bytes**). Collisions stay within one client key's scope, so no crypto hash is needed. | MEDIUM |
| sha2 / base64 / rand | 0.11.0 / 0.23.1 / 0.10.3 | Client API key hashing, PKCE, key generation | Client keys: 32 random bytes → `rsr-…` base64url. Store **SHA-256** for O(1) lookup (not argon2: keys are high-entropy and hashed on every request). PKCE: `S256(verifier)`. | HIGH |
| secrecy | 0.10.3 | `SecretString` | Wrap upstream API keys and OAuth tokens so `Debug`/tracing never prints them | MEDIUM |
| uuid | 1.26.1 (`v7`) | Request/attempt IDs | Time-sortable IDs for log correlation. Keep SQLite PKs as `INTEGER PRIMARY KEY`. | MEDIUM |
| jiff | 0.2.37 | Time formatting | Store timestamps as `INTEGER` unix-ms in SQLite (compact, no sqlx type feature needed). Use jiff only for UI formatting and timezones. | MEDIUM |
| url | 2.5.8 | URL handling | Provider base URLs, OAuth callback parsing (user pastes the callback URL) | HIGH |
| clap | 4.6.7 (`derive`, `env`) | Bootstrap config | `--api-listen`, `--admin-listen`, `--db`, `--log-level`, each with env fallback (`RSR_*`). All other settings live in SQLite. Skip figment (last release 2024) and config-rs. | HIGH |
| thiserror / anyhow | 2.0.21 / 1.0.104 | Errors | thiserror in library crates (typed `Failure` classification drives cooldowns). anyhow only in `main`. | HIGH |
| zstd | 0.14.0 | Log body compression | Compress stored request/response bodies (JSON compresses roughly 5–10×). This stops the SQLite bloat OmniRoute hit (1.4 GB DB, 3–8 GB RSS). | MEDIUM |
| rust-embed | 8.12.0 (`mime-guess`, `include-exclude`) | Embed `assets/` into the binary | Serve with a tiny handler that sets `ETag` (the embedded hash) and `Cache-Control: immutable` for hashed filenames. In debug builds rust-embed reads from disk, so assets hot-reload. | HIGH |
| prost | 0.14.4 | Protobuf | **Cursor only** (Connect-RPC `application/connect+proto` over HTTP/2 bidi). Hand-written `#[derive(Message)]` structs mean no `protoc` at build time. Put it behind a cargo feature. | MEDIUM |

### Development Tools

| Tool | Version | Purpose | Notes |
|---|---|---|---|
| sqlx-cli | 0.9.0 | `sqlx migrate add/run`, `cargo sqlx prepare` | `cargo install sqlx-cli --no-default-features --features sqlite`. Commit the `.sqlx/` offline cache. CI and Docker build with `SQLX_OFFLINE=true`. |
| cargo-zigbuild | 0.23.4 (+ zig pinned) | Cross-compile musl amd64 + arm64 from one host | Use the `ghcr.io/rust-cross/cargo-zigbuild` image, or pin a zig version known to work with it. Zig 0.16.0 is current. Check compatibility before bumping. |
| cargo-nextest | 0.9.146 | Test runner | Faster, isolated test processes. Good for the large golden-test suites the translators need. |
| insta | 1.48.0 (`json`, `redactions`) | Snapshot tests | Golden fixtures for OpenAI↔IR↔Anthropic request, response, and stream translation. |
| tempfile | 3.27.0 | Temp SQLite DBs | Migration and repository tests on real files (WAL does not behave the same in `:memory:`) |
| cargo-deny | 0.20.2 | Licenses, advisories, **duplicate versions** | Fail CI on duplicate `reqwest`/`rustls`/`rand` majors (catches crates like `oauth2` 5.0 sneaking in old stacks) |
| cargo-auditable | 0.7.6 | Embed dependency list in the binary | Lets `cargo audit bin` / Trivy scan the shipped image |
| bacon | 3.26.0 | Dev loop | `bacon run` for check/clippy/test watch |
| Tailwind CSS standalone CLI | 4.3.3 | CSS build **without Node** | Single binary for linux-x64/arm64. Scans `templates/**/*.html`. |
| daisyUI | 5.7.46 (as the standalone `daisyui.mjs` plugin file) | Component classes for a polished UI | Loaded via `@plugin "./daisyui.mjs"` in the Tailwind input CSS. No npm needed. |

Fake upstreams in integration tests should be **small axum servers inside the test**, not wiremock. They need real chunk timing (delayed SSE events, mid-stream resets, 200-then-`event: error`) to exercise prime-then-commit, and wiremock returns whole bodies.

---

## Installation

Use a cargo workspace (matching ARCHITECTURE.md's crates) with versions pinned once in `[workspace.dependencies]`:

```toml
# Cargo.toml (workspace root)
[workspace]
resolver = "3"
members = ["crates/*", "rsrouter"]

[workspace.package]
edition = "2024"
rust-version = "1.94"

[workspace.dependencies]
# runtime & server
tokio         = { version = "1.53", features = ["rt-multi-thread", "macros", "net", "time", "sync", "signal", "fs"] }
tokio-util    = { version = "0.7.19", features = ["rt"] }
tokio-stream  = { version = "0.1.19", features = ["sync"] }
futures-util  = "0.3.34"
axum          = { version = "0.8.9", default-features = false, features = ["http1", "http2", "json", "query", "form", "tokio", "matched-path", "original-uri", "tracing"] }
tower         = "0.5.3"
tower-http    = { version = "0.7.1", features = ["trace", "timeout", "limit", "csrf", "request-id", "set-header", "sensitive-headers", "catch-panic", "on-early-drop", "compression-gzip", "compression-br"] }
http          = "1"
http-body-util = "0.1.5"
bytes         = "1.12"

# outbound
reqwest  = { version = "0.13.5", default-features = false, features = ["rustls", "http2", "socks", "stream", "json", "gzip", "brotli", "zstd"] }  # NO "system-proxy": proxying is explicit per client
rustls   = { version = "0.23.45", default-features = false, features = ["aws_lc_rs", "std", "tls12", "logging", "prefer-post-quantum"] }
rustls-platform-verifier = "0.7.1"

# storage
sqlx = { version = "0.9", default-features = false, features = ["runtime-tokio", "sqlite-bundled", "migrate", "macros", "json"] }

# ui
askama     = "0.16.1"
askama_web = { version = "0.16", features = ["axum-0.8"] }
rust-embed = { version = "8.12", features = ["mime-guess", "include-exclude"] }

# data & util
serde      = { version = "1.0.229", features = ["derive"] }
serde_json = { version = "1.0.151", features = ["raw_value"] }
indexmap   = { version = "2.14", features = ["serde"] }
arc-swap   = "1.9"
dashmap    = "6.2"
moka       = { version = "0.12.16", features = ["sync"] }
governor   = "0.10.4"
xxhash-rust = { version = "0.8.19", features = ["xxh3"] }
sha2 = "0.11"
base64 = "0.23"
rand = "0.10"
secrecy = "0.10"
uuid = { version = "1.26", features = ["v7"] }
jiff = "0.2.37"
url = "2.5"
zstd = "0.14"
prost = "0.14.4"          # rsr-providers, behind feature "cursor"
clap = { version = "4.6", features = ["derive", "env"] }
thiserror = "2.0"
anyhow = "1.0"
tracing = "0.1.44"
tracing-subscriber = { version = "0.3.23", features = ["env-filter", "fmt", "json"] }
mimalloc = { version = "0.1.52", default-features = false }

# dev
insta = { version = "1.48", features = ["json", "redactions"] }
tempfile = "3.27"

[profile.release]
opt-level = 3
lto = "fat"
codegen-units = 1
strip = "symbols"
panic = "unwind"      # NOT "abort": one panicking request task must not kill the router
```

```rust
// rsrouter/src/main.rs
#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;

fn main() -> anyhow::Result<()> {
    // Must happen before any reqwest/rustls use: makes provider choice explicit and
    // avoids "no process-level CryptoProvider" panics if feature unification ever
    // enables both aws-lc-rs and ring.
    rustls::crypto::aws_lc_rs::default_provider().install_default().ok();
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .max_blocking_threads(32)   // default 512 is too high for 1 GB hosts (DNS getaddrinfo + sqlx use blocking threads)
        .build()?
        .block_on(run())
}
```

```bash
# Tooling
cargo install sqlx-cli --no-default-features --features sqlite
cargo install cargo-nextest cargo-deny cargo-zigbuild cargo-auditable bacon
rustup target add x86_64-unknown-linux-musl aarch64-unknown-linux-musl
# CSS (no Node): download tailwindcss-linux-{x64,arm64} v4.3.3 + daisyui.mjs into tools/
```

---

## Key Patterns the Stack Implies

### 1. Per-proxy `reqwest::Client` cache (answers "can reqwest do per-request proxies cheaply?")

**Yes, if you use one Client per proxy.** reqwest has no `RequestBuilder::proxy()`. The `Proxy` is fixed at `ClientBuilder` time (checked in 0.13.5 source). This is also the *correct* design regardless of library: hyper-util's pool keys connections by destination `(scheme, authority)`, so a single pool shared across proxies would reuse a connection opened through proxy A for a request meant for proxy B. Separate Clients give separate pools, which is what you want.

The cost is small if you do three things:
1. **Share one `rustls::ClientConfig`** across all clients via `ClientBuilder::tls_backend_preconfigured(cfg.clone())`. Otherwise each `build()` creates its own platform verifier and loads the root store again, once per proxy. `ClientConfig` clones share the `Arc`'d verifier. **You must set `cfg.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1".to_vec()]` yourself.** reqwest only sets ALPN on configs it builds itself (verified in source), so without this HTTP/2 to upstreams is silently lost.
2. **Cap idle pools**: `pool_max_idle_per_host(4..8)` (the default is unlimited) and `pool_idle_timeout(60s)`.
3. **Build lazily and store clients in the `ArcSwap` snapshot** keyed by `ProxyId` (plus one `direct` client). Rebuild only when the proxy row changes. Dropping the old Client closes its pool.

```rust
fn build_client(tls: &rustls::ClientConfig, proxy: Option<&ProxyRow>) -> reqwest::Result<reqwest::Client> {
    let mut b = reqwest::Client::builder()
        .tls_backend_preconfigured(tls.clone())
        .connect_timeout(Duration::from_secs(10))
        .read_timeout(Duration::from_secs(300))      // idle-between-chunks, NOT total; never use .timeout() on streams
        .pool_max_idle_per_host(8)
        .pool_idle_timeout(Duration::from_secs(60))
        .tcp_keepalive(Duration::from_secs(30))
        .http2_keep_alive_interval(Duration::from_secs(30))
        .http2_keep_alive_while_idle(true);
    b = match proxy {
        // "http://", "https://", "socks5://" (local DNS), "socks5h://" (DNS via proxy); user:pass@ supported
        Some(p) => b.proxy(reqwest::Proxy::all(&p.url)?),
        None => b.no_proxy(),                         // ignore HTTP(S)_PROXY env: routing must be explicit
    };
    b.build()
}
```

Memory per idle Client (estimate): a few KB plus live connections, since each TLS connection costs about 30–60 KB of buffers. Tens of proxies are negligible. Hundreds are still fine with the idle caps above. **Confidence: HIGH** on the API facts, **MEDIUM** on the byte estimates (measure with `/proc/self/status` in phase testing).

**Why not raw hyper-util:** it offers the same proxy connectors reqwest uses internally (`Tunnel`, SOCKS), but you would re-implement redirects-off, decompression, timeouts, and body plumbing, and still need per-proxy pools. You gain nothing.

### 2. Two listeners, one process

```rust
let token = CancellationToken::new();
let api   = axum::serve(TcpListener::bind(cfg.api_listen).await?, api_router(state.clone()))
              .with_graceful_shutdown(token.clone().cancelled_owned());
let admin = axum::serve(TcpListener::bind(cfg.admin_listen).await?, admin_router(state.clone()))
              .with_graceful_shutdown(token.clone().cancelled_owned());
tokio::try_join!(api.into_future(), admin.into_future())?;
```
- The **AI router** gets `DefaultBodyLimit::max(64 * 1024 * 1024)`. axum's default is **2 MB** (verified in axum-core source), and it would 413 image and long-context requests.
- The **admin router** gets `CsrfLayer`, a Host-header allow-list (DNS-rebinding defense for an unauthenticated UI), and compression.
- Graceful shutdown waits for in-flight streams, so add a hard deadline (e.g. 30 s) after cancel. A 10-minute generation must not block a container stop forever.

### 3. SQLite with sqlx: split writer and reader pools

```rust
let opts = SqliteConnectOptions::from_str(&db_url)?
    .create_if_missing(true)
    .journal_mode(SqliteJournalMode::Wal)
    .synchronous(SqliteSynchronous::Normal)
    .busy_timeout(Duration::from_secs(5))
    .foreign_keys(true)
    .auto_vacuum(SqliteAutoVacuum::Incremental)   // must be set before first table is created
    .pragma("cache_size", "-4000")                // ~4 MB page cache per connection
    .pragma("temp_store", "memory");
let writer = SqlitePoolOptions::new().max_connections(1).connect_with(opts.clone()).await?;
let reader = SqlitePoolOptions::new().max_connections(4).connect_with(opts.read_only(true)).await?;
sqlx::migrate!("../../migrations").run(&writer).await?;
```
- A single writer connection removes `SQLITE_BUSY`/"database is locked" from deferred transactions upgrading to write locks. This is the most-reported sqlx+SQLite failure.
- Request logs go through `mpsc` into **one writer task** that batches (every ~250 ms or 100 rows) in one transaction. The hot path never awaits SQLite.
- The retention job deletes in chunks, then runs `PRAGMA incremental_vacuum(N)`. Never run a blocking full `VACUUM` (OmniRoute #12821).
- sqlx 0.9 note: `query()` now takes `impl SqlSafeStr`, so dynamic SQL must be wrapped in `AssertSqlSafe(...)`. Prefer the macros.

### 4. Assets and CSS in the single binary
`assets/` (htmx.min.js, htmx-ext-sse.js, app.css built by the Tailwind standalone CLI, icons) is embedded with rust-embed. **Commit the generated `app.css`** and have CI fail if a rebuild changes it. Plain `cargo build` / `cargo install` then works without any CSS toolchain, which suits a public single-binary project. Use `tailwindcss --watch` in development.

### 5. Multi-arch static builds and Docker

```dockerfile
# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM ghcr.io/rust-cross/cargo-zigbuild:<pinned> AS build
ARG TARGETARCH
WORKDIR /src
COPY . .
ENV SQLX_OFFLINE=true
RUN case "$TARGETARCH" in amd64) T=x86_64-unknown-linux-musl;; arm64) T=aarch64-unknown-linux-musl;; esac \
 && rustup target add $T \
 && cargo zigbuild --release --locked --target $T -p rsrouter \
 && cp target/$T/release/rsrouter /rsrouter

FROM gcr.io/distroless/static-debian13:nonroot
COPY --from=build /rsrouter /usr/local/bin/rsrouter
USER nonroot
EXPOSE 8080 8081
VOLUME ["/data"]
ENTRYPOINT ["/usr/local/bin/rsrouter", "--db", "/data/rsrouter.db"]
```
- The compile stage runs on the **build platform** and cross-compiles. The final stage only `COPY`s. So `docker buildx build --platform linux/amd64,linux/arm64` never runs rustc under QEMU, which is 5–10× slower and often OOMs.
- distroless `static-debian13` includes CA certificates (needed by rustls-platform-verifier), tzdata, and a `nonroot` user (65532). There is no shell, so image size is roughly the binary.
- Add cargo-chef layers for dependency caching once the build gets slow.
- **Fallback if zig + aws-lc-sys cross-compilation breaks:** build natively on GitHub's `ubuntu-24.04` and `ubuntu-24.04-arm` runners (free for public repos) with `musl-tools`, then assemble the image from the two binaries (`COPY rsrouter-${TARGETARCH}`).
- For the author's homelab (1 GB AMD VMs, Pi 5 with 16K pages, Ampere A1), use the pinned image with `user: "${PUID}:${PGID}"` in compose and a named volume at `/data`.

---

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|---|---|---|
| axum 0.8 | actix-web 4 | Never for this project. It runs its own per-thread runtime model and HTTP stack alongside reqwest's hyper (two HTTP implementations in one binary) and has a weaker tower middleware story. Its benchmark lead does not matter when upstream LLM latency dominates. |
| reqwest 0.13 (client per proxy) | hyper-util legacy client directly | Only if you need something reqwest hides, such as a custom connector with per-connection proxy selection and a custom pool key. Not needed in v1. |
| reqwest 0.13 | wreq / TLS-impersonating clients | Only if a provider starts fingerprinting TLS/JA3. None of the v1 providers requires it (OmniRoute uses impersonation only for web-scraping providers outside rsrouter's scope). Keep `Transport` behind a trait so it can be swapped. |
| aws-lc-rs provider | ring (`reqwest` `rustls-no-provider` + `rustls/ring`) | If aws-lc-sys cross-compilation causes problems or binary size becomes critical. ring is ~8.5 MB of C/asm source vs ~69 MB and gives a smaller binary. ring has seen no crate release since 0.17.14 (2025-03), though the repo is active. Remember to call `install_default()` for ring. |
| sqlx 0.9 | rusqlite 0.40 + rusqlite_migration 2.6 + tokio-rusqlite 0.8 / deadpool-sqlite 0.14 | If you want the absolute leanest dependency tree, direct SQLite APIs (backup, hooks), and do not care about compile-time query checking. It is a solid choice. It loses the `query!` safety net and `FromRow` derives, and you hand-write row mapping. **Do not combine with sqlx**: both link `libsqlite3-sys` (`links = "sqlite3"`), so versions must match exactly or the build fails. |
| sqlx 0.9 | sea-orm 2.0 | If the admin surface becomes large CRUD with complex filters. For about 12 tables, an ORM adds a large compile-time and dependency cost without much benefit. |
| sqlx 0.9 | diesel 2.3 (+ diesel-async) | Rarely. diesel is sync-first, diesel-async's SQLite support is only a sync wrapper, and migrations need `diesel_migrations`. |
| askama 0.16 | minijinja 2.24 (+ minijinja-embed, minijinja-autoreload) | If you want hot-reload of templates in development and are willing to accept runtime template errors. Also viable if end users should be able to theme the UI. |
| askama 0.16 | maud 0.27 | If you prefer HTML-in-Rust macros. Its last release was 2025-02, and a long, designer-quality UI is harder to maintain in a macro DSL than in `.html` files. |
| htmx 2.0.x | htmx 4.0.0 (released 2026-08-28, npm `next` until early 2027) | Reasonable for a greenfield project, since assets are pinned and embedded and the CDN "latest" breakage risk does not apply. Its fetch-based streaming and morph swaps are nice. Not recommended yet: it is one month old, attribute inheritance is now explicit (`:inherited`) and breaks 2.x idioms silently, event names changed, and agents and docs overwhelmingly produce 2.x code. Revisit when 4.x becomes `latest`. |
| htmx | Datastar (crate `datastar` 0.4, SSE-native) | If the UI becomes mostly live and reactive. It conflicts with PROJECT.md's htmx decision, and its ecosystem is small. |
| Tailwind 4 + daisyUI 5 | Hand-written CSS / Pico CSS / Open Props | If you want zero CSS build step. Pico gives a clean look with no build, but "beautiful, dense admin UI" is far easier with daisyUI components. |
| mimalloc v3 | tikv-jemallocator 0.7 | Only with `JEMALLOC_SYS_WITH_LG_PAGE=16` set at build time (to support 4K/16K/64K pages) and tuned decay. jemalloc aborts with "Unsupported system page size" on Pi 5 16K-page kernels (Home Assistant, Quickwit, Windmill, and Typesense have all hit this). |
| Custom ~150-LOC SSE framer | sse-stream 0.3 / eventsource-stream 0.2.3 | Use sse-stream (maintained, 2026-09) in translation paths if you prefer. The recommendation is to own the framer, because passthrough needs byte-exact forwarding plus a *tap* that peeks at `event: error`/usage without re-encoding. The crates allocate owned `String` events. eventsource-stream has had no release since 2022. |
| tokio loops for jobs | tokio-cron-scheduler 0.15 / apalis 0.7 | Only if users need cron expressions or persisted job queues. Quota probes, proxy health checks, and pruning are fixed intervals, and a scheduler crate pulls chrono-tz, croner, and uuid for nothing. |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|---|---|---|
| `reqwest` `.timeout()` on streaming requests | It is a **total** deadline and kills long generations mid-stream | `.read_timeout()` (idle per read) + an explicit first-byte timeout in the attempt loop |
| reqwest default features (`system-proxy`) | Silently routes upstream traffic through `HTTP(S)_PROXY`/system proxy, bypassing rsrouter's explicit proxy assignment | `default-features = false` + `.no_proxy()` on the direct client |
| `oauth2` crate 5.0.0 | Its latest release (2025-01) depends on reqwest 0.12, rand 0.8, sha2 0.10, and thiserror 1, which duplicates half the tree. Provider flows (Claude Code, Codex, Antigravity, Copilot device flow, Cursor) are non-standard anyway. | Hand-rolled PKCE + token exchange on reqwest 0.13, one module per provider |
| `jsonwebtoken` for reading id_token claims | Pulls a crypto backend just to base64-decode a payload you are not verifying | `base64` URL_SAFE_NO_PAD decode of segment 2 + `serde_json` |
| tiktoken-rs / tokenizers | Embeds multi-MB BPE tables in RAM, which conflicts with 1 GB hosts. TPM limits only need estimates. | `bytes / 4` pre-estimate + the upstream's `usage` for actual accounting |
| sonic-rs / simd-json | Negligible gain (LLM latency dominates), more unsafe code and platform variance | serde_json + `RawValue` (parse less, not faster) |
| `#[serde(flatten)]` for "unknown field passthrough" | Buffers into `Content`, is slow, cannot hold `RawValue`, loses ordering | `IndexMap<String, Box<RawValue>>` at top level |
| jemalloc (default build) | Pi 5 16K-page abort, see above | mimalloc |
| musl's default allocator | Severe contention and slowness with many threads | mimalloc `#[global_allocator]` |
| `panic = "abort"` in release | A panic in any request task kills the whole router and every open stream | Keep `unwind`. Use `CatchPanicLayer` on routers. |
| Tracing layer as the request-log store | Couples diagnostics verbosity to the product feature and makes batching and body-storage toggles awkward | Explicit `LogSink` (mpsc → batch writer; broadcast → live UI) |
| Compiling in QEMU (`buildx` without cross) | 5–10× slower, frequent OOM on proc-macros | `--platform=$BUILDPLATFORM` + cargo-zigbuild, or native arm64 runners |
| glibc `+crt-static` / `*-gnu` static | Unsupported by zig. glibc NSS/DNS breaks when static. | `*-unknown-linux-musl` |
| `sqlx` default features | Pulls `any` + all-driver machinery | `default-features = false` + explicit sqlite features |
| `sqlx` `sqlite` umbrella feature | Also enables `load-extension`, `deserialize`, `unlock-notify` | `sqlite-bundled` only |
| tower-http `CompressionLayer` on the AI router | Unneeded CPU for passthrough, and a risk to streaming flush behavior | Compress only admin HTML/CSS/JS (SSE is excluded by the default predicate anyway) |
| Storing client API keys with argon2/bcrypt | Adds about 50 ms and MBs of memory to *every* AI request for a high-entropy secret | SHA-256 digest lookup |
| Leptos/Dioxus/Yew (WASM UI) | Separate build target, larger artifacts, contradicts the server-rendered decision | askama + htmx |

---

## Stack Patterns by Variant

**If a host is a 1 GB VM (denethor, elrond, gollum-class):**
- Keep the tokio default worker count (= vCPUs) and `max_blocking_threads(32)`.
- sqlx reader pool of 2, `cache_size` -2000. Body storage off or zstd-compressed with short retention.
- Expected idle RSS for an axum+reqwest+sqlx binary on mimalloc is ~15–30 MB (MEDIUM, estimate; verify in phase testing).

**If deploying on Raspberry Pi 5 (pippin) or Ampere A1 (aragorn, legolas):**
- Use the `aarch64-unknown-linux-musl` binary or the arm64 image. mimalloc handles 16K pages, so no special build.

**If a provider later requires TLS/HTTP2 fingerprint impersonation:**
- Add a second `Transport` implementation behind the transport trait for that provider only. Do not switch the global client.

**If Cursor support is enabled (`--features cursor`):**
- Needs HTTP/2 **bidirectional** streaming (a request body fed by a channel while reading the response) plus prost. reqwest over h2 can do full-duplex (`Body::wrap_stream` with an `mpsc` receiver). Validate this in a spike first, since OmniRoute falls back to raw `node:http2` here.

**If the user runs behind a corporate TLS-intercepting proxy:**
- rustls-platform-verifier picks up OS-installed CAs. For the Docker image, mount the CA bundle, or add a setting that calls `tls_certs_merge()` with a user-supplied PEM.

---

## Version Compatibility

| Package A | Compatible With | Notes |
|---|---|---|
| axum 0.8.9 | tower 0.5.3, tower-http 0.7.1, hyper 1.11, http 1 | axum only depends on tower-http 0.6 through an optional/private-docs path, so 0.7 works as a layer (it only needs `http`/`tower` traits) |
| askama_web 0.16.0 | askama 0.16.x, axum-core 0.5 (axum 0.8) | Enable feature `axum-0.8`. Keep askama and askama_web on the same minor. |
| reqwest 0.13.5 | hyper-rustls 0.27, tokio-rustls 0.26, rustls 0.23.4+, rustls-platform-verifier 0.6–0.7 | Your direct `rustls` dependency must be 0.23.x so `tls_backend_preconfigured` downcasts to the same `rustls::ClientConfig` type. A mismatch silently becomes `UnknownPreconfigured` and fails at `build()`. |
| sqlx 0.9.0 | MSRV 1.94. Bundled libsqlite3-sys via a version *range* | Do not add rusqlite unless its libsqlite3-sys resolves to the same version (`links = "sqlite3"`) |
| mimalloc 0.1.52 | libmimalloc-sys 0.1.49 → mimalloc **v3** | The `v2` feature opts back into v2 if a regression appears |
| htmx 2.0.11 | htmx-ext-sse 2.2.4 | The SSE extension is separate in 2.x (`hx-ext="sse"`, `sse-connect`, `sse-swap`) |
| Tailwind 4.3.3 standalone | daisyUI 5.7.x via `daisyui.mjs` plugin file | No Node/npm required |
| cargo-zigbuild 0.23.4 | zig (pin; 0.16.0 current) | Pin the zig version in CI/Docker. aws-lc-sys (C) is compiled by zig cc for both musl targets. |

---

## Sources

- crates.io registry API (`/api/v1/crates/<name>` and `/<name>/<version>`, fetched 2026-09-28): versions, dates, feature flags, and dependency ranges for every crate listed. **Primary.**
- Published crate source tarballs: `reqwest-0.13.5` (Proxy API, no per-request proxy, `tls_backend_preconfigured`, ALPN only set on reqwest-built configs, per-build platform verifier), `libmimalloc-sys-0.1.49` (v3 default), `tower-http-0.7.1` (compression predicate excludes SSE), `axum-core-0.5.6` (2 MB `DEFAULT_LIMIT`). **Primary.**
- reqwest CHANGELOG 0.13.0–0.13.5 (github.com/seanmonstar/reqwest): rustls default, aws-lc default, `rustls-tls` → `rustls`, roots features removed, `query`/`form` opt-in.
- sqlx CHANGELOG 0.9.0 (raw.githubusercontent.com/launchbadge/sqlx): MSRV 1.94, `SqlSafeStr`, `sqlx.toml`, libsqlite3-sys range and SQLite feature split, move to transact-rs org.
- tower-http CHANGELOG 0.7.0/0.7.1: `CsrfLayer`, `DeadlineBody`, `on-early-drop`, `ServeDir` backend trait.
- docs.rs askama_web: `axum-0.8` feature, `WebTemplate` derive.
- aws-lc-rs Linux requirements (aws.github.io/aws-lc-rs/requirements/linux.html): CMake and Go only for FIPS.
- [htmx 4.0.0 release announcement](https://four.htmx.org/announcements/2026-08-28-htmx-4.0.0-is-released), [InfoQ coverage](https://www.infoq.com/news/2026/09/htmx-4-released/), [Dennis Morello, htmx 4 breaking changes](https://morello.dev/blog/htmx-4): 4.x stays on npm `next` until early 2027, 2.x supported indefinitely, SSE via the `hx-sse` extension. MEDIUM.
- npm registry: htmx.org 2.0.11 / 4.0.0 (`next`), htmx-ext-sse 2.2.4, @tailwindcss/cli 4.3.3, daisyui 5.7.46.
- jemalloc on Pi 5 16K pages: [EasyTier #1990](https://github.com/EasyTier/EasyTier/issues/1990), [quickwit #4785](https://github.com/quickwit-oss/quickwit/issues/4785), [home-assistant #105768](https://github.com/home-assistant/core/issues/105768), [windmill #4422](https://github.com/windmill-labs/windmill/issues/4422). MEDIUM (many independent reports).
- ring vs aws-lc-rs build footprint: [pingora #965](https://github.com/cloudflare/pingora/issues/965), [corinth-canal PR #194](https://github.com/rmems/corinth-canal/pull/194). MEDIUM.
- [GoogleContainerTools/distroless README](https://github.com/GoogleContainerTools/distroless): debian13 images current.
- static.rust-lang.org channel manifest: stable 1.98.1 (2026-09-01). ziglang.org download index: zig 0.16.0.
- OmniRoute source (github.com/diegosouzapw/OmniRoute @ 3.8.51): Cursor executor uses Connect-RPC protobuf over `node:http2` bidi. SQLite bloat and RSS notes in `src/lib/db/cleanup.ts`. Depends on `socks`, `fetch-socks`, `https-proxy-agent`.

---
*Stack research for: self-hosted Rust AI API router (rsrouter)*
*Researched: 2026-09-28*
