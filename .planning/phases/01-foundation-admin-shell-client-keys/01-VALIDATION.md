---
phase: "1"
slug: "foundation-admin-shell-client-keys"
# status lifecycle: draft (seeded by plan-phase) → validated (set by validate-phase §6)
# audit-milestone §5.5 distinguishes NOT-VALIDATED (draft) from PARTIAL (validated + nyquist_compliant: false) (#2117)
status: draft
nyquist_compliant: false
wave_0_complete: false
created: "2026-09-28"
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Rust built-in `cargo test` + `#[tokio::test]`; router tests via `tower::ServiceExt::oneshot`; real temp DBs via `tempfile` |
| **Config file** | none — Wave 0 creates the Cargo workspace |
| **Quick run command** | `cargo test -p <touched crate>` |
| **Full suite command** | `SQLX_OFFLINE=true cargo test --workspace && cargo clippy --workspace --all-targets -- -D warnings && cargo fmt --check` |
| **Estimated runtime** | ~30 seconds (after first build) |

---

## Sampling Rate

- **After every task commit:** Run `cargo test -p <touched crate>`
- **After every plan wave:** Run the full suite command plus `cargo sqlx prepare --workspace --check`
- **Before `/gsd-verify-work`:** Full suite must be green + manual UI walkthrough
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

Filled by the planner/executor per task. Requirement → test mapping from RESEARCH.md:

| Requirement | Behavior | Test Type | Automated Command | File Exists | Status |
|-------------|----------|-----------|-------------------|-------------|--------|
| FOUND-01 | Two listeners on separate addrs; routes isolated per port | integration | `cargo test -p rsrouter --test listeners` | ❌ W0 | ⬜ pending |
| FOUND-02 | Fresh DB migrated; data survives reopen; newer DB → VersionMissing; read-only reader pool works | integration | `cargo test -p rsr-store --test migrations --test pools` | ❌ W0 | ⬜ pending |
| FOUND-03 | Config precedence CLI > env > TOML > default; unknown TOML key rejected | unit | `cargo test -p rsrouter config::` | ❌ W0 | ⬜ pending |
| FOUND-04 | Cross-site POST 403; bad Origin 403; bad Host 421; X-Frame-Options DENY | integration | `cargo test -p rsr-admin --test hardening` | ❌ W0 | ⬜ pending |
| FOUND-05 | New key accepted without restart; disabled rejected immediately; no per-request DB read; concurrent creates not lost | integration | `cargo test -p rsr-engine --test hot_config` | ❌ W0 | ⬜ pending |
| FOUND-07 | All 8 traits swappable with stubs via `Arc<dyn …>` | unit + integration | `cargo test -p rsr-engine --test component_swap` | ❌ W0 | ⬜ pending |
| KEYS-01 | Secret format `rsr-…`, only SHA-256 stored, shown once, `Cache-Control: no-store` | integration | `cargo test -p rsr-admin --test keys` | ❌ W0 | ⬜ pending |
| KEYS-02 | Rename / disable / enable / delete | integration | `cargo test -p rsr-admin --test keys` | ❌ W0 | ⬜ pending |
| KEYS-03 | first_used / last_used tracked in memory, flushed to DB | integration | `cargo test -p rsr-engine --test usage` | ❌ W0 | ⬜ pending |
| KEYS-04 | Bearer + x-api-key auth; OpenAI/Anthropic-shaped 401s | integration | `cargo test -p rsr-api --test auth` | ❌ W0 | ⬜ pending |
| UI-01 | Shell with nav for all areas; every nav route 200; HTML escaping | integration | `cargo test -p rsr-admin --test shell` | ❌ W0 | ⬜ pending |
| UI-05 | opencode (`baseURL` ends `/v1`) + Claude Code (`ANTHROPIC_BASE_URL` no `/v1`) snippets with secret | integration | `cargo test -p rsr-admin --test keys snippets` | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] Cargo workspace (`Cargo.toml`, `rust-toolchain.toml`, crate skeletons, `[workspace.dependencies]`, `[profile.release]` with `panic = "unwind"`)
- [ ] `cargo install sqlx-cli --no-default-features --features sqlite` + dev DB + first `cargo sqlx prepare --workspace` + committed `.sqlx/`
- [ ] `crates/rsr-store/build.rs` (`cargo:rerun-if-changed=../../migrations`)
- [ ] Shared test helpers: `test_state()`, `oneshot_json()`, CSRF/Host header builders
- [ ] `scripts/build-css.sh` (pinned Tailwind + sha256) + vendored `htmx.min.js` + `daisyui.mjs` + committed `app.css`
- [ ] `.gitignore`: `tools/`, `data/`, `target/`, `*.db*` (not `.sqlx/`)

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Visual polish / responsiveness of admin shell | UI-01 | Visual judgment | Open admin port at desktop and mobile widths; check nav, layout, theme |
| Real-client smoke with snippet values | UI-05 | Needs real opencode / Claude Code | `curl` with snippet values → `/v1/models` 200; optionally run opencode / Claude Code against it |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
