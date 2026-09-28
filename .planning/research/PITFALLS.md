# Pitfalls Research

**Domain:** Self-hosted AI API router/gateway (OpenAI + Anthropic surfaces, multi-provider, multi-key, combos/fallback, subscription OAuth providers) written in Rust
**Researched:** 2026-09-28
**Confidence:** MEDIUM-HIGH overall. Most failure modes below come from primary sources: issues filed against OmniRoute, LiteLLM and CLIProxyAPI, often with root-cause analysis down to the line. Anthropic API rules are HIGH (official docs, cached 2026-09-25). reqwest/sqlx facts are HIGH (read from crate source and changelog). Subscription-provider ToS and ban behavior is MEDIUM: the policies changed several times in 2026 and are still moving.

> **How to read this file.** The roadmap has no phase numbers yet, so pitfalls map to the *phase themes* listed below. The roadmapper should turn these into numbered phases.
>
> | Theme tag | Scope |
> |---|---|
> | **[Foundation]** | Workspace, config, SQLite and migrations, secrets at rest, the two listeners (AI port and UI port), HTTP client factory |
> | **[Passthrough]** | Same-format streaming proxy (OpenAI to OpenAI, Anthropic to Anthropic), header policy, error dialects, body limits, SSE plumbing |
> | **[Routing]** | Combos, sequential fallback, stream commit point, error classification, cooldowns, 429/Retry-After parsing, cache affinity |
> | **[Translation]** | OpenAI and Anthropic request, response and stream translation: tools, thinking, usage, stop reasons, cache_control |
> | **[Proxies]** | Proxy pool, SOCKS5/HTTP, health checks, per-key assignment |
> | **[OAuth]** | Claude Code, Codex, Antigravity, Copilot and Cursor credential lifecycle and request-shape fidelity |
> | **[Observability]** | Request/route logs, body capture, retention, live log SSE |
> | **[UI]** | Admin UI (Axum + templates + htmx) |
> | **[Release]** | Single binary, musl/glibc, multi-arch Docker |

---

## Critical Pitfalls

### Pitfall 1: Falling back after bytes have already reached the client (the stream "commit point")

**What goes wrong:**
A streaming request goes to model A. The router forwards `200 OK` plus `message_start` (Anthropic) or a first `data:` chunk with `role: assistant` (OpenAI), and then A fails: a 429 arriving as an SSE `error` event, `overloaded_error`, a mid-stream disconnect, or a stall. By then the router can no longer fall back transparently. The client has a half-built message, and splicing model B's stream after it gives duplicated `message_start` frames, content-block indexes that restart at 0, mixed models, and broken usage. LiteLLM #38610 shows the subtler version: the primary attempt held back lifecycle frames, but the *fallback* attempt did not. When both failed, the client got `200` + `message_start` + an in-band error instead of a real 429/5xx. LiteLLM #33273 and #39462 show mid-stream fallback dropping the partial usage consumed before the failure.

**Why it happens:**
The naive design pipes `upstream.bytes_stream()` straight into the response body, so status and headers go out as soon as the upstream answers. Developers also treat an upstream `200` as success. Several providers (Anthropic, Codex, Antigravity) send `200` and then fail inside the stream.

**How to avoid:**
- Define an explicit **commit point** per attempt. Buffer upstream lifecycle events (`message_start`, `ping`, empty `content_block_start`, OpenAI role-only first chunk) until the first *meaningful* delta arrives: non-empty text, a thinking delta, a tool-call start, or a clean terminal event. Only then send status and headers to the client and flush the buffer. Every attempt, including fallbacks, goes through the same hold-back code path.
- Before commit, any failure (HTTP error, in-stream `error` event, stream EOF, first-token timeout) is a normal fallback trigger.
- After commit, never splice another model in. Send an in-dialect terminal error (Anthropic `event: error`; OpenAI `data: {"error":{...}}` followed by closing without `[DONE]`), then close. Record it as "committed failure" in the route log.
- Make the first-meaningful-token timeout a separate knob from the idle-read timeout (see Pitfall 11).
- Record usage from *every* attempt, including partial ones, in the log. Upstreams bill aborted attempts.

**Warning signs:**
Clients report "stream ended unexpectedly", or Claude Code shows `API Error` after a long spinner. Logs show a 200 status with no `message_stop`/`[DONE]`, two `message_start` frames on one client response, or fallback success rates that look too good.

**Phase to address:** [Routing] (design the commit point before any fallback code). [Passthrough] must already expose the upstream as a parsed event stream, not raw bytes.

---

### Pitfall 2: The router silently destroys upstream prompt caching

**What goes wrong:**
Cache hit rate through the router is far below what the same client gets talking to the provider directly. Real cases:
- OmniRoute #13305: Antigravity got **0%** cache hits because OmniRoute generated a random `sessionId` per request. CLIProxyAPI got 92% on the same account.
- OmniRoute #9436: hoisting mid-conversation `role: system` blocks into top-level `system` dragged `cache_control` with them. That left no breakpoint in `messages` and caused about 11% full-history cache rebuilds.
- OmniRoute #10220: passthrough lost the 1h TTL behavior, so the Claude subscription 5-hour window was "exhausted in minutes".
- OmniRoute issue #1712 (cited in its code): identity/billing blocks were re-prepended on every retry and stacked up, which broke prefix matching.
- LiteLLM #43324 and #41424 dropped message-level `cache_control` or dropped it in a bridge. LiteLLM #38679 injected breakpoints past Anthropic's 4-breakpoint cap.
- Anthropic caches are **isolated per workspace/organization**, so rotating the same conversation across upstream keys from different accounts means a cold write every time.

**Why it happens:**
Caching is an exact byte-prefix match. The render order is `tools` → `system` → `messages`, and any change invalidates everything after it. Tool-definition changes and model switches invalidate *all* tiers. Every "harmless" normalization is a cache buster: reordering JSON keys, re-serializing tool schemas, injecting a timestamp or request ID, moving system text, adding tools. So is round-robin across keys. Rust adds a trap of its own: `serde_json::Map` is a **BTreeMap by default**, which sorts object keys alphabetically on re-serialization unless the `preserve_order` feature is enabled.

**How to avoid:**
- **Passthrough is byte-faithful.** For same-format routes, parse only what routing needs (`model`, `stream`, a few flags). Rewrite the `model` field by splicing, or re-serialize with `serde_json` `preserve_order` (IndexMap). Never deserialize into typed structs and serialize back: unknown fields disappear.
- Any mutation a provider requires (Claude OAuth identity, Codex `instructions`, Antigravity envelope) must be **deterministic** and a function of stable inputs (account + conversation), never per request. Apply it to the **pristine client body on every attempt**, never to the previous attempt's mutated body, so retries stay idempotent.
- Derive provider session identifiers (Antigravity `sessionId`, Codex `session_id`/`prompt_cache_key`, Claude `X-Claude-Code-Session-Id`/`metadata.user_id`) from the client's own session header when present. Otherwise use the cache-affinity fingerprint. Never use `Uuid::new_v4()` per request.
- Preserve `cache_control` exactly where the client put it, including `ttl`, on `tool_result` blocks (not moved inside `tool_result.content`, LiteLLM #41954), and on messages whose content is a list.
- Do not hoist, merge or reorder system content on Anthropic-to-Anthropic routes. Newer Claude models accept mid-conversation `role: "system"` natively.
- If auto-injecting breakpoints for OpenAI-format clients on Anthropic upstreams (a nice differentiator), count existing markers *including tools* and stay at 4 or fewer. Place them deterministically: last system block plus last user turn.
- Keep conversations sticky to one upstream account (see Pitfall 8).
- Build a **cache regression test**: send the same 3-turn conversation twice through the router and assert `cache_read_input_tokens > 0` (Anthropic) or `prompt_tokens_details.cached_tokens > 0` (OpenAI) on turn 2+. Also golden-diff the outbound body across consecutive turns: the overlap must be byte-identical apart from `cache_control` placement.

**Warning signs:**
`cache_read_input_tokens` is zero or low on turn 2+. `cache_creation_input_tokens` stays close to the full prompt size on every turn. Subscription quotas drain much faster than with native clients. A diff of consecutive outbound bodies shows changes inside the shared prefix.

**Phase to address:** [Passthrough] (byte-faithful forwarding, `preserve_order`), [Routing] (stickiness), [OAuth] (deterministic identity injection), [Translation] (cache_control mapping). Add the cache regression test to the [Passthrough] exit criteria.

---

### Pitfall 3: Corrupting thinking blocks, signatures and encrypted reasoning

**What goes wrong:**
Multi-turn agent sessions die several turns in with 400s:
- `Invalid signature in thinking block` (OmniRoute #7899).
- `thinking or redacted_thinking blocks in the latest assistant message cannot be modified` (CLIProxyAPI #3624).
- Codex `/responses`: `The encrypted content ... could not be verified` (CLIProxyAPI #3189).
- Gemini/Antigravity: "Corrupted thought signature" (CLIProxyAPI #1538).

Specific mechanisms seen in the wild:
- **Fabricating** a placeholder signature for unsigned thinking blocks. In OmniRoute #14671 this cascaded every combo into `503 Maximum combo retry limit reached`.
- **Stripping** signature-only blocks. Current Claude models return `{"thinking":"", "signature":"..."}` by default, and LiteLLM #41361 dropped these as "empty", silently removing the reasoning.
- Replaying thinking or encrypted reasoning produced by one account/model against another after a fallback or key rotation.

As of 2026 the Anthropic API also enforces **preserved thinking**: a thinking block's signature binds the prefix that produced it (system, tool set, all earlier messages). New accounts created on or after 2026-08-31 get a **400** if *any* earlier turn was edited: hoisted system text, stripped old tool results, per-request injected text, or a rebuilt `system`/`tools`. So router-side history edits are now hard failures, not just cache misses.

**Why it happens:**
Developers treat thinking as display text rather than opaque state. OpenAI Chat format has no field for a signature, so translated routes lose it, and someone "fixes" that by inventing one. Fallback and key rotation ignore the fact that reasoning state is bound to identity.

**How to avoid:**
- **Never fabricate, edit, reorder or drop signed blocks** on Anthropic-to-Anthropic routes. Treat `thinking`, `redacted_thinking`, Gemini `thoughtSignature` and Codex `encrypted_content` as opaque bytes.
- One targeted recovery only. If an Anthropic upstream returns the exact invalid-signature / "bound to a different conversation" 400, strip **all** `thinking`/`redacted_thinking` blocks from history (keep `text` and `tool_use`) and retry **once** against the same target. Do not count this as a key failure or trigger cooldown (OmniRoute #7899 spec, and Anthropic's documented recovery). For Codex, strip `encrypted_content` reasoning items when the target account differs from the producer.
- When a combo moves to a **different model family or format**, strip foreign reasoning artifacts proactively. Anthropic models silently drop other models' blocks, but non-Anthropic upstreams will choke on them.
- For OpenAI-format clients hitting Anthropic upstreams with thinking plus tools, keep a server-side, bounded, TTL'd map `tool_call_id → (thinking blocks + signature)` so the next request can re-attach them. If a signature is missing, drop the thinking and never invent one. Surface reasoning to OpenAI clients as `reasoning_content` (the de-facto field) and never as `<think>` text in `content` (OmniRoute #5123 leaked `</think>` into content, which clients then fed back as history).
- Normalize the `thinking` config **only when a combo substitutes a different model** than the client requested, using a per-model capability table. Current Claude models reject `thinking:{type:"disabled"}` and `budget_tokens` (400), and Claude Code sends `disabled` on background calls (OmniRoute #3554).

**Warning signs:**
400s whose message contains `signature`, `thinking`, `encrypted content` or `bound to a different conversation`. Errors that appear only after N turns, or only after a fallback or key switch. Literal `<think>`/`</think>` text in assistant content.

**Phase to address:** [Translation] (signature cache, reasoning mapping) and [Routing] (strip-on-family-change, once-only recovery that does not count toward cooldown). Flag [Translation] for deeper research: model capability tables change with every Claude/GPT/Gemini release.

---

### Pitfall 4: Tool-call translation bugs that permanently poison a conversation

**What goes wrong:**
Tool calls break in ways that are sticky: once a malformed assistant turn is in the client's history, *every* later request fails.
- OmniRoute #14798: a tool called with no arguments was emitted to the OpenAI client as `arguments: ""`. On replay that became Anthropic `tool_use.input: ""`, and every later turn got a 400.
- OmniRoute #3415: forcing `anthropic-beta` flags the client never asked for (interleaved thinking) produced `stop_reason: tool_use` with unparseable input. Claude Code aborted with "tool call could not be parsed".
- Cursor reported malformed args even though the proxy had logged valid ones (OmniRoute #12725).

**Why it happens:**
The formats differ in shape:
- **OpenAI streams tool calls as fragments** keyed by `index`. `id` and `name` come on the first fragment only, and `arguments` is a JSON *string* split at arbitrary points. Some OpenAI-compatible providers omit `index`, repeat `id` on every chunk, send the whole call in one chunk, or send interleaved parallel calls.
- **Anthropic streams** each call as `content_block_start{type:tool_use,id,name,input:{}}` → `input_json_delta{partial_json}`* → `content_block_stop`. Content blocks must be closed in order, and text and tool blocks get distinct indexes.
- **Tool results:** OpenAI uses one `role:"tool"` message per call. Anthropic needs all `tool_result` blocks in **one** user message immediately after the assistant turn, with `is_error` available.
- **`tool_choice`:** OpenAI `"required"` ↔ Anthropic `{"type":"any"}`; `"none"` ↔ `{"type":"none"}`; `parallel_tool_calls:false` ↔ `disable_parallel_tool_use:true`. The newest Claude models (Opus 5.5, Sonnet 5.5, Fable 5.1) **reject forced `tool_choice` any/tool with a 400**.
- **Name and schema constraints** differ. Gemini/Antigravity reject many JSON-Schema keywords and have stricter name rules.
- Anthropic rejects empty text blocks. OpenAI assistant tool-call messages often have `content: null`.

**How to avoid:**
- Write the stream translators as explicit **state machines** (open block index, per-tool-call accumulators keyed by `index` with `id` as fallback, pending-close queue), not ad-hoc chunk mappers.
- Normalize empty or `null` arguments to `"{}"` in both directions. Parse `arguments` strictly. If JSON is invalid, pass the string through inside a wrapper rather than sending `input: "<string>"`.
- Group consecutive `tool` messages into one Anthropic user message. Merge adjacent same-role messages. Drop empty text blocks.
- Never add `anthropic-beta` flags the client didn't send on passthrough. Forward the client's header as-is. Only provider adapters that *require* a flag (Claude OAuth `oauth-2025-04-20`) may add exactly that flag.
- Put `tool_choice` handling behind the model capability table. If forced choice is unsupported, return a clear 400 in the client's dialect, or (configurable) downgrade to `auto`. Never silently fail over to another model and burn quota.
- Build a **round-trip property test**: generate client request → translate → fake upstream response/stream → translate back → client appends to history → translate again. The second outbound request must be valid for the upstream schema. Cover 0 args, parallel calls, unicode, and huge args.

**Warning signs:**
400s mentioning `tool_use.input`, `tool_result`, `messages.N.content`, or "all messages must have non-empty content" (claude-code-router #4). The client says "tool call could not be parsed". Failures are sticky for the rest of a session.

**Phase to address:** [Translation] (state machines plus round-trip property tests); [Passthrough] (beta header policy).

---

### Pitfall 5: Wrong usage accounting and stop-reason mapping

**What goes wrong:**
Clients and dashboards show 0% cache hits, wrong token counts, or truncated answers that look complete.
- OmniRoute #10535: OpenAI-format responses never carried `prompt_tokens_details.cached_tokens`.
- OmniRoute #11817: the OpenAI→Anthropic stream translator dropped the trailing `choices: []` chunk, which is exactly where Fireworks and other providers put usage.
- LiteLLM #43012: `stop_reason: model_context_window_exceeded` mapped to `stop`, so truncation was invisible.
- LiteLLM #40857: content-filtered turns reported as `end_turn`.

**Why it happens:**
- **Token semantics differ.** Anthropic `input_tokens` **excludes** `cache_read_input_tokens` and `cache_creation_input_tokens`. OpenAI `prompt_tokens` **includes** `cached_tokens`.
- **Usage arrives in different places.** Anthropic streams put input and cache usage in `message_start` and cumulative `output_tokens` in `message_delta`. OpenAI streams send usage only when `stream_options.include_usage: true`, as a final chunk with `choices: []`, and many translators index `choices[0]` and skip it.
- **Stop-reason sets differ.**

**How to avoid:**
- Mapping (write unit tests for every row):
  - Anthropic→OpenAI: `prompt_tokens = input + cache_read + cache_creation`, `cached_tokens = cache_read`.
  - OpenAI→Anthropic: `input_tokens = prompt_tokens − cached_tokens`, `cache_read_input_tokens = cached_tokens`.
  - Stop reasons: `end_turn`/`stop_sequence` ↔ `stop`; `max_tokens` and `model_context_window_exceeded` → `length`; `tool_use` ↔ `tool_calls`; `refusal` ↔ `content_filter`; `pause_turn` → pass through or `stop` with a log flag. Log unknown values and pass them through verbatim; never default them to `stop`.
- If rsrouter injects `stream_options.include_usage` for its own accounting, **strip the extra usage-only chunk** from the client stream when the client didn't ask for it. Some clients index `choices[0]` and crash.
- Handle usage-only chunks (`choices: []`) in every stream consumer.
- Store raw upstream usage JSON in the log alongside normalized fields so mapping bugs can be fixed later.

**Warning signs:**
Cache hit ratio in opencode or other clients stays at 0 while Anthropic's console shows cache reads. Token counts in logs are estimates rather than upstream numbers. Users say answers "just stop".

**Phase to address:** [Translation] (mapping tables and tests); [Observability] (store raw usage).

---

### Pitfall 6: Error misclassification causes false bans, cooldown storms and quota-burning fallback loops

**What goes wrong:**
- OmniRoute #12859: a single Anthropic OAuth `403 "Request not allowed"` (a per-request refusal) was persisted as a terminal **ban**, and the provider was dead until someone re-logged.
- OmniRoute #13157: a Cloudflare managed challenge on one Codex sub-endpoint permanently banned a healthy Codex account.
- OmniRoute #14258: a Grok content refusal (403) banned the connection.
- OmniRoute #14261: refresh-less `claude setup-token` credentials were flapped to "expired" every sweep while serving 377 requests without error.

The opposite failure: client-caused 400s (bad schema, context too long for *every* model) trigger fallback through a whole combo. That resends a 100k-token prompt N×M times and burns every account's quota for nothing. A thinking-signature 400 cascaded into a 503 across whole combos (OmniRoute #14671).

**Why it happens:**
Status codes are overloaded. 403 can mean a banned account, a WAF/TLS-fingerprint block (Cloudflare 1010), a region block, or a content refusal. 429 can mean a short rate limit, a quota exhausted for hours, or (on Antigravity) content-triggered throttling (CLIProxyAPI #4696). Anthropic's 529 `overloaded_error` says nothing about the key.

**How to avoid:**
- Build one **error classifier** module with provider-specific overrides. It returns `{scope: request|key+model|key|provider, action: fallback|retry_same|fail_to_client|cooldown(d)|needs_reauth, reason}`.
- Classify on status **plus body plus headers**. HTML bodies or `cf-ray`/challenge markers mean transport/WAF, not the account. `invalid_grant` on refresh means `needs_reauth`. A network error during refresh is retryable. 529 or `overloaded` means fallback to another provider with no key cooldown.
- **Never make a terminal state from a single observation.** Use a sliding window (for example 3 matching failures in 10 minutes), and have a background half-open probe that auto-recovers. Leave "banned" as a label only a human sets, or only after repeated `401 authentication_error` plus refresh `invalid_grant`.
- Client-caused 400s (schema or validation errors not specific to the model) go straight back to the client with **no fallback and no cooldown**. Model-specific 400s (context length for this model, unsupported parameter) may fall back to the next *different* model, not to other keys of the same model.
- Enforce a **per-request attempt budget and wall-clock budget** across combo × keys × retries (for example at most 6 attempts and at most 120 s before first byte). The route log shows each skip and why.

**Warning signs:**
Accounts marked dead that work when tested manually. Combos returning "max retries" for requests that fail identically on every model. Quota use spikes during an upstream incident.

**Phase to address:** [Routing] (classifier plus budgets, the core of this phase); [OAuth] adds per-provider overrides. Flag for deeper research during [OAuth]: each subscription provider's error vocabulary.

---

### Pitfall 7: Mis-parsed Retry-After / reset times and cooldown thundering herds

**What goes wrong:**
- OmniRoute #14479: z.ai reset stamps have no timezone and are printed in **Asia/Shanghai**. They were parsed as UTC, so a healthy provider was cooled down for about 8h55m instead of 55 minutes.
- CLIProxyAPI #5404: a stale Codex `resets_at` blocked failover after the account was already usable again.
- When a cooldown expires, every queued and new request hits the key at the same instant, gets 429 again, and the cooldown escalates. Under combos, a whole pool can flap in lockstep.
- OmniRoute #14990: every cooldown write invalidated the whole `/v1/models` catalog cache, and the rebuild stalled the event loop for about 50 s under 429-heavy traffic.

**Why it happens:**
Reset information arrives in at least six formats:
- `Retry-After` in seconds *or* as an HTTP-date
- `anthropic-ratelimit-*-reset` (RFC 3339)
- OpenAI `x-ratelimit-reset-requests` as Go-style durations (`1s`, `6m0s`)
- Google `RetryInfo.retryDelay` (`"30s"`) inside the JSON error body
- free-text body timestamps in provider-local time
- provider quota endpoints

Cooldown is also often applied at the wrong scope, for example a per-model limit cooling down the whole key.

**How to avoid:**
- Implement a `ResetHint` parser per provider with fixture tests built from real error bodies. Clamp results to a sane range (for example 1 s to 24 h) and log the raw hint. If there is no hint, use exponential backoff with **full jitter**.
- Scope cooldowns correctly: (key, model) by default, and (key) only when the provider's limit is account-wide (Claude subscription 5h/7d windows, Codex plan windows).
- On expiry, move to **half-open**: let exactly one request (or one background probe) through, and reopen fully on success. Add ±10–20% jitter to expiry times.
- Keep cooldown state **in memory** (DashMap/ArcSwap) and persist it asynchronously. Never tie hot-path state writes to config or catalog cache invalidation.
- Prefer proactive limits (user RPM/TPM/concurrency) enforced with a semaphore or token bucket *before* sending, so bursts get queued or rerouted instead of producing 429s.

**Warning signs:**
Cooldowns lasting hours for providers known to reset hourly. Periodic 429 bursts at exact intervals. UI or `/models` latency spikes that correlate with 429 storms.

**Phase to address:** [Routing]. Per-provider reset parsers are added in [OAuth] and in the non-OAuth provider work.

---

### Pitfall 8: Cache affinity that silently overrides the routing contract, or keys on the wrong fingerprint

**What goes wrong:**
OmniRoute #8370: a three-model "priority" combo expanded to 13 account targets and kept starting at model 2 long after model 1 had recovered. Prompt-cache affinity, session stickiness, "last known good provider" and account rotation all reordered targets without a single owner, and disabling per-combo stickiness didn't help because a *global* affinity flag also applied. The same issue explicitly describes the product problem: users could not predict or observe the effective plan. A separate failure: fingerprints built from system prompt plus first message collide across unrelated conversations (every Claude Code session starts with the same system prompt and often the same first message), which funnels load onto one key.

**Why it happens:**
Affinity is added as a layer after the strategy instead of as an input to it. Fingerprints ignore explicit session identifiers the clients already send.

**How to avoid:**
- One routing pipeline with one owner: `resolve combo → candidate list (ordered by strategy) → affinity *reorders within constraints* → filter cooldowns → attempt`. Write the rules down:
  - Affinity may pin **the key within the chosen model**.
  - Pinning to a lower-priority *model* is allowed only for the life of the cache TTL (5 min or 1 h), is shown in the route log ("affinity: stayed on model 2, cache-warm"), and can be turned off per combo.
- Build the affinity key in this order of preference:
  1. an explicit client session id: `x-claude-code-session-id`, `metadata.user_id` session component, Codex `prompt_cache_key`/`session_id`, opencode session header if present
  2. hash of (client key + model/combo + tools hash + system + first user message + first assistant message), once available
- Expire affinity entries on the cache TTL horizon, and immediately when the pinned key enters cooldown.
- Bound the affinity map (LRU, for example 10k entries) so it can't grow without limit.

**Warning signs:**
"My combo ignores the order." One key getting a disproportionate share of traffic. Users unable to explain the route taken.

**Phase to address:** [Routing]. The effective plan must appear in the route log from day 1 ([Observability]).

---

### Pitfall 9: OAuth token refresh races, single-use refresh tokens and wedged refresh locks

**What goes wrong:**
- OmniRoute #14970: a hung refresh HTTP call held the per-connection refresh mutex **forever**. For 14+ hours every request on that connection joined the in-flight refresh and got 504 until the process restarted.
- OmniRoute #13183: one failed Claude refresh (~8h TTL) left the connection "sticky-dead". Using the *same* account in two refreshers (OmniRoute plus CLIProxyAPI) invalidated each other's tokens.
- Codex refresh tokens are **single-use**: `refresh_token_reused` reports in openai/codex-plugin-cc#789 and manaflow-ai/subrouter#129. Two concurrent refreshes, or a crash between receiving and persisting the new token, lose the account.
- CLIProxyAPI #3654: waves of Codex `token_invalidated` 401s that needed re-login.

**Why it happens:**
Refresh gets implemented lazily inside the request path, with no single-flight, no timeout, and persistence after use. Users also import credential files from other tools (`~/.codex/auth.json`, `~/.claude/.credentials.json`), which creates two consumers of one rotating token family.

**How to avoid:**
- **Single-flight per credential**, for example a `tokio::sync::Mutex<()>` or a `Shared` future stored in a per-credential slot. The refresh call gets a hard timeout (for example 20 s) and the lock is released on every path. Use a guard object and never hold the lock across an unbounded await.
- **Persist the new refresh token (and access token) transactionally before returning it to callers.** If persisting fails, keep it in memory, mark the credential "unsaved", and alert in the UI.
- Refresh **proactively** in a background task at about 80% of lifetime with jitter, so request-path refresh becomes rare.
- Classify refresh failures: `invalid_grant`/`refresh_token_reused` means needs re-auth (show clearly in the UI); network or 5xx means retry with backoff while the old access token is still used until its real expiry.
- Model credential *types* explicitly: `{access+refresh}`, `{long-lived no refresh}` (setup-token), `{two-stage}` (Copilot: GitHub OAuth token → short-lived Copilot token, about 30 min). Absence of a refresh token is not expiry.
- Refresh must use the **same proxy** as the inference traffic for that credential.
- UI copy: "Log in fresh here. Do not import credentials from a CLI you keep using. Rotating refresh tokens will log one of them out."

**Warning signs:**
Credential flapping between valid and expired. Several refresh calls in the same second for one credential. 504s clustered on one credential. `invalid_grant` right after a restart.

**Phase to address:** [OAuth] (core of the phase). [Foundation] provides the transactional secret store.

---

### Pitfall 10: Subscription providers need faithful client impersonation, and carry ToS and ban risk the user must be told about

**What goes wrong:**
Subscription endpoints only work, or only bill correctly, when requests look like the official client:
- **Claude OAuth** needs `anthropic-beta: oauth-2025-04-20`, a Claude-Code system prefix ("You are Claude Code, Anthropic's official CLI for Claude."), stainless/user-agent headers pinned to a real CLI version, and session IDs that match between header and `metadata.user_id`. OmniRoute's `claudeIdentity.ts` is about 500 lines of this. Without the prefix, CLIProxyAPI #2599 users were billed as "extra usage" instead of plan usage.
- **Codex** needs `chatgpt-account-id`, `originator`, and `session_id`, plus non-empty `instructions` and the right `store` handling.
- **Antigravity** needs a project id and envelope. It has returned 429 for all accounts despite quota (CLIProxyAPI #1015) and 429 based on the *system prompt identity text* (CLIProxyAPI #4696).
- **Copilot** needs the two-stage token plus editor headers. `x-initiator: agent|user` affects premium-request billing, so forwarding it matters.
- **Cursor** is protobuf/Connect over HTTP/2 using an imported token.

The ban risk is real:
- Google/Antigravity accounts were banned for ToS violation while using a proxy (CLIProxyAPI #1637 and #1631, reported as detected across different instances and IPs).
- GitHub's abuse detection flags scripted or multi-account Copilot use.
- Anthropic's consumer terms (clarified Feb 2026) state that Free/Pro/Max OAuth tokens may not be used in third-party products, and Anthropic has since changed billing treatment several times (third-party usage drawn from "extra usage", a credit plan announced and then paused).

**Why it happens:**
These APIs are private, undocumented and change without notice. Every client-version bump can break the shape. Multi-account rotation, a core rsrouter feature, is the exact pattern the providers' abuse detection is built to catch.

**How to avoid:**
- Isolate each subscription provider behind a `ProviderAdapter` trait with a **versioned "identity profile"** (headers, UA, version strings, required system prefix) kept in data or config so it can be bumped without refactoring. Port OmniRoute's captured shapes, and record the capture date and source client version.
- Keep required mutations minimal, deterministic and idempotent (Pitfall 2). Never rewrite beta headers beyond the required ones.
- **ToS/ban warning is a product requirement, not a footnote.** When a user adds a Claude/Codex/Antigravity/Copilot/Cursor OAuth credential, show a per-provider risk notice ("using this subscription outside the official client may violate the provider's terms; accounts have been suspended; do not rotate multiple accounts to evade limits"). Require an explicit acknowledgement, stored with a timestamp. Keep it in README as well.
- Default to **one account per provider** without rotation for OAuth providers, and put multi-account rotation behind an "I understand" toggle.
- Pin each OAuth credential to a stable egress (fixed proxy or direct). Never let it hop across IPs or regions (see Pitfall 15).
- Build an **adapter health canary** (a cheap periodic request per provider) so breakage from client-version drift shows up before users hit it.

**Warning signs:**
"Extra usage" billing messages. 429 with quota available. 403 with a ToS text body. Sudden fleet-wide failures right after a provider client release.

**Phase to address:** [OAuth]. Flag it as the highest-risk, research-heavy phase, and schedule it *after* the core is stable so churn there doesn't block the rest. The risk-acknowledgement UI belongs in [UI] but is specified in [OAuth].

---

### Pitfall 11: SSE broken by buffering, timeouts and disconnect handling in the path

**What goes wrong:**
Streams arrive in one burst at the end, die at exact intervals, or keep running after the client leaves.
- CLIProxyAPI #5545: Codex streams for deep-reasoning models died after about 30/60/90 s of **upstream silence**, a 20% drop rate on some models.
- Reverse proxies buffer `text/event-stream` (nginx `proxy_buffering on` by default) or compress it.
- Cloudflare-proxied hostnames (the author's `*.gabiflix.uk`, `*.carap.icu` and similar) return **524 if no response bytes arrive within about 100–120 s**. That clashes directly with "hold the client socket open until the first model replies" across a long combo.
- OmniRoute #10017: forwarding raw upstream SSE control lines (`id:`, `event:`, `retry:`, comments) broke a strict OpenAI-SDK-based client.
- When the client disconnects, a detached upstream task keeps generating and billing.

**Why it happens:**
- In reqwest, `ClientBuilder::timeout()` is a **total** timeout that includes the streaming body, so a sensible-looking 60 s cuts off long generations. `read_timeout()` (a per-read idle timeout) is the right tool.
- Middleware stacks (compression, timeout layers) get applied globally, including to the AI routes.

**How to avoid:**
- For upstream clients, use `connect_timeout` (about 10 s), a per-provider **idle** `read_timeout` (generous, for example 300 s, since reasoning models go quiet), HTTP/2 keepalive pings, and TCP keepalive. Never use a total `timeout` on streaming requests. Implement the first-byte timeout yourself in the router.
- Never put a `TimeoutLayer` on AI routes. For `CompressionLayer`, make sure the predicate excludes `text/event-stream` (tower-http's default predicate does; custom ones often don't), or just don't compress the AI port.
- Set `Cache-Control: no-cache`, `X-Accel-Buffering: no` and `Connection: keep-alive` on SSE responses. Document the nginx, Traefik/Pangolin and Cloudflare caveats in deployment docs, including "Cloudflare-proxied hostnames cap time-to-first-byte at about 100 s; keep the AI API on a non-Cloudflare-proxied (grey-cloud) hostname or expect 524s on long fallback chains".
- **Re-emit normalized SSE**: parse upstream events and write clean `event:`/`data:` frames in the client dialect. Don't pipe raw bytes, even on passthrough. Parse properly: events split across TCP chunks, `\r\n` line endings, multi-line `data:`, and UTF-8 code points split across chunks.
- Tie the upstream request's lifetime to the client response body. If the body stream is dropped (client disconnected), the upstream future is dropped too. Use a drop guard to finalize the log row ("client_cancelled", partial usage). If work must move to a spawned task, pass a `CancellationToken`.
- Decide and document the policy for long pre-commit waits. Option A: keep holding headers, with a first-byte budget below 100 s. Option B: after a threshold, commit a 200 and send SSE comment keepalives (`: ping`); from then on failures can only be reported in-band. Test both against Claude Code, opencode (ai-sdk) and openai-python.
- Graceful shutdown: stop accepting, then drain in-flight streams with a deadline (for example 30 s), then abort. Axum's graceful shutdown otherwise waits indefinitely for long streams.

**Warning signs:**
Tokens appearing all at once. Drops at round-number durations. 524s in Cloudflare logs. Upstream usage continuing after client cancels.

**Phase to address:** [Passthrough] (this is the core of that phase). Deployment docs go in [Release].

---

### Pitfall 12: SQLite write contention, WAL misconfiguration and migration traps

**What goes wrong:**
`database is locked` (SQLITE_BUSY) errors under concurrent streaming load, usually because every request writes a log row, updates `last_used` on the client key, and records cooldown state.
- sqlx has no reader/writer split (transact-rs/sqlx #459). A task running many transactions starves the others into SQLITE_BUSY (#3920). Deferred `BEGIN` transactions that later write fail *immediately* with BUSY on WAL snapshot conflicts, even with `busy_timeout` set (the reason for the IMMEDIATE requests in #1182/#481).
- The WAL file grows without bound when a long-lived reader (for example a live-log SSE handler holding a read transaction) prevents checkpoints.
- OmniRoute #13325: a DB grew to 3.7 GiB from tables without retention, and `DELETE` doesn't shrink the file.
- Migrations: editing an already-applied sqlx migration fails at startup with "previously applied but has been modified" (sqlx #3794). SQLite `ALTER TABLE` can't change column types or constraints, so a 12-step table rebuild is needed. `PRAGMA foreign_keys` is off by default per connection. An older binary started against a newer schema corrupts assumptions.

**Why it happens:**
SQLite allows one writer at a time. Treating it like Postgres (a pool of N writers plus ORM-style transactions) causes lock storms, and the hot path gets used for writes that could be batched.

**How to avoid:**
- Set connection pragmas on *every* connection: `journal_mode=WAL`, `synchronous=NORMAL`, `busy_timeout=5000`, `foreign_keys=ON`, and `auto_vacuum=INCREMENTAL` (set before the first table is created). Keep a modest `cache_size`.
- **Single writer.** Use a dedicated writer (a pool with `max_connections(1)`, or better, a writer task fed by a bounded mpsc channel) with `BEGIN IMMEDIATE` for multi-statement writes. Run a separate read pool.
- Keep **hot routing state in memory** (cooldowns, affinity, counters, `last_used`) and flush it in batches every few seconds. The request path never awaits a DB write.
- Write log rows in **batched transactions** (for example 100 rows or 250 ms). If the log channel is full, drop or sample logs and increment a "dropped logs" counter. Never block or slow a request because of logging.
- Retention pruning deletes in small batches (`DELETE ... WHERE id IN (SELECT ... LIMIT 1000)`) followed by `PRAGMA incremental_vacuum`. Run `wal_checkpoint(TRUNCATE)` periodically. Never hold a read transaction open across an SSE stream to the UI; re-query per batch.
- Migrations: embed with `sqlx::migrate!()`, run at startup, and back up the DB file before applying (`VACUUM INTO 'backup-<version>.db'`). **Never edit an applied migration.** Store `schema_version`/app version and **refuse to start** if the DB is newer than the binary. Write table rebuilds as explicit 12-step migrations with `foreign_keys=OFF`. Add a CI test that migrates a DB fixture from every released version.
- Document: don't put the DB on NFS/SMB. WAL needs shared memory on a local filesystem. ZFS datasets and Docker named volumes are fine.

**Warning signs:**
`database is locked` in logs. p99 latency correlating with log volume. A `-wal` file larger than the main DB. DB size growing without bound.

**Phase to address:** [Foundation] (pragmas, writer task, migration policy). [Observability] (batched log writer, retention).

---

### Pitfall 13: Memory blowups from buffered bodies, JSON amplification and chat logging

**What goes wrong:**
- OmniRoute #11804: RSS jumped from 1.5 GB to 16 GB under combo routing.
- OmniRoute #13621: completed-request previews kept multi-MB request strings alive, reaching 10 GiB RSS after about 3 h.
- OmniRoute #13123: log export buffered the whole table and caused an OOM.
- OmniRoute #12577 and #11222: unbounded in-memory buffering in executors.
- OmniRoute #14815: a stack overflow in request translation on a single 3.5 MB base64 image.

rsrouter targets **1 GB VMs and a Raspberry Pi**, so the same patterns would kill it immediately. A Claude Code request with a 1M-token context or images can be 5–30 MB. Parsing into `serde_json::Value` multiplies that several times, and keeping per-attempt copies for fallback plus a logging copy plus a response accumulator multiplies it again.

**Why it happens:**
Fallback needs the request body replayable, so developers clone it per attempt. Logging keeps full bodies in memory until the stream ends. Channels and caches have no bounds.

**How to avoid:**
- Hold the client body once as `bytes::Bytes` (clones are refcounted and zero-copy). Parse once. Build each attempt's outbound body from the original; don't keep N mutated copies alive. Drop them when the attempt ends.
- Parse only what's needed for passthrough (for example `serde_json::value::RawValue` or partial structs with `#[serde(flatten)]` of `RawValue`s) so large `messages` arrays aren't expanded into `Value` trees. Translation will need full trees. Put those behind a **semaphore for "heavy" requests** (size above a threshold) so at most K big bodies are in memory at once.
- Set explicit body limits: raise axum's `DefaultBodyLimit` (default **2 MB**, which will reject real Claude Code traffic with 413) to a configurable value (for example 64 MB) *on AI routes only*, and fail cleanly above it.
- Log capture: stream bodies to storage with **caps** (for example keep the first and last 256 KB and record the truncation), compress with zstd, write off the hot path. Keep the response accumulator only while logging is enabled, with a size cap.
- Bound every channel, cache and map (affinity LRU, signature cache, `tool_call_id` map, live-log broadcast with lagging receivers dropped).
- Use `mimalloc` (or jemalloc) as the global allocator, especially on musl (Pitfall 16). Test with `heaptrack`/`dhat` under a load test of 20 concurrent 5 MB streaming requests on a 1 GB cgroup.
- Recursive JSON walkers must be iterative or depth-limited, because deeply nested client JSON otherwise overflows the stack.

**Warning signs:**
RSS grows with request size and does not return to baseline. OOM kills in `docker events`. Latency cliffs under concurrent large requests.

**Phase to address:** [Passthrough] (body handling, limits, allocator); [Translation] (heavy-request semaphore); [Observability] (capped, streamed body capture). Add a memory load test to the [Passthrough] exit criteria.

---

### Pitfall 14: Secrets at rest, header leakage, and an unauthenticated admin UI reachable from the browser

**What goes wrong:**
- Upstream API keys and OAuth refresh tokens sit in plaintext in SQLite, in backups, and in logs of captured request headers. OmniRoute #12572 wrote session tokens with default file permissions. OmniRoute #14484 echoed secrets in plaintext in GET responses.
- Client `Authorization`/`x-api-key` headers get forwarded **upstream**, leaking the rsrouter key to third-party providers.
- The admin UI has no auth by design. A UI on a LAN IP is reachable by **any web page the admin visits**: CSRF via form POST, or DNS rebinding that turns the victim's browser into a proxy to `http://192.168.1.3:<ui-port>`. That allows reading keys or adding a provider pointed at an attacker's endpoint so all traffic is exfiltrated.
- Captured chat bodies are sensitive, and the client keys are shared with friends, so their conversations get logged.

**How to avoid:**
- Encrypt secrets at rest: XChaCha20-Poly1305 or AES-GCM with a master key from `RSROUTER_MASTER_KEY` or a key file (0600). Generate one on first run and warn loudly if it lives next to the DB. Decrypt into `secrecy::SecretString`, and make sure `Debug`/`Display` never print secrets.
- Store client API keys **hashed** (SHA-256 is fine because they are 256-bit random). Show them once. Compare in constant time. Keep a short prefix for identification.
- Forward headers by **allowlist** per provider (for example `anthropic-version`, `anthropic-beta`, `x-initiator`, `x-claude-code-session-id`, `user-agent` only where appropriate). Always drop `authorization`, `x-api-key`, `cookie`, hop-by-hop headers, `host`, `content-length` and `accept-encoding` (otherwise compressed upstream bytes break SSE parsing). Redact auth headers and known secret fields in logs.
- UI hardening without adding auth:
  - Bind the UI to `127.0.0.1` by default outside Docker. In compose, publish it to `127.0.0.1:` or an internal network only.
  - Check `Host` against an allowlist to stop DNS rebinding.
  - Require a matching `Origin`/`Sec-Fetch-Site: same-origin` on every state-changing request to stop CSRF. htmx sends `HX-Request`, and requiring it adds another cheap gate.
  - Never render full secrets back. Use masked values plus an explicit reveal action.
- Offer an optional "log bodies for these client keys only" setting, and show on the client-key page whether bodies are being logged.
- Model discovery and custom provider base URLs are admin-only SSRF surfaces. That's acceptable, but the **AI API must never accept upstream URLs from clients**.

**Warning signs:**
`grep` of the DB file or logs finds `sk-`/`sk-ant-`/`ghu_` strings. Upstream providers see an unfamiliar bearer token. UI endpoints accept a POST without an `Origin` header.

**Phase to address:** [Foundation] (encryption, hashing, header allowlist primitives); [UI] (Host/Origin checks, masking); [Observability] (redaction, per-key body logging).

---

### Pitfall 15: Proxy pool DNS leaks, fail-open to direct, and inherited environment proxies

**What goes wrong:**
- In reqwest, `socks5://` resolves the target hostname **locally** (a DNS leak that exposes which providers you call and may resolve to the wrong region). Only `socks5h://` resolves through the proxy (see reqwest's own Tor example).
- When an assigned proxy dies, naive code falls back to a direct connection. The subscription account then appears from a second IP, which is exactly the multi-location signal that gets accounts flagged.
- reqwest honors `HTTP_PROXY`/`HTTPS_PROXY`/`ALL_PROXY`/`NO_PROXY` from the environment by default, so "direct" clients in Docker may secretly go through a corporate or sidecar proxy.
- Proxy health checks that only open a TCP connection to the proxy pass while the proxy can't reach the provider. OmniRoute #9694 and #7080 had false negatives on IPv4-only SOCKS5. OmniRoute #14157 probed the wrong port for HTTP proxies on port 80.
- OmniRoute #12529: Cloudflare-fronted upstreams reject some TLS fingerprints (error 1010). rustls's fingerprint is distinctive.

**Why it happens:**
The proxy is configured on the reqwest `Client`, not per request, so developers either build a client per request (slow, Pitfall 16) or reuse one client and lose per-key proxy semantics.

**How to avoid:**
- Normalize user-entered SOCKS5 proxies to `socks5h://` (show "remote DNS: on") and allow opting out explicitly.
- Keep a **client cache keyed by egress** (`Direct`, `Proxy(id, version)`): one `reqwest::Client` per egress, rebuilt only when that proxy's config changes. Direct clients are built with `.no_proxy()` unless the user opts into env proxies.
- **Fail closed**: if a key is assigned to a proxy and that proxy is down, the key is unavailable (skip it, log "egress down"). Never go direct. Token refresh, quota probes and model discovery for that key use the same egress.
- Health-check *through* the proxy to a real target (the provider's base URL or a neutral endpoint), with IPv4/IPv6 awareness and a latency measurement. Retire proxies after N consecutive failures and re-probe with backoff.
- Treat TLS impersonation (the `wreq` crate, BoringSSL-based and Chrome-like) as an optional future transport behind the transport trait. Classify Cloudflare 1010 responses as "fingerprint rejection" rather than a key failure (Pitfall 6).

**Warning signs:**
DNS queries for provider domains visible on the LAN resolver while traffic should be going through a proxy. Account security emails about a "new location". Proxy marked healthy while every request through it times out.

**Phase to address:** [Proxies]; the egress-keyed client cache belongs in [Foundation].

---

### Pitfall 16: Rust-specific traps (HTTP client lifecycle, blocking the runtime, musl/TLS builds, serde fidelity)

**What goes wrong and how to avoid it:**
- **`reqwest::Client` per request.** This throws away the connection pool, TLS session resumption and HTTP/2 multiplexing, and adds 50–300 ms and CPU per request (painful on a Pi). Build clients once per egress (Pitfall 15) and clone the handle (it's an `Arc`). Set `pool_idle_timeout` below the upstream's keepalive (for example 60 s) and cap `pool_max_idle_per_host`.
- **Blocking in async.** zstd compression of big bodies, large JSON (de)serialization, argon2-style hashing, synchronous `rusqlite` calls, and local tokenizers (tiktoken-rs) all stall Tokio workers and freeze every stream. Use `spawn_blocking` for CPU-heavy work above a size threshold. Don't count tokens locally for TPM limits: estimate as bytes/4 and reconcile with upstream usage. Never hold a `std::sync::Mutex` across `.await`. For hot shared state, prefer `ArcSwap` config snapshots and `DashMap`.
- **musl + TLS + allocator.**
  - reqwest **0.13 defaults to rustls with the aws-lc provider**. `aws-lc-sys` needs a C toolchain (and cmake/clang in some setups) to cross-compile to `aarch64-unknown-linux-musl`. Either use `cargo-zigbuild`/`cross` in CI with that toolchain, or pick the `ring` provider (`rustls-no-provider` + ring).
  - Avoid `native-tls`/OpenSSL entirely on musl (the classic `openssl-sys` cross-compile pain).
  - reqwest 0.13 uses `rustls-platform-verifier` (the system trust store). A `scratch`/distroless image **without CA certificates fails every TLS handshake**. Either ship `ca-certificates` in the image or configure bundled `webpki-roots` explicitly.
  - musl's malloc is slow under multi-threaded contention. Set `#[global_allocator]` to mimalloc.
- **serde fidelity.**
  - Enable `serde_json`'s `preserve_order` (Pitfall 2).
  - Never round-trip client JSON through typed structs without `#[serde(flatten)] extra: Map<String, Value>` capture, because unknown fields (new API features) get dropped silently.
  - Beware `f64` re-formatting and integer precision if you re-serialize numbers. `RawValue` avoids both.
- **Model id parsing.** `provider-prefix/model` must split on the **first** `/` only. Upstream ids contain slashes (`meta/llama-...` on NVIDIA, `anthropic/claude-...` on OpenRouter).
- **Axum defaults.** `DefaultBodyLimit` is 2 MB (Pitfall 13). Extractor rejections return plain-text errors, so wrap them to return errors in the client's dialect (Anthropic `{"type":"error","error":{...}}` versus OpenAI `{"error":{...}}`). Claude Code decides whether to retry based on status and error type.

**Warning signs:**
High p50 latency with low CPU (client per request). All streams stalling together (blocked runtime). CI arm64 build failing in `aws-lc-sys`. `UnknownIssuer`/cert errors only inside Docker.

**Phase to address:** [Foundation] (client factory, allocator, serde settings, error envelopes); [Release] (cross-compile matrix, CA certs in image).

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Pipe raw upstream bytes to the client on passthrough | Zero parsing cost, trivially "transparent" | No commit point, no usage capture, control lines leak, no in-dialect errors | Never for chat; OK for binary endpoints (audio/image download) |
| Typed request structs for everything | Nice Rust ergonomics | Unknown fields dropped, cache-busting re-serialization, breaks with every new API feature | Only for fields rsrouter *reads*; carry the rest as `RawValue`/flattened map |
| Terminal "banned" state on first 401/403 | Simple state machine | False outages lasting until a human notices (OmniRoute #12859/#13157) | Never; require repeated evidence plus a failed refresh |
| One global `reqwest::Client` ignoring proxies | Simple | Keys assigned to proxies go direct: IP leak and account flags | Only before [Proxies] exists, and never for OAuth providers |
| Synchronous DB write per request (logs, last_used) | Easy consistency | SQLITE_BUSY storms, latency tied to disk | MVP with fewer than 5 concurrent users; replace in [Observability] |
| Porting OmniRoute's TypeScript adapters line by line | Fast to "working" | Inherits their bug classes (fabricated signatures, forced betas, random session ids) | Port the *captured request shapes and constants*, not the control flow |
| Hard-coding provider identity strings (CLI versions, UA) in code | Quick | Every upstream client release needs a rebuild and release | Put them in data/config with a capture date from day 1 |
| Automatic `thinking`/`tool_choice` normalization everywhere | Fewer 400s | Violates transparency, changes model behavior silently | Only when a combo substituted a different model and the capability table says unsupported |
| Storing full request/response bodies inline in the log table | Easy UI queries | DB bloat, slow pruning, memory spikes on export | Store capped, compressed bodies in a separate table or blob store keyed by log id |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Anthropic Messages (API key) | Forwarding the client's `x-api-key`/`Authorization`; overriding `anthropic-beta` | Replace auth with the upstream key; forward client `anthropic-beta`/`anthropic-version` verbatim |
| Anthropic `/v1/messages/count_tokens` | Not implemented, so Claude Code errors or pre-flight checks slow down | Route to upstream count_tokens where available; otherwise return a clearly-labelled estimate |
| Anthropic `/v1/models` format | Serving the OpenAI list shape to Anthropic clients | Serve `{data:[{type:"model",id,display_name,created_at}],has_more,first_id,last_id}` on the Anthropic surface |
| Claude Code auth to rsrouter | Accepting only `Authorization: Bearer` | Accept both `x-api-key` (ANTHROPIC_API_KEY) and Bearer (ANTHROPIC_AUTH_TOKEN) on both surfaces |
| OpenAI Chat streaming | Forgetting `stream_options.include_usage`, or crashing on `choices: []` | Inject when needed for accounting, strip for clients that didn't ask, handle empty choices |
| OpenAI-compatible providers (NVIDIA, DeepSeek, OpenRouter, Z.ai, Cloudflare) | Assuming spec-exact tool-call deltas and error bodies | Tolerant accumulator; per-provider error and reset parsers (z.ai local-time stamps, OpenRouter per-key 429s) |
| Reasoning fields | Only handling `reasoning_content` | Handle `reasoning_content` (DeepSeek-style), `reasoning`/`reasoning_details` (OpenRouter), inline `<think>` tags; per-provider rule on whether reasoning must be echoed back |
| Codex (ChatGPT subscription) | Treating it as Chat Completions; sending empty `instructions`; round-robin across accounts | Responses API adapter; deterministic `instructions`; `session_id`/`prompt_cache_key` stickiness; strip `encrypted_content` on account change |
| Antigravity | Random `sessionId` per request; unsanitized JSON Schema for tools | Stable session id from affinity key; Gemini-compatible schema sanitizer; preserve `thoughtSignature` |
| Copilot | Treating the GitHub OAuth token as the API token; dropping `x-initiator` | Two-stage token with its own refresh; forward `x-initiator` so agent turns aren't billed as premium requests |
| Cursor | Treating it as an HTTP/JSON API | Protobuf/Connect over HTTP/2 with an imported token; isolate completely, lowest priority |
| OAuth loopback redirects (Codex `localhost:1455`, others) | Expecting the redirect to reach rsrouter | The paste-callback-URL flow is correct: tell the user the browser page will "fail to load" and to copy the full URL from the address bar |
| Device-code flows (Copilot, Codex device flow) | Forcing them into the callback-URL paste model | Support a `DeviceCode` flow type (show user_code + verification URL, poll in the background) alongside `PkcePaste` and `ImportToken` |
| Cloudflare-proxied hostnames in front of rsrouter | Long pre-commit waits with no bytes | Keep time-to-first-byte under about 100 s or serve the AI API on a non-proxied hostname |

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Client built per request | High latency, CPU spikes on TLS | Egress-keyed client cache | Immediately on a Pi; above ~5 rps elsewhere |
| Full `Value` parse of every body | RSS spikes with request size | `RawValue` passthrough; heavy-request semaphore | Around 10 concurrent 5 MB Claude Code requests on a 1 GB VM |
| Per-request DB writes on the hot path | `database is locked`, p99 tied to disk | In-memory state plus batched writer | Around 10–20 concurrent streams |
| Catalog or config cache invalidated by hot-path state | `/models` latency spikes during 429 storms | Separate config cache from runtime state | First 429 storm (OmniRoute #14990) |
| Unbounded live-log broadcast to UI | Memory growth with the UI open | `tokio::sync::broadcast` with bounded capacity; lagged receivers skip | Hours of UI open at moderate traffic |
| Multiplicative retry fan-out (combo × keys × retries) | Quota evaporates during incidents; slow failures | Attempt and wall-clock budgets per request | Combos with more than 3 models × more than 3 keys |
| Local token counting for TPM limits | CPU-bound runtime stalls | Byte-based estimate plus upstream reconciliation | Large contexts on ARM |
| Log table without retention | Multi-GB DB, slow UI queries | Retention by days and size, incremental vacuum | Weeks of body logging (OmniRoute #13325) |

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Plaintext upstream keys and refresh tokens in SQLite and backups | Credential theft from a stolen volume or backup | Envelope encryption with an external master key; `secrecy` types |
| Forwarding client auth headers upstream | rsrouter key leaked to every provider | Header allowlist; strip auth, cookies, hop-by-hop |
| Unauthenticated UI with no Host/Origin checks | CSRF and DNS rebinding from any website lead to key exfiltration or traffic redirection | Localhost bind by default, Host allowlist, Origin/Sec-Fetch-Site checks, masked secrets |
| Logging Authorization headers and full bodies by default | Secrets and private chats in logs | Redact by default; body logging opt-in per client key, with retention |
| Client keys stored reversibly or compared non-constant-time | Key disclosure or timing leaks | SHA-256 hash, constant-time compare, show once |
| AI API accepting arbitrary upstream URLs or provider config from clients | SSRF into the homelab LAN | Only admins (UI port) define upstreams |
| OAuth credentials moving across egress IPs | Provider account suspension | Pin the credential to an egress; fail closed |
| Shared client keys with body logging silently on | Privacy breach for friends and family | Per-key logging visibility; README note |

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Opaque routing ("which model answered?") | Users can't trust combos (OmniRoute #8370) | Route log per request: each candidate, skip reason (cooldown until T, affinity, egress down), attempt outcome, commit point |
| Credential state flapping or unexplained "expired" | Operators re-login needlessly | Show last success, last error with raw body, next refresh time, and why it isn't renewed (OmniRoute #13450) |
| Silent ToS risk | Users lose paid accounts | Mandatory per-provider risk acknowledgement at connect time |
| Undiscoverable fixes (TLS fingerprint, remote DNS, Cloudflare timeouts) | Users give up (OmniRoute #12529) | Classifier emits a *remediation hint* shown next to the error |
| Cooldowns without a visible reset time or manual override | "Why is my key unused?" | Show countdown, source of the reset hint (header or body), and a "clear cooldown" button |
| Model lists flooded by every discovered model | Clients like opencode become unusable | Enable/disable per key (already planned); default newly discovered models to *disabled* or show them in a "new" section |

## "Looks Done But Isn't" Checklist

- [ ] **Streaming passthrough:** often missing the commit point. Verify that a fake upstream returning `200` then an in-stream 429 before content results in a clean fallback with a single `message_start` to the client.
- [ ] **Fallback:** often missing hold-back on the *fallback* attempt. Verify that a double failure before content gives the client a real error status (LiteLLM #38610).
- [ ] **Prompt caching:** often missing byte-fidelity. Verify turn 2 of a real Claude Code session shows `cache_read_input_tokens > 0` through rsrouter, and an OpenAI-format client sees `cached_tokens`.
- [ ] **Tool calls:** often missing zero-argument and parallel-call cases. Verify the round-trip property test passes for empty args, 3 parallel calls, and interleaved text.
- [ ] **Thinking:** often missing signature-only blocks. Verify `{"thinking":"","signature":"..."}` survives passthrough unchanged, and nothing fabricates signatures.
- [ ] **Usage:** often missing the `choices: []` final chunk. Verify usage from Fireworks-style trailing chunks reaches Anthropic-format clients.
- [ ] **Stop reasons:** often missing `model_context_window_exceeded`, `refusal` and `pause_turn`. Verify the mapping tests.
- [ ] **Client disconnect:** often missing upstream cancellation. Verify that killing curl mid-stream stops upstream bytes within 1 s and the log row reads "client_cancelled" with partial usage.
- [ ] **Body limits:** often missing the axum 2 MB default. Verify a 20 MB request with a base64 image succeeds.
- [ ] **Token refresh:** often missing single-flight plus timeout. Verify 50 concurrent requests on an expired token cause exactly 1 refresh call, and a hung refresh endpoint doesn't wedge the credential.
- [ ] **Proxies:** often missing fail-closed and `socks5h`. Verify a key assigned to a dead proxy never goes direct, and DNS for the provider isn't resolved locally.
- [ ] **Migrations:** often missing the downgrade guard and backup. Verify an older binary refuses a newer DB and a pre-migration backup file exists.
- [ ] **Docker image:** often missing CA certificates. Verify an HTTPS upstream call works from the published arm64 and amd64 images.
- [ ] **Anthropic surface:** often missing `count_tokens`, `x-api-key` auth, and Anthropic-format `/v1/models` and error envelopes. Verify with the real Claude Code CLI.
- [ ] **UI:** often missing Origin/Host checks. Verify a cross-origin form POST to the UI port is rejected.

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Raw-byte passthrough shipped without a commit point | HIGH | Re-architect the stream layer around parsed events; everything downstream (fallback, usage, logging) is affected. Avoid by doing it first |
| Cache-busting mutations discovered late | MEDIUM | Add outbound-body golden diffs per turn, find the first divergent byte, make the mutation deterministic |
| Terminal ban states persisted wrongly | LOW | Migration that resets `banned` to `needs_check`; half-open probe reactivates |
| Lost rotating refresh token | MEDIUM (per account) | User must re-login; add persist-before-use and single-flight to prevent recurrence |
| SQLite lock storms | MEDIUM | Introduce the writer task and in-memory hot state; no schema change needed if designed behind a repository trait |
| DB bloat from body logs | LOW-MEDIUM | Batched purge, `incremental_vacuum`, or `VACUUM INTO` offline; move bodies to a separate table |
| Provider identity drift (new CLI version breaks OAuth) | LOW if identity profiles are data | Bump the profile in config or DB; canary confirms |
| Account banned by provider | HIGH, often unrecoverable | Nothing technical; the ToS acknowledgement and conservative defaults are the only mitigation |

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| 1. Fallback after partial stream / commit point | [Routing] (stream API from [Passthrough]) | Fake-upstream tests: pre-content 429, mid-stream error, double failure, stall |
| 2. Prompt-cache destruction | [Passthrough], [Routing], [OAuth], [Translation] | Cache regression test (turn 2+ cache reads > 0); outbound-body golden diff |
| 3. Thinking/signature/encrypted reasoning corruption | [Translation], [Routing] | Signature-only block passthrough test; once-only strip-and-retry test; cross-family strip test |
| 4. Tool-call translation poisoning | [Translation] | Round-trip property tests (0 args, parallel, huge args, unicode) |
| 5. Usage and stop-reason mapping | [Translation] | Table-driven unit tests; trailing `choices: []` stream fixture |
| 6. Error misclassification and fan-out | [Routing] (+ [OAuth] overrides) | Classifier fixture suite from real error bodies; attempt/wall-clock budget tests |
| 7. Retry-After parsing and thundering herd | [Routing] | Reset-hint parser fixtures (incl. z.ai local time); half-open probe test |
| 8. Affinity overriding routing | [Routing], [Observability] | Combo-order test after recovery; effective plan shown in route log |
| 9. OAuth refresh races | [OAuth] ([Foundation] store) | 50-concurrent-expired-token test gives 1 refresh; hung refresh endpoint test |
| 10. Impersonation fidelity and ToS risk | [OAuth], [UI] | Canary per provider; risk acknowledgement stored; identity profile in data |
| 11. SSE buffering, timeouts, disconnects | [Passthrough], [Release] docs | Long-silence stream test (5 min idle); disconnect cancellation test; nginx/Traefik smoke |
| 12. SQLite contention and migrations | [Foundation], [Observability] | Load test with logging on and 0 BUSY errors; migration-from-every-version CI |
| 13. Memory blowups | [Passthrough], [Translation], [Observability] | 1 GB cgroup load test with large bodies; RSS returns to baseline |
| 14. Secrets and UI exposure | [Foundation], [UI], [Observability] | grep DB/logs for key patterns; cross-origin POST rejected; Host allowlist test |
| 15. Proxy DNS leak and fail-open | [Proxies] ([Foundation] client cache) | Dead-proxy fail-closed test; `socks5h` normalization; env-proxy isolation test |
| 16. Rust build and runtime traps | [Foundation], [Release] | Client reuse metric; `tokio-console` blocked-task check; arm64 musl CI; TLS works in the image |

**Ordering implications for the roadmap:**
1. [Foundation] then [Passthrough] must come first and must include the parsed-event stream layer, byte-faithful forwarding, body limits and the cache regression test. Almost every later pitfall depends on these being right.
2. [Routing] (commit point, classifier, cooldowns, affinity) comes before [Translation]. Fallback semantics are easier to verify with same-format traffic.
3. [Translation] needs its own phase with property tests. It is the largest bug surface in every comparable project.
4. [OAuth] comes late and gets a research flag. It is volatile, carries ToS risk, and each provider is effectively its own mini-project (Cursor lowest priority).
5. [Observability] can be built incrementally, but the route log's *data model* (attempts, skips, reasons) should exist from [Routing] onward.

## Sources

Primary sources: issue trackers, most with root-cause analysis. Confidence: MEDIUM, as individual reports.

- OmniRoute (diegosouzapw/OmniRoute) issues: #3415 (forced anthropic-beta corrupts tool_use), #3554 (thinking disabled forwarded to incompatible combo model), #5123 (`</think>` leaked to content), #7899 (thinking-signature recovery spec), #8370 (affinity reorders priority combos), #9436 (system hoisting moves cache_control), #10017 (SSE control lines), #10220 (cache TTL on passthrough), #10535 (cached_tokens dropped), #11804 (combo OOM), #11817 (usage on `choices:[]` chunk dropped), #12529 (Cloudflare 1010 TLS fingerprint), #12577/#11222/#13123 (unbounded buffering), #12859/#13157/#14258 (false permanent bans), #13183 (sticky-dead OAuth, dual refreshers), #13305 (random Antigravity sessionId, 0% cache), #13325 (SQLite growth), #13621 (retained request strings, 10 GiB RSS), #14261 (setup-token flagged expired), #14479 (z.ai reset timezone), #14671 (fabricated signature cascades), #14798 (empty tool args poison history), #14815 (stack overflow on large image), #14970 (wedged refresh mutex), #14990 (cooldown writes bust catalog cache).
- OmniRoute source (open-sse/executors/claudeIdentity.ts, codex.ts, github.ts; src/lib/oauth/*): Claude OAuth identity shape and beta-flag gating, Codex headers and instructions, Copilot two-stage token and `x-initiator`, loopback-redirect and device-flow handling.
- LiteLLM (BerriAI/litellm) issues: #33273/#39462 (mid-stream fallback usage), #38610 (fallback leaks message_start), #38679 (cache breakpoint cap), #40857 (content filter as end_turn), #41361/#42550 (signature-only thinking dropped), #41424/#43324 (cache_control dropped), #41954 (tool_result cache_control moved), #43012 (context-window stop reason mapped to stop).
- CLIProxyAPI (router-for-me/CLIProxyAPI) issues: #1015 and #4696 (Antigravity 429 despite quota / identity-phrase throttling), #1538 (thought signature), #1631/#1637 (Google account bans), #2599 (OAuth metered as extra usage without Claude Code identity), #3189 (encrypted reasoning bound to account), #3624 (thinking blocks modified), #3654 (Codex token invalidation), #5404 (stale resets_at), #5545 (streams die on upstream silence).
- claude-code-router (musistudio/claude-code-router) #4 (non-empty content requirement).
- Codex single-use refresh tokens: openai/codex-plugin-cc#789, manaflow-ai/subrouter#129.

Official and authoritative. Confidence: HIGH.

- Anthropic API docs (claude-api reference, cached 2026-09-25): prompt caching prefix rules, invalidation hierarchy, 4-breakpoint cap, 5m/1h TTL, per-workspace cache isolation, model-dependent minimums; preserved thinking (history-edit check enforced for accounts created on or after 2026-08-31, strip-and-retry recovery); forced `tool_choice` and `thinking:disabled` 400s on newest models; stop reasons incl. `refusal`, `pause_turn`.
- reqwest 0.13.5 source and CHANGELOG: `socks5` vs `socks5h` (examples/tor_socks.rs, src/proxy.rs); 0.13.0 default rustls + aws-lc + platform verifier; `timeout` covers the streaming body, `read_timeout` is idle-based; env proxy handling.
- sqlx (transact-rs/sqlx) issues #459, #1182, #3794, #3920: SQLite single-writer guidance, IMMEDIATE transactions, migration checksum failures, transaction starvation.

Policy and news. Confidence: MEDIUM; the policy is volatile, so re-check before release.

- [The Register: Anthropic clarifies ban on third-party tool access to Claude (2026-02-20)](https://www.theregister.com/2026/02/20/anthropic_clarifies_ban_third_party_claude_access/)
- [VentureBeat: Anthropic reinstates third-party agent usage on Claude subscriptions, with a catch](https://venturebeat.com/technology/anthropic-reinstates-openclaw-and-third-party-agent-usage-on-claude-subscriptions-with-a-catch)
- [TechTimes: Anthropic ends subscription subsidy for agents June 15](https://www.techtimes.com/articles/317625/20260602/anthropic-ends-subscription-subsidy-agents-june-15-credit-pool-replaces-flat-rate-access.htm)
- [GitHub community: Copilot abuse-detection warnings](https://github.com/orgs/community/discussions/160013) and [suspension discussion](https://github.com/orgs/community/discussions/161697)
- [Cloudflare docs: Error 524](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-5xx-errors/error-524/) (100–120 s proxy read timeout)

---
*Pitfalls research for: self-hosted multi-provider AI router in Rust (rsrouter)*
*Researched: 2026-09-28*
