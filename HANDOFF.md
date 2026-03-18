# HANDOFF.md — Claude Code Session Guide

## What This Is

OpenClaw Browser is a scaffolded Cargo workspace for a Rust-native AI browser — the first one built for LLM control, not humans. The API surface, types, and architecture are defined across 8 crates. Your job is to **implement the TODOs** and get it compiling, then shipping.

Read SOUL.md first. Then read the BLUEPRINT.md for the full god-tier feature roadmap.

## First Steps

```bash
cd openclaw-browser
cargo check 2>&1 | head -80   # See what's broken
```

The workspace may have dependency resolution issues since some crate versions in Cargo.toml are aspirational. Your first task is to **get `cargo check` passing** across all crates by:

1. Fixing any version mismatches in workspace Cargo.toml
2. Adding missing dependencies (e.g., `async-trait` if needed for `LlmBackend` trait)
3. Resolving any import issues in the stub modules
4. Some crates like `rmcp` may not exist yet on crates.io — stub or replace with alternatives

## The 8 Crates

| Crate | Description | Lines |
|-------|-------------|-------|
| `openclaw-browser-core` | CDP browser control via chromiumoxide, DOM snapshots, actions | ~580 |
| `openclaw-content-extract` | HTML readability extraction + markdown conversion | ~210 |
| `openclaw-cache` | SQLite + Tantivy knowledge store with adaptive TTLs | ~1530 |
| `openclaw-identity` | Encrypted credential vault, fingerprint spoofing, auth flows | ~2100 |
| `openclaw-search` | Web metasearch + local Tantivy content index | ~90 |
| `openclaw-agent-loop` | Observe→Think→Act AI decision cycle | ~310 |
| `openclaw-mcp-server` | MCP protocol server for Claude Code/Cursor | ~150 |
| `openclaw-browser` (cli) | Main binary with subcommands | ~185 |

## Implementation Priority

### Phase 1: Get it Compiling (30 min)
- [ ] Fix all `cargo check` errors across workspace
- [ ] Ensure all crate dependencies resolve
- [ ] All stubs compile (can return dummy data)

### Phase 2: Browser Core (2-3 hours)
- [ ] Wire up `chromiumoxide` in `BrowserSession::launch()`
- [ ] Implement `TabHandle::navigate()` with real CDP calls
- [ ] Implement `TabHandle::snapshot()` — **THE KILLER FEATURE**
  - Use CDP `Accessibility.getFullAXTree` for the accessibility tree
  - Use `DOM.getDocument` + `DOM.querySelectorAll` for interactive elements
  - Map each element to a `DomElement` with unique `ref_id`
  - Include bounding boxes from `DOM.getBoxModel`
  - Detect page type (login form, search results, article, etc.)
- [ ] Implement `TabHandle::execute()` for Click, Fill, Scroll, KeyPress
- [ ] Implement `TabHandle::screenshot()` via CDP `Page.captureScreenshot`
- [ ] Implement `TabHandle::eval_js()` via CDP `Runtime.evaluate`

### Phase 3: Identity & Vault (2 hours)
- [ ] Test `CredentialVault` encrypt/decrypt roundtrip
- [ ] Implement fingerprint capture via CDP (launch real browser, run CAPTURE_SCRIPT)
- [ ] Wire CDP overrides injection (`Page.addScriptToEvaluateOnNewDocument`)
  - User-Agent via `Network.setUserAgentOverride`
  - Viewport via `Emulation.setDeviceMetricsOverride`
  - Timezone via `Emulation.setTimezoneOverride`
  - Extra headers via `Network.setExtraHTTPHeaders`
  - Navigator/screen JS overrides via injection script
- [ ] Implement auth flow detection from DOM analysis
- [ ] Wire `TerminalHumanCallback` for interactive CLI prompts
- [ ] Add `vault` and `fingerprint` subcommands to CLI

### Phase 4: Content Extraction (1-2 hours)
- [ ] Implement `strip::strip_noise()` using `lol_html` element content handlers
- [ ] Implement readability algorithm in `readability::extract_article()`
- [ ] Implement `markdown::html_to_markdown()`
- [ ] Implement `markdown::html_to_plain_text()`

### Phase 5: Cache / Knowledge Store (2 hours)
- [ ] Wire up SQLite schema (it's in `src/sql/schema.sql`)
- [ ] Implement `put_page()` — compress HTML, store blob, insert SQLite, index Tantivy
- [ ] Implement `get_page()` with staleness check
- [ ] Implement `search_knowledge()` — Tantivy fulltext + SQLite metadata enrichment
- [ ] Wire auto-caching into `TabHandle::navigate()` — every visit caches
- [ ] Implement `put_search()` / `get_search()` for metasearch result caching
- [ ] Implement adaptive TTL computation in `staleness.rs`
- [ ] Wire domain profile learning (track content_hash changes over time)

### Phase 6: MCP Server (1-2 hours)
- [ ] Read rmcp docs or find alternative Rust MCP SDK
- [ ] Register tools: browse_navigate, browse_click, browse_fill, browse_snapshot, etc.
- [ ] Wire up stdio transport for Claude Code
- [ ] Wire up SSE transport for remote agents
- [ ] Add `browse_vault_store` and `browse_vault_get` tools for credential management

### Phase 7: Agent Loop (2 hours)
- [ ] Implement `ClaudeBackend` using reqwest + Anthropic API
- [ ] Implement `OllamaBackend` using reqwest + Ollama API
- [ ] Implement action parsing from LLM responses (`ACTION:` line extraction)
- [ ] Wire action execution to `TabHandle::execute()`
- [ ] Implement conversation history with token budgeting

### Phase 8: Search (1 hour)
- [ ] Implement DuckDuckGo provider (HTML scraping, no API key needed)
- [ ] Implement Brave Search provider (optional API key)
- [ ] Wire cache: check `get_search()` before hitting providers
- [ ] Wire `search_engine::local::index_page()` into navigation flow

### Phase 9: Polish & Ship
- [ ] `cargo clippy -- -D warnings`
- [ ] `cargo test` — integration tests for core flows
- [ ] `cargo build --release`
- [ ] Test MCP server with Claude Code locally
- [ ] Update README with any API changes
- [ ] `git push` to GitHub

## Key Design Principles

1. **Cache-first everything.** Before any web fetch, check the KnowledgeStore. Only fetch on miss or stale.
2. **Token efficiency.** Every output to the LLM is budgeted. DOM snapshots use @ref IDs not HTML. Content extraction truncates to token budgets.
3. **Secrets never in plaintext.** All credentials in the encrypted vault. SecretString for in-memory. Zeroize on drop.
4. **Fingerprint injection on every navigation.** Before `page.goto()`, always inject the CDP overrides from the active fingerprint profile.
5. **Log everything.** `tracing` with structured fields. Never `println!`. Every state change is observable.
6. **The agent never asks the same question twice.** Adaptive cache TTLs learn per domain.

## Environment

- Rust 1.75+ (2021 edition)
- Chrome/Chromium must be installed for browser-core
- No database server needed (SQLite is embedded)
- No external API keys needed for basic functionality
- Anthropic API key needed for Claude backend in agent-loop
- Ollama needed for local LLM backend

## Testing Strategy

- **browser-core**: Needs Chrome installed. `#[tokio::test]` with real browser.
- **identity**: Unit tests for vault encrypt/decrypt roundtrip. No Chrome needed.
- **content-extract**: Unit tests with fixture HTML files. No Chrome needed.
- **cache**: Unit tests for SQLite + compression roundtrip. No Chrome needed.
- **mcp-server**: Integration tests using MCP test client.
- **agent-loop**: Mock `LlmBackend` for deterministic testing.
- **search-engine**: Mock HTTP responses for provider tests.
