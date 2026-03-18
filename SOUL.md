# SOUL.md — OpenClaw Browser

## Identity

You are working on **OpenClaw Browser**, the first native Rust web browser built for AI agent control. This is a FOSS project under AGPL-3.0 maintained by Matt Gates (suhteevah) at Ridge Cell Repair LLC.

## Architecture Context

This is a Cargo workspace with 8 crates:

- `openclaw-browser-core` — CDP browser control via `chromiumoxide`, DOM snapshots, actions
- `openclaw-content-extract` — HTML readability extraction + markdown conversion for LLMs
- `openclaw-cache` — Persistent AI knowledge store: SQLite + Tantivy for cached pages, search results, DOM snapshots. Adaptive staleness with per-domain learned TTLs. Compressed blob storage for raw HTML. The agent never asks the web the same question twice.
- `openclaw-identity` — Encrypted credential vault (AES-256-GCM), browser fingerprint capture/spoofing, auth flow orchestration (password, OAuth, TOTP, SSO), human-in-the-loop callbacks for CAPTCHAs/passkeys. NO .env files for secrets — everything in the encrypted vault.
- `openclaw-search` — Web metasearch (SearXNG-style) + local Tantivy content index
- `openclaw-agent-loop` — Observe→Think→Act AI decision cycle with pluggable LLM backends
- `openclaw-mcp-server` — MCP protocol server (rmcp) for Claude Code/Cursor
- `openclaw-browser` (cli) — Main binary with subcommands

## Non-Negotiable Rules

1. **ALWAYS use `tracing`** — never `println!()`, never `eprintln!()`, never `dbg!()` in library code
2. **ALWAYS use `#[instrument]`** on public async functions
3. **NEVER `.unwrap()` or `.expect()`** in production paths — return `Result<T, E>`
4. **ALWAYS add structured fields** to tracing spans (request IDs, URLs, step numbers)
5. **Token efficiency matters** — every output format is optimized for LLM context windows
6. **Test what you build** — integration tests for every public API
7. **NEVER store secrets in plaintext** — credentials go in the encrypted vault only
8. **Secrets are `SecretString`/`SecretVec`** — zeroized on drop, never logged, never serialized

## Key Design Decisions

- DOM snapshots use **flat element lists with @ref IDs**, not HTML trees
- Content extraction has a **token budget** system for context window management
- The agent loop is **LLM-agnostic** via the `LlmBackend` trait
- MCP server supports both **stdio** (Claude Code) and **SSE** (remote agents)
- Search is **dual-mode**: web metasearch + local Tantivy index
- Chrome is controlled via **CDP (chromiumoxide)**, not WebDriver
- Every page visit is **auto-cached** in the KnowledgeStore with adaptive TTLs
- The agent **searches local cache first**, then the web — only fetches if cache miss or stale
- Credentials use **AES-256-GCM encryption** with argon2id key derivation
- Browser fingerprint is **captured from the user's real browser** and replayed on every headless page
- Auth flows that need humans (CAPTCHA, SMS, passkey tap) go through the **HumanCallback trait**
- Identity profiles **isolate cookies/credentials** — personal vs work vs client

## Security Architecture

- Vault: `~/.openclaw/vault.db` — SQLite, each secret individually AES-256-GCM encrypted
- Master key: argon2id(salt, passphrase) -> 256-bit key, held in memory only
- File permissions: 0600 on vault file (Unix)
- Audit log: every credential access logged in `vault_audit_log` table
- Domain approval: credentials only auto-used on pre-approved domains
- Human gating: `CredentialKind::requires_human_approval()` for passwords, SSH keys, passkeys
- Zeroize: all secrets wrapped in `secrecy::SecretString` — zeroized from memory on drop

## Current State

This project is **scaffolded but not implemented**. The API surface is defined, types are in place, and stubs have TODO markers. Priority implementation order:

1. `browser-core` — Get chromiumoxide wired up, implement `BrowserSession::launch()` and `TabHandle::snapshot()`
2. `identity` — Implement `CredentialVault::encrypt/decrypt`, fingerprint capture via CDP, test vault roundtrip
3. `content-extract` — Implement `strip_noise()` with lol_html, readability algorithm, markdown conversion
4. `cache` — Wire up SQLite schema, implement `put_page()`/`get_page()` with staleness checks
5. `mcp-server` — Wire up rmcp tool registration with browser-core + cache
6. `agent-loop` — Implement LLM backends (Claude API + Ollama)
7. `search-engine` — Implement DuckDuckGo provider + cache integration
8. `cli` — Wire everything end-to-end, add `vault` and `fingerprint` subcommands

## Dependencies to Know

- `chromiumoxide` 0.7 — async CDP client (the foundation of browser control)
- `rmcp` — Official Rust MCP SDK
- `lol_html` — Cloudflare's streaming HTML rewriter (for noise stripping)
- `html5ever` — Servo's HTML5 parser
- `scraper` — CSS selector queries on parsed HTML
- `tantivy` — Rust full-text search engine (like Lucene)
- `rusqlite` — SQLite with bundled build (zero external deps)
- `aes-gcm` — AES-256-GCM authenticated encryption
- `argon2` — Memory-hard password hashing (key derivation)
- `secrecy` — Secret-holding types that zeroize on drop
- `zeroize` — Securely clear memory
- `totp-rs` — TOTP code generation for 2FA
- `flate2` — gzip compression for blob storage
- `blake3` — Fast cryptographic hashing for cache keys
