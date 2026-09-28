# Phase 1: Foundation, Admin Shell & Client Keys - Pattern Map

**Mapped:** 2026-09-28
**Files analyzed:** 24 (grouped by crate)
**Analogs found:** 0 / 24 in-repo. This is a greenfield project. `git ls-files` shows only `.planning/`, `.claude/CLAUDE.md`, `AGENTS.md`, `README.md`, `LICENSE`, `.gitignore`, and no source code.

**Rule for the planner:** there are no in-repo analogs. Every reference pattern below points to the spike-verified excerpts in `01-RESEARCH.md` (abbreviated R§, with line numbers). Cite those, not invented files. The files created in this phase become the analogs for Phases 2 and later.

## File Classification

| New File | Role | Data Flow | Reference Pattern (01-RESEARCH.md) |
|---|---|---|---|
| `Cargo.toml` (workspace), `rust-toolchain.toml` | config | n/a | Standard Stack R§94-171, structure R§237-258 |
| `migrations/0001_client_keys.sql` | migration | CRUD schema | R§311-325 (`client_keys` STRICT table only) |
| `rsrouter.example.toml` | config | n/a | R§327-346 |
| `scripts/build-css.sh`, `crates/rsr-admin/assets-src/input.css` | build utility | file-I/O | R§598-610 |
| `crates/rsr-core/src/{ids,dialect,domain,errors}.rs` | model | transform | R§348-363, R§376-386 |
| `crates/rsr-core/src/traits/*.rs` (Storage, LogSink, ProviderAdapter, UpstreamCodec, RoutingStrategy, CooldownTracker, QuotaProber, ProxyTransport) | trait/interface | seam | R§455-463 (`#[async_trait]` is required, E0038) |
| `crates/rsr-store/src/{lib,open,client_keys}.rs` | repository | CRUD | R§286-309 (writer then reader pool) |
| `crates/rsr-store/build.rs` | config | build | R§305 (`rerun-if-changed=../../migrations`) |
| `crates/rsr-engine/src/{state,snapshot,control}.rs` | service/store | event-driven (mutate, rebuild, swap) | R§348-363, Pitfall 3 R§508-511 |
| `crates/rsr-engine/src/usage.rs` | service + background job | batch | R§445-453 |
| `crates/rsr-api/src/{router,auth,models,errors}.rs` | middleware + controller | request-response | R§365-391 |
| `crates/rsr-admin/src/{router,layers,host_guard}.rs` | middleware | request-response | R§393-420 |
| `crates/rsr-admin/src/pages/{keys,settings,placeholder}.rs` | controller | CRUD (htmx partials) | R§422-443 |
| `crates/rsr-admin/src/assets.rs` | controller | file-I/O (embedded) | R§583-596 |
| `crates/rsr-admin/src/snippets.rs` | utility | transform | R§550-581, Pitfall 8 R§532-537 |
| `crates/rsr-admin/templates/{base,keys,settings,placeholder}.html` | component (template) | render | R§422-443 |
| `crates/rsr-admin/assets/app.js` | component (static JS) | event-driven | R§443, Pitfall 6 R§521-523 |
| `crates/rsrouter/src/{main,config,wiring,shutdown}.rs` | composition root / config | startup | R§270-284, R§327-346, R§459 |
| `crates/rsrouter/tests/*.rs` (e2e), per-crate unit tests | test | request-response | Validation Architecture R§672-713, Pitfall 9 R§539-543 |
| `.sqlx/` (committed cache) | generated config | n/a | Pitfall 2 R§497-506 |

## Pattern Assignments

These are pointers only. The code lives in 01-RESEARCH.md, so it is not duplicated here.

- **Startup and shutdown (`main.rs`, `shutdown.rs`):** R§272-281. Bind both listeners first, share one `CancellationToken`, then `try_join!`. After the servers drain, run the final `usage.flush()`, then close both pools (this checkpoints the WAL). Handle `ctrl_c` and SIGTERM. Defaults are API `0.0.0.0:8080` and admin `127.0.0.1:8081`.
- **DB open (`rsr-store/open.rs`):** R§288-303 as written. The reader options must be built from scratch (Pitfall 1). Map `VersionMissing` to the startup error "database is newer than this binary".
- **Config (`config.rs`):** R§329-343. `resolve(cli, file) -> Config` must be a pure function. Test it with `Cli::try_parse_from`, never `set_var`.
- **ControlPlane (`control.rs`):** R§353-361. Hold one `tokio::sync::Mutex<()>` across write, rebuild and store. Keep disabled keys in the snapshot.
- **Key generation and auth (`auth.rs`):** `gen_key` is at R§368-374. The auth steps are at R§377-379 (Anthropic dialect when `anthropic-version` or `x-api-key` is present). Error envelopes are at R§381-384, and the `/v1/models` shapes are at R§389-391.
- **Admin layers (`admin/router.rs`):** layer order is at R§396-403 (the last `.layer()` is outermost, so the Host guard runs first). The Host guard spec is at R§420: it returns 421 and falls back to the URI authority.
- **Pages (`pages/keys.rs`):** askama `blocks = ["rows"]` plus an `hx-request` branch, at R§425-433. The htmx verbs are at R§437-441: return 200 with an empty body for delete, never 204. For errors, use either a 200 error partial or a 422 `htmx-config` entry (R§442).
- **Usage flusher (`usage.rs`):** the SQL is at R§449-451. The page overlays the in-memory runtime map on top of the DB values.
- **Assets (`assets.rs`):** R§586-594, using the route `/assets/{*path}`.

## Shared Patterns

| Concern | Source | Apply to |
|---|---|---|
| Secret hygiene: return the secret only in the POST response with `no-store`, never log or persist it, add `SetSensitiveRequestHeadersLayer` | R§469-470, R§525-530 | `rsr-api/router.rs`, `pages/keys.rs` |
| Strict CSP: no inline script and no `hx-on:*`, JS goes in `assets/app.js` | R§400, R§443 | all templates, `app.js` |
| sqlx macros with a committed `.sqlx` cache, `as "enabled: bool"` for INTEGER booleans | R§497-506 | `rsr-store/*` |
| Trait objects via `Arc<dyn …>` in `Components`; concrete types chosen only in `wiring.rs` | R§455-463 | all crates |
| Tests: `TempDir` DBs (not `:memory:`), `tower` `util` feature for `oneshot`, kill processes by PID | R§539-543 | all tests |
| `CatchPanicLayer` on routers; keep `panic = "unwind"` | R§403, CLAUDE.md | both routers |

## No Analog Found

All 24 files have no in-repo analog because the repository has no source code yet. The planner should use the 01-RESEARCH.md patterns cited above, all of which were compiled and run in the Phase 1 spike unless marked [ASSUMED] there.

## Metadata

**Analog search scope:** entire repository (`git ls-files`)
**Files scanned:** 5 tracked non-planning files (no source)
**Pattern extraction date:** 2026-09-28
