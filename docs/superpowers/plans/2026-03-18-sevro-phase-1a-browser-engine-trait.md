# Sevro Phase 1A: BrowserEngine Trait Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Introduce a `BrowserEngine` trait that unifies NativeClient, ChromeEngine, and (future) SevroEngine behind a single object-safe interface, then rewire MCP server, agent loop, and CLI to use it.

**Architecture:** Define the trait in `browser-core/src/engine.rs` using `async-trait` for object safety (`dyn BrowserEngine`). Implement for `NativeClient` (sync-wrapped-in-async) and `ChromeEngine` (existing `BrowserSession` + `TabHandle`). Update MCP server and agent loop to accept `Arc<tokio::sync::Mutex<dyn BrowserEngine>>`. Add `--engine` flag to CLI. Feature-gate Chrome behind `chrome-legacy`.

**Tech Stack:** Rust, `async-trait` crate, tokio, existing chromiumoxide + reqwest dependencies.

**Spec:** `docs/superpowers/specs/2026-03-18-sevro-engine-design.md`

---

## File Map

| Action | File | Responsibility |
|--------|------|----------------|
| Create | `crates/browser-core/src/engine.rs` | `BrowserEngine` trait (async-trait), `EngineCapabilities`, `ScreenshotCapability`, engine factory |
| Create | `crates/browser-core/src/engine_native.rs` | `BrowserEngine` impl wrapping `NativeClient` |
| Create | `crates/browser-core/src/engine_chrome.rs` | `BrowserEngine` impl wrapping `BrowserSession`+`TabHandle` (feature-gated) |
| Modify | `crates/browser-core/src/lib.rs` | Register new modules, re-export trait |
| Modify | `crates/browser-core/src/error.rs` | Feature-gate `Chrome` variant |
| Modify | `crates/browser-core/Cargo.toml` | Add `async-trait`, feature-gate `chromiumoxide` |
| Modify | `Cargo.toml` | Add `async-trait` to workspace deps |
| Modify | `crates/mcp-server/src/server.rs` | Switch from `NativeClient` to `dyn BrowserEngine` |
| Modify | `crates/agent-loop/src/agent.rs` | Switch from `BrowserSession` to `dyn BrowserEngine` |
| Modify | `crates/cli/src/main.rs` | Add `--engine` flag, engine factory |
| Create | `crates/browser-core/tests/engine_conformance.rs` | Parameterized test suite |

---

### Task 1: Add async-trait to workspace + define BrowserEngine trait

**Files:**
- Modify: `Cargo.toml` (workspace root)
- Create: `crates/browser-core/src/engine.rs`
- Modify: `crates/browser-core/src/lib.rs`
- Modify: `crates/browser-core/Cargo.toml`

- [ ] **Step 1: Add async-trait to workspace deps**

In root `Cargo.toml` under `[workspace.dependencies]`:
```toml
async-trait = "0.1"
```

In `crates/browser-core/Cargo.toml`:
```toml
async-trait.workspace = true
```

- [ ] **Step 2: Write failing test**

```rust
// crates/browser-core/tests/engine_trait_compiles.rs
use openclaw_browser_core::engine::{EngineCapabilities, ScreenshotCapability};

#[test]
fn engine_capabilities_construct() {
    let caps = EngineCapabilities {
        javascript: false,
        screenshots: ScreenshotCapability::None,
        layout: false,
        cookies: true,
        stealth: false,
    };
    assert!(caps.cookies);
    assert!(!caps.javascript);
}

#[test]
fn trait_is_object_safe() {
    // This test verifies dyn BrowserEngine compiles.
    fn _accepts_dyn(_: &dyn openclaw_browser_core::engine::BrowserEngine) {}
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cargo test -p openclaw-browser-core --test engine_trait_compiles`
Expected: FAIL — `engine` module doesn't exist

- [ ] **Step 4: Create engine.rs with object-safe trait**

```rust
// crates/browser-core/src/engine.rs
use crate::dom::DomSnapshot;
use crate::actions::{BrowserAction, ActionResult};
use crate::error::BrowserResult;
use async_trait::async_trait;
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ScreenshotCapability {
    None,
    ViewportOnly,
    FullPage,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EngineCapabilities {
    pub javascript: bool,
    pub screenshots: ScreenshotCapability,
    pub layout: bool,
    pub cookies: bool,
    pub stealth: bool,
}

/// Unified browser engine interface.
///
/// Object-safe via async-trait. Stored behind Arc<tokio::sync::Mutex<dyn BrowserEngine>>.
/// All methods are async to accommodate CDP round-trips in ChromeEngine.
/// NativeClient and SevroEngine wrap sync operations in async (zero-cost).
#[async_trait]
pub trait BrowserEngine: Send + Sync {
    async fn navigate(&mut self, url: &str) -> BrowserResult<()>;
    async fn snapshot(&self) -> BrowserResult<DomSnapshot>;
    async fn execute_action(&mut self, action: BrowserAction) -> BrowserResult<ActionResult>;
    async fn eval_js(&self, script: &str) -> BrowserResult<String>;
    async fn page_source(&self) -> BrowserResult<String>;
    async fn current_url(&self) -> Option<String>;
    async fn screenshot(&self) -> BrowserResult<Vec<u8>>;
    fn capabilities(&self) -> EngineCapabilities;
    async fn shutdown(&mut self) -> BrowserResult<()>;
}
```

Register in `lib.rs`:
```rust
pub mod engine;
pub use engine::{BrowserEngine, EngineCapabilities, ScreenshotCapability};
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cargo test -p openclaw-browser-core --test engine_trait_compiles`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add Cargo.toml crates/browser-core/Cargo.toml crates/browser-core/src/engine.rs crates/browser-core/src/lib.rs crates/browser-core/tests/engine_trait_compiles.rs
git commit -m "feat(engine): add object-safe BrowserEngine trait via async-trait"
```

---

### Task 2: Implement BrowserEngine for NativeClient

**Files:**
- Create: `crates/browser-core/src/engine_native.rs`
- Modify: `crates/browser-core/src/lib.rs`

Key API translations (NativeClient → BrowserEngine):
- `navigate(url) -> DomSnapshot` → discard snapshot, return `Ok(())`
- `snapshot() -> Result<DomSnapshot>` (sync) → wrap in async
- `page_source() -> Result<&str>` → `.map(|s| s.to_string())`
- `current_url() -> Option<&str>` → `.map(|s| s.to_string())`
- `execute(action) -> Result<ActionResult>` → delegate directly

- [ ] **Step 1: Write failing test**

```rust
// crates/browser-core/tests/engine_conformance.rs
use openclaw_browser_core::engine::BrowserEngine;
use openclaw_browser_core::engine_native::NativeEngine;

#[tokio::test]
async fn native_engine_capabilities() {
    let engine = NativeEngine::new();
    let caps = engine.capabilities();
    assert!(!caps.javascript);
    assert!(caps.cookies);
    assert!(!caps.layout);
}

#[tokio::test]
async fn native_engine_eval_js_returns_error() {
    let engine = NativeEngine::new();
    let result = engine.eval_js("1+1").await;
    assert!(result.is_err());
}

#[tokio::test]
async fn native_engine_screenshot_returns_error() {
    let engine = NativeEngine::new();
    let result = engine.screenshot().await;
    assert!(result.is_err());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cargo test -p openclaw-browser-core --test engine_conformance`
Expected: FAIL — `engine_native` module doesn't exist

- [ ] **Step 3: Implement NativeEngine**

```rust
// crates/browser-core/src/engine_native.rs
use crate::dom::DomSnapshot;
use crate::actions::{BrowserAction, ActionResult};
use crate::engine::{BrowserEngine, EngineCapabilities, ScreenshotCapability};
use crate::error::{BrowserResult, BrowserError};
use crate::native::NativeClient;
use async_trait::async_trait;
use tracing::instrument;

pub struct NativeEngine {
    client: NativeClient,
}

impl NativeEngine {
    pub fn new() -> Self {
        Self { client: NativeClient::new() }
    }
}

impl Default for NativeEngine {
    fn default() -> Self { Self::new() }
}

#[async_trait]
impl BrowserEngine for NativeEngine {
    #[instrument(skip(self), fields(url = %url))]
    async fn navigate(&mut self, url: &str) -> BrowserResult<()> {
        // NativeClient::navigate returns DomSnapshot; discard it.
        // The snapshot is cached internally and retrievable via snapshot().
        self.client.navigate(url).await?;
        Ok(())
    }

    async fn snapshot(&self) -> BrowserResult<DomSnapshot> {
        self.client.snapshot()
    }

    async fn execute_action(&mut self, action: BrowserAction) -> BrowserResult<ActionResult> {
        self.client.execute(action).await
    }

    async fn eval_js(&self, _script: &str) -> BrowserResult<String> {
        Err(BrowserError::JsEvalFailed(
            "JavaScript not available in native engine".to_string()
        ))
    }

    async fn page_source(&self) -> BrowserResult<String> {
        self.client.page_source().map(|s| s.to_string())
    }

    async fn current_url(&self) -> Option<String> {
        self.client.current_url().map(|s| s.to_string())
    }

    async fn screenshot(&self) -> BrowserResult<Vec<u8>> {
        Err(BrowserError::ScreenshotFailed(
            "Screenshots not available in native engine".to_string()
        ))
    }

    fn capabilities(&self) -> EngineCapabilities {
        EngineCapabilities {
            javascript: false,
            screenshots: ScreenshotCapability::None,
            layout: false,
            cookies: true,
            stealth: false,
        }
    }

    async fn shutdown(&mut self) -> BrowserResult<()> {
        Ok(())
    }
}
```

Register in `lib.rs`:
```rust
pub mod engine_native;
```

- [ ] **Step 4: Run tests**

Run: `cargo test -p openclaw-browser-core --test engine_conformance`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add crates/browser-core/src/engine_native.rs crates/browser-core/src/lib.rs crates/browser-core/tests/engine_conformance.rs
git commit -m "feat(engine): implement BrowserEngine for NativeClient"
```

---

### Task 3: Feature-gate Chrome in error.rs + Cargo.toml

**Files:**
- Modify: `crates/browser-core/src/error.rs`
- Modify: `crates/browser-core/Cargo.toml`

- [ ] **Step 1: Add feature definition to Cargo.toml**

```toml
[features]
default = ["chrome-legacy"]
chrome-legacy = ["dep:chromiumoxide", "dep:chromiumoxide_cdp"]

[dependencies]
chromiumoxide = { workspace = true, optional = true }
chromiumoxide_cdp = { workspace = true, optional = true }
```

- [ ] **Step 2: Feature-gate the Chrome error variant**

In `crates/browser-core/src/error.rs`:
```rust
#[cfg(feature = "chrome-legacy")]
#[error(transparent)]
Chrome(#[from] chromiumoxide::error::CdpError),
```

- [ ] **Step 3: Feature-gate Chrome modules in lib.rs**

```rust
#[cfg(feature = "chrome-legacy")]
pub mod session;
#[cfg(feature = "chrome-legacy")]
pub mod tab;
```

- [ ] **Step 4: Verify builds with default features**

Run: `cargo check -p openclaw-browser-core`
Expected: Compiles (chrome-legacy on by default)

- [ ] **Step 5: Commit**

```bash
git add crates/browser-core/Cargo.toml crates/browser-core/src/error.rs crates/browser-core/src/lib.rs
git commit -m "feat(engine): feature-gate chromiumoxide behind chrome-legacy"
```

---

### Task 4: Implement BrowserEngine for ChromeEngine

**Files:**
- Create: `crates/browser-core/src/engine_chrome.rs`
- Modify: `crates/browser-core/src/lib.rs`

- [ ] **Step 1: Write failing test**

```rust
// append to engine_conformance.rs
#[cfg(feature = "chrome-legacy")]
mod chrome_tests {
    use openclaw_browser_core::engine::BrowserEngine;
    use openclaw_browser_core::engine_chrome::ChromeEngine;
    use openclaw_browser_core::config::BrowserConfig;

    #[tokio::test]
    async fn chrome_engine_capabilities() {
        let engine = ChromeEngine::launch(BrowserConfig::default()).await.unwrap();
        let caps = engine.capabilities();
        assert!(caps.javascript);
        assert!(caps.layout);
    }
}
```

- [ ] **Step 2: Implement ChromeEngine**

```rust
// crates/browser-core/src/engine_chrome.rs
#![cfg(feature = "chrome-legacy")]

use crate::config::BrowserConfig;
use crate::dom::DomSnapshot;
use crate::actions::{BrowserAction, ActionResult};
use crate::engine::{BrowserEngine, EngineCapabilities, ScreenshotCapability};
use crate::error::BrowserResult;
use crate::session::BrowserSession;
use async_trait::async_trait;
use tracing::info;

pub struct ChromeEngine {
    session: Option<BrowserSession>,  // Option for take() in shutdown
}

impl ChromeEngine {
    pub async fn launch(config: BrowserConfig) -> BrowserResult<Self> {
        let session = BrowserSession::launch(config).await?;
        Ok(Self { session: Some(session) })
    }

    fn session(&self) -> BrowserResult<&BrowserSession> {
        self.session.as_ref().ok_or_else(|| {
            crate::error::BrowserError::CdpError("Engine shut down".to_string())
        })
    }
}

#[async_trait]
impl BrowserEngine for ChromeEngine {
    async fn navigate(&mut self, url: &str) -> BrowserResult<()> {
        let session = self.session()?;
        if session.list_tabs().await.is_empty() {
            session.new_tab(url).await?;
        } else {
            let mut tab = session.active_tab().await?;
            tab.navigate(url).await?;
        }
        Ok(())
    }

    async fn snapshot(&self) -> BrowserResult<DomSnapshot> {
        self.session()?.active_tab().await?.snapshot().await
    }

    async fn execute_action(&mut self, action: BrowserAction) -> BrowserResult<ActionResult> {
        let mut tab = self.session()?.active_tab().await?;
        tab.execute(action).await
    }

    async fn eval_js(&self, script: &str) -> BrowserResult<String> {
        self.session()?.active_tab().await?.eval_js(script).await
    }

    async fn page_source(&self) -> BrowserResult<String> {
        self.session()?.active_tab().await?.page_source().await
    }

    async fn current_url(&self) -> Option<String> {
        self.session().ok()?.active_tab().await.ok()
            .map(|tab| tab.current_url.clone())
    }

    async fn screenshot(&self) -> BrowserResult<Vec<u8>> {
        let tab = self.session()?.active_tab().await?;
        match tab.execute(BrowserAction::Screenshot { full_page: false }).await? {
            ActionResult::Screenshot { png_base64, .. } => {
                use base64::Engine;
                base64::engine::general_purpose::STANDARD.decode(&png_base64)
                    .map_err(|e| crate::error::BrowserError::ScreenshotFailed(e.to_string()))
            }
            _ => Err(crate::error::BrowserError::ScreenshotFailed("Unexpected result".into())),
        }
    }

    fn capabilities(&self) -> EngineCapabilities {
        EngineCapabilities {
            javascript: true,
            screenshots: ScreenshotCapability::FullPage,
            layout: true,
            cookies: true,
            stealth: true,
        }
    }

    async fn shutdown(&mut self) -> BrowserResult<()> {
        if let Some(session) = self.session.take() {
            info!("Shutting down Chrome engine");
            session.shutdown().await?;
        }
        Ok(())
    }
}
```

Register in `lib.rs`:
```rust
#[cfg(feature = "chrome-legacy")]
pub mod engine_chrome;
```

- [ ] **Step 3: Run tests**

Run: `cargo test -p openclaw-browser-core --test engine_conformance`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add crates/browser-core/src/engine_chrome.rs crates/browser-core/src/lib.rs crates/browser-core/tests/engine_conformance.rs
git commit -m "feat(engine): implement BrowserEngine for Chrome with proper shutdown"
```

---

### Task 5: Engine Factory

**Files:**
- Modify: `crates/browser-core/src/engine.rs`

- [ ] **Step 1: Write failing test**

```rust
// append to engine_conformance.rs
#[tokio::test]
async fn engine_factory_creates_native() {
    use openclaw_browser_core::engine::create_engine;
    let engine = create_engine("native").await.unwrap();
    let caps = engine.lock().await.capabilities();
    assert!(!caps.javascript);
}

#[tokio::test]
async fn engine_factory_unknown_returns_error() {
    use openclaw_browser_core::engine::create_engine;
    assert!(create_engine("nonexistent").await.is_err());
}
```

- [ ] **Step 2: Implement factory**

Add to `engine.rs`:
```rust
use std::sync::Arc;
use tokio::sync::Mutex;

pub async fn create_engine(name: &str) -> BrowserResult<Arc<Mutex<dyn BrowserEngine>>> {
    match name {
        "native" => {
            Ok(Arc::new(Mutex::new(crate::engine_native::NativeEngine::new())))
        }
        #[cfg(feature = "chrome-legacy")]
        "chrome" => {
            let engine = crate::engine_chrome::ChromeEngine::launch(
                crate::config::BrowserConfig::default()
            ).await?;
            Ok(Arc::new(Mutex::new(engine)))
        }
        "sevro" => {
            Err(crate::error::BrowserError::CdpError(
                "Sevro engine not yet implemented".to_string()
            ))
        }
        "auto" => {
            #[cfg(feature = "chrome-legacy")]
            {
                if let Ok(engine) = crate::engine_chrome::ChromeEngine::launch(
                    crate::config::BrowserConfig::default()
                ).await {
                    return Ok(Arc::new(Mutex::new(engine)));
                }
                tracing::warn!("Chrome not available, falling back to native");
            }
            Ok(Arc::new(Mutex::new(crate::engine_native::NativeEngine::new())))
        }
        other => Err(crate::error::BrowserError::CdpError(
            format!("Unknown engine: '{}'. Options: native, chrome, sevro, auto", other)
        )),
    }
}
```

- [ ] **Step 3: Run tests**

Run: `cargo test -p openclaw-browser-core --test engine_conformance`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add crates/browser-core/src/engine.rs crates/browser-core/tests/engine_conformance.rs
git commit -m "feat(engine): add create_engine factory with auto-fallback"
```

---

### Task 6: Rewire MCP Server

**Files:**
- Modify: `crates/mcp-server/src/server.rs`
- Modify: `crates/mcp-server/Cargo.toml`

This is the largest refactoring task. The MCP server currently calls NativeClient-specific methods. Key translations:

| Current NativeClient call | New BrowserEngine call |
|---|---|
| `browser.navigate(url).await` → returns `DomSnapshot` | `engine.navigate(url).await` then `engine.snapshot().await` |
| `browser.snapshot()` (sync) | `engine.snapshot().await` |
| `browser.page_source()` → returns `&str` | `engine.page_source().await` → returns `String` |
| `browser.current_url()` → returns `Option<&str>` | `engine.current_url().await` → returns `Option<String>` |
| `browser.execute(action).await` | `engine.execute_action(action).await` |
| `browser.needs_javascript()` | Check `engine.capabilities().javascript` or remove (Sevro handles JS) |

- [ ] **Step 1: Update WraithHandler struct**

```rust
// Before:
pub struct WraithHandler {
    tools: Vec<Tool>,
    browser: Arc<Mutex<NativeClient>>,
}

// After:
use openclaw_browser_core::engine::BrowserEngine;

pub struct WraithHandler {
    tools: Vec<Tool>,
    engine: Arc<tokio::sync::Mutex<dyn BrowserEngine>>,
}
```

- [ ] **Step 2: Update constructor**

```rust
impl WraithHandler {
    pub fn new() -> Self {
        Self::with_engine(Arc::new(tokio::sync::Mutex::new(
            openclaw_browser_core::engine_native::NativeEngine::new()
        )))
    }

    pub fn with_engine(engine: Arc<tokio::sync::Mutex<dyn BrowserEngine>>) -> Self {
        // ... tool registration unchanged ...
        Self { tools, engine }
    }
}
```

- [ ] **Step 3: Update all 14 tool dispatch arms**

For each tool that calls `self.browser.lock().await`, change to `self.engine.lock().await` and update method names. The most significant change is `browse_navigate`:

```rust
// Before:
let snapshot = browser.navigate(&input.url).await...;

// After:
engine.navigate(&input.url).await...;
let snapshot = engine.snapshot().await...;
```

Replace `browser.current_url()` with `engine.current_url().await` everywhere.

Remove `needs_javascript()` calls — the engine handles JS capability internally.

- [ ] **Step 4: Run MCP server tests**

Run: `cargo test -p openclaw-mcp-server`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add crates/mcp-server/src/server.rs crates/mcp-server/Cargo.toml
git commit -m "refactor(mcp): switch to dyn BrowserEngine abstraction"
```

---

### Task 7: Rewire Agent Loop

**Files:**
- Modify: `crates/agent-loop/src/agent.rs`

Key translation: The agent currently holds `BrowserSession` and calls `session.active_tab().await?.method()`. With the trait, the engine IS the tab — all calls go directly to the engine.

- [ ] **Step 1: Update Agent struct**

```rust
// Before:
pub struct Agent<L: LlmBackend> {
    pub config: AgentConfig,
    pub session: BrowserSession,
    pub llm: L,
    pub history: StepHistory,
    pub cache: Option<Arc<openclaw_cache::KnowledgeStore>>,
}

// After:
pub struct Agent<L: LlmBackend> {
    pub config: AgentConfig,
    pub engine: Arc<tokio::sync::Mutex<dyn BrowserEngine>>,
    pub llm: L,
    pub history: StepHistory,
    pub cache: Option<Arc<openclaw_cache::KnowledgeStore>>,
}
```

- [ ] **Step 2: Update Agent::new() and Agent::run()**

```rust
impl<L: LlmBackend> Agent<L> {
    pub fn new(
        config: AgentConfig,
        engine: Arc<tokio::sync::Mutex<dyn BrowserEngine>>,
        llm: L,
    ) -> Self {
        Self { config, engine, llm, history: StepHistory::new(), cache: None }
    }

    pub async fn run(&mut self, task: BrowsingTask) -> Result<String, AgentError> {
        if let Some(ref url) = task.start_url {
            let mut engine = self.engine.lock().await;
            engine.navigate(url).await.map_err(AgentError::Browser)?;
        }

        for step in 0..self.config.max_steps {
            // OBSERVE
            let snapshot = {
                let engine = self.engine.lock().await;
                engine.snapshot().await.map_err(AgentError::Browser)?
            };
            // ... rest of loop unchanged, just lock engine for actions ...
        }
    }
}
```

- [ ] **Step 3: Update execute_action**

```rust
async fn execute_action(&self, action: &ParsedAction) -> Result<(), AgentError> {
    let mut engine = self.engine.lock().await;
    match action {
        ParsedAction::Navigate(url) => {
            engine.navigate(url).await.map_err(AgentError::Browser)?;
            // auto-cache
            drop(engine); // release lock before cache work
            self.auto_cache_page(url, "").await;
        }
        ParsedAction::Click(ref_id) => {
            engine.execute_action(BrowserAction::Click { ref_id: *ref_id }).await
                .map_err(AgentError::Browser)?;
        }
        // ... remaining arms translate similarly ...
    }
    Ok(())
}
```

- [ ] **Step 4: Update auto_cache_page**

```rust
async fn auto_cache_page(&self, url: &str, title: &str) {
    let Some(ref cache) = self.cache else { return };
    let html = {
        let engine = self.engine.lock().await;
        match engine.page_source().await {
            Ok(h) => h,
            Err(_) => return,
        }
    };
    // ... rest of extraction + caching unchanged ...
}
```

- [ ] **Step 5: Run agent-loop tests**

Run: `cargo test -p openclaw-agent-loop`
Expected: PASS (action parsing and history tests are engine-independent)

- [ ] **Step 6: Commit**

```bash
git add crates/agent-loop/src/agent.rs
git commit -m "refactor(agent): switch to dyn BrowserEngine abstraction"
```

---

### Task 8: Add --engine flag to CLI

**Files:**
- Modify: `crates/cli/src/main.rs`

- [ ] **Step 1: Add engine arg to CLI struct**

```rust
#[derive(Parser)]
struct Cli {
    #[command(subcommand)]
    command: Commands,

    #[arg(short, long, global = true)]
    verbose: bool,

    /// Browser engine: "auto", "native", "chrome"
    #[arg(long, global = true, default_value = "auto")]
    engine: String,
}
```

- [ ] **Step 2: Create engine at startup, pass to commands**

```rust
// In main(), after tracing init:
let engine = openclaw_browser_core::engine::create_engine(&cli.engine).await?;
```

Update `Commands::Navigate`, `Commands::Task`, `Commands::Extract`, `Commands::Serve` to use the engine instead of constructing `BrowserConfig`/`BrowserSession`/`NativeClient` directly.

- [ ] **Step 3: Test**

Run: `cargo run -p openclaw-browser -- --help`
Expected: Shows `--engine <ENGINE>` with default "auto"

Run: `cargo run -p openclaw-browser -- --engine native search "test"`
Expected: Works with native engine

- [ ] **Step 4: Commit**

```bash
git add crates/cli/src/main.rs
git commit -m "feat(cli): add --engine flag for backend selection"
```

---

### Task 9: Full Integration Test + Clippy

**Files:**
- Modify: `crates/browser-core/tests/engine_conformance.rs`

- [ ] **Step 1: Write cross-engine conformance test**

```rust
use openclaw_browser_core::engine::BrowserEngine;
use openclaw_browser_core::engine_native::NativeEngine;

async fn conformance_navigate_snapshot(engine: &mut dyn BrowserEngine) {
    engine.navigate("https://example.com").await.unwrap();
    let snapshot = engine.snapshot().await.unwrap();
    assert!(!snapshot.elements.is_empty());
    let source = engine.page_source().await.unwrap();
    assert!(source.contains("Example Domain"));
    let url = engine.current_url().await;
    assert!(url.is_some());
}

#[tokio::test]
async fn conformance_native() {
    let mut engine = NativeEngine::new();
    conformance_navigate_snapshot(&mut engine).await;
}
```

- [ ] **Step 2: Verify no-default-features builds**

Run: `cargo check -p openclaw-browser-core --no-default-features`
Expected: Compiles (NativeEngine only, no Chrome)

- [ ] **Step 3: Run full test suite + clippy**

Run: `cargo test && cargo clippy`
Expected: All 306+ tests pass, zero warnings

- [ ] **Step 4: Commit**

```bash
git add crates/browser-core/tests/engine_conformance.rs
git commit -m "test(engine): add cross-backend conformance tests"
```

---

## What This Enables

After Phase 1A, the codebase is engine-agnostic. The MCP server, agent loop, and CLI all operate through `dyn BrowserEngine`. Adding SevroEngine in Phase 1C requires only implementing the trait — no MCP, agent, or CLI changes.

## Next Plans

- **Phase 1B:** `2026-MM-DD-sevro-phase-1b-servo-fork-strip.md` — Clone Servo, strip to headless core, get mozjs building
- **Phase 1C:** `2026-MM-DD-sevro-phase-1c-sevro-engine.md` — Headless port, agent APIs, wire SevroEngine into BrowserEngine trait
