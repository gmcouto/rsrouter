# rsrouter — Agent Guidelines

Planning context lives in `.planning/` (PROJECT.md, REQUIREMENTS.md, ROADMAP.md, research/).

## MANDATORY: Provider fidelity

Every OAuth/subscription provider (Claude Code, Codex, Copilot, Antigravity, Cursor, and any future one), and any provider that has an official client tool, MUST be implemented so its upstream traffic is **indistinguishable from the provider's own official tool**. The only thing allowed to differ is the prompt content itself. rsrouter does not rewrite client prompts to hide its use.

Mimic everything else as closely as possible:
- User-Agent strings and client version headers
- Provider-specific headers (beta flags, client/session/device IDs, request IDs) and their order where it matters
- Magic bytes, body markers, identity/system blocks, and metadata fields the official tool injects
- OAuth flow parameters (client IDs, scopes, redirect URIs, PKCE details) exactly as the official tool uses them
- Transport fingerprint (HTTP version, TLS/ALPN behavior) where feasible
- Refresh, probe and model-discovery calls: these must look like the official tool too

How to implement it:
- Port these contracts from OmniRoute (`https://github.com/diegosouzapw/OmniRoute`) and verify them against the real official tools.
- Keep tool versions and header values configurable, so they can be updated when the official tools change.
- When the connecting client *is* the official tool (e.g. Claude Code → Claude Code provider), forward its real identity headers instead of synthesizing them.
- This rule overrides the general "minimal body modification" principle for these providers. It does NOT allow altering prompt content.
