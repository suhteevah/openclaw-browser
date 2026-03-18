# Sevro Engine — Design Specification

**Date:** 2026-03-18
**Status:** Approved
**Author:** Matt Gates (suhteevah) + Claude
**License:** AGPL-3.0 (Servo-originated files retain MPL-2.0)

## Summary

Sevro is a stripped-down fork of Servo embedded in the Wraith Browser monorepo. It replaces Chrome/chromiumoxide as the default browser engine, exposing DOM, layout, and JavaScript execution directly to Rust for sub-millisecond agent operations. Chrome is deprecated gradually via a three-phase migration behind a unified `BrowserEngine` trait.

## Motivation

Wraith currently depends on Chrome via chromiumoxide/CDP for JavaScript execution, DOM snapshots, element interaction, and screenshots. This creates problems:

- **Binary dependency:** Chrome must be installed on every machine running Wraith
- **Performance overhead:** Every agent operation round-trips through CDP serialization (~50-200ms per snapshot)
- **Resource cost:** Chrome uses 100-200MB per tab, limiting concurrency
- **Stealth weakness:** Chrome exposes automation markers that bot detectors flag

Sevro eliminates all of these by running DOM, CSS layout, and SpiderMonkey in-process, with direct Rust access to the live DOM tree.

## Repository Structure

```
wraith-browser/
├── crates/                    # Existing Wraith crates (unchanged)
│   ├── browser-core/          # Gets BrowserEngine trait + SevroEngine backend
│   ├── agent-loop/
│   ├── cache/
│   ├── content-extract/
│   ├── identity/
│   ├── mcp-server/
│   ├── search-engine/
│   ├── scripting/
│   └── cli/
├── sevro/                     # Servo fork (stripped)
│   ├── components/
│   │   ├── dom/               # HTML parsing + DOM tree
│   │   ├── script/            # SpiderMonkey JS bindings
│   │   ├── style/             # Stylo CSS engine
│   │   ├── layout/            # Element positioning / bounding boxes
│   │   ├── net/               # HTTP networking + cookies
│   │   └── url/               # URL handling
│   ├── ports/
│   │   └── headless/          # NEW: Headless agent port (replaces servoshell)
│   └── Cargo.toml
├── Cargo.toml                 # Workspace root includes sevro/
└── BLUEPRINT.md
```

### Key: `sevro/ports/headless/`

This is the new code — a minimal embedding port that exposes the engine to Wraith without windowing, GPU, or UI. It owns the `SevroEngine` struct and the agent-specific APIs.

## Engine Abstraction Layer

All three backends implement a unified trait:

```rust
pub trait BrowserEngine: Send + Sync {
    async fn navigate(&mut self, url: &str) -> Result<()>;
    async fn snapshot(&self) -> Result<DomSnapshot>;
    async fn execute_action(&mut self, action: BrowserAction) -> Result<ActionResult>;
    async fn eval_js(&self, script: &str) -> Result<String>;
    async fn page_source(&self) -> Result<String>;
    async fn screenshot(&self) -> Result<Vec<u8>>;
    fn capabilities(&self) -> EngineCapabilities;
}

pub struct EngineCapabilities {
    pub javascript: bool,
    pub screenshots: bool,
    pub layout: bool,
    pub cookies: bool,
    pub stealth: bool,
}
```

### Backend Capabilities

| Capability | NativeClient | ChromeEngine | SevroEngine |
|-----------|-------------|-------------|-------------|
| JavaScript | No | Yes | Yes |
| Screenshots | No | Yes | No (Phase 1), Yes (Phase 3) |
| Layout/BBox | No | Yes | Yes |
| Cookies | Yes | Yes | Yes |
| Stealth evasions | No | Yes | Yes |

The MCP server and agent loop switch from hardcoded `NativeClient` to `Box<dyn BrowserEngine>`, selected at startup. Auto-fallback: Sevro first, NativeClient if Sevro is not compiled (feature-gated).

## Servo Strip-Down

### Keep

| Component | Purpose | Why |
|-----------|---------|-----|
| `components/script/` | SpiderMonkey bindings, DOM APIs | JS execution for SPAs |
| `components/dom/` | HTML parser, DOM tree | Page structure |
| `components/style/` | Stylo CSS engine | Selector matching, computed styles |
| `components/layout/` | Box construction, positioning | Element bounding boxes |
| `components/net/` | HTTP, cookies, HSTS | Network requests |
| `components/url/` | URL parsing | Servo's URL type |
| `components/constellation/` | Page lifecycle (stripped) | Navigation control |

### Cut

| Component | Reason |
|-----------|--------|
| `components/webrender/` | GPU rendering not needed headless |
| `components/canvas/` | Canvas API not needed for agent |
| `components/webgpu/` | GPU compute not needed |
| `components/compositing/` | Display compositor not needed |
| `ports/servoshell/` | Desktop browser UI |
| `components/webxr/` | VR/AR device APIs |
| `components/bluetooth/` | Bluetooth API |
| `components/gamepad/` | Gamepad API |
| `components/media/` | Audio/video (GStreamer) |
| `components/devtools/` | Inspector protocol |
| `components/gfx/` | Font rasterization (keep minimal subset for layout metrics) |

### Modify

| Component | Changes |
|-----------|---------|
| `constellation/` | Remove multi-process IPC, run everything in-process for speed |
| `script/` | Add direct Rust API to read DOM tree without JS roundtrip |
| `net/` | Wire to Wraith's reqwest + TLS fingerprint profiles |

### Size Estimate

Servo: ~400K lines Rust + ~200K lines SpiderMonkey C++.
Sevro after strip: ~80-120K lines Rust + SpiderMonkey.

## Agent-Specific APIs

The core value of Sevro over headless Chrome — direct Rust access to engine internals:

```rust
impl SevroEngine {
    /// Read the live DOM tree as a Wraith DomSnapshot.
    /// Walks the DOM in Rust, computes layout boxes, extracts interactive elements.
    /// Replaces the 100-line SNAPSHOT_SCRIPT that runs via CDP today.
    fn dom_snapshot(&self) -> DomSnapshot;

    /// Computed style for any element (Stylo query, no JS).
    fn computed_style(&self, node_id: u64, property: &str) -> String;

    /// Bounding box from layout engine (no getBoundingClientRect JS call).
    fn bounding_box(&self, node_id: u64) -> Option<BoundingBox>;

    /// Query selector directly against the DOM tree.
    fn query_selector(&self, selector: &str) -> Vec<u64>;

    /// Read/write element attributes directly.
    fn get_attribute(&self, node_id: u64, name: &str) -> Option<String>;
    fn set_attribute(&mut self, node_id: u64, name: &str, value: &str);

    /// Simulate input events through DOM event system (no CDP dispatch).
    fn click_element(&mut self, node_id: u64);
    fn fill_element(&mut self, node_id: u64, text: &str);
    fn focus_element(&mut self, node_id: u64);

    /// Cookies from network layer directly.
    fn get_cookies(&self, domain: &str) -> Vec<Cookie>;
    fn set_cookie(&mut self, cookie: Cookie);

    /// Network request interception (replaces CDP Network domain).
    fn set_request_interceptor(&mut self, handler: impl Fn(&Request) -> RequestAction);
}
```

### Performance Target

| Operation | Chrome/CDP (current) | Sevro (target) |
|-----------|---------------------|----------------|
| DOM snapshot | 50-200ms | <1ms |
| Page load to DOM-ready | baseline | within 2x |
| Memory per page | 100-200MB | <50MB |
| Concurrent pages (16GB) | 6-8 | 20+ |

## Build System & Distributed Compilation

SpiderMonkey C++ compilation is the bottleneck. Three tiers of acceleration:

### Tier 1: Solo Dev (default)

```toml
# .cargo/config.toml
[build]
rustc-wrapper = "sccache"
```

- `sccache` caches Rust compilation
- Clean build: ~5 min
- Incremental: ~15-30s

### Tier 2: Team / CI

```bash
# Environment
CC="distcc gcc"
CXX="distcc g++"
DISTCC_HOSTS="localhost/8 @buildcluster"
```

- SpiderMonkey C++ distributes across N machines via `distcc`
- Clean build: ~90s with 4 nodes

### Tier 3: CI Cache

- GitHub Actions caches `sccache` + `distcc` artifacts
- PRs only rebuild changed crates + touched C++ files
- Typical PR build: ~30-60s

### Setup Script

```bash
# One-time setup for contributors
cargo install sccache
apt install distcc    # or brew install distcc
./sevro/scripts/setup-build.sh  # auto-configures .cargo/config.toml
```

SpiderMonkey only recompiles when SM version or bindings change. Day-to-day Wraith development won't touch it.

## Migration Path

### Phase 1: Coexistence

- Add `BrowserEngine` trait
- Implement for `NativeClient`, `ChromeEngine`, `SevroEngine`
- MCP server and agent loop use `Box<dyn BrowserEngine>`
- Default: SevroEngine with NativeClient fallback
- ChromeEngine available via `--engine chrome`
- All existing tests pass against all three backends

### Phase 2: Sevro Default

- MCP server and CLI default to Sevro
- Chrome path gets `#[deprecated]` warnings
- Feature-gate Chrome behind `--features chrome-legacy`
- Remove `chromiumoxide` from default dependencies

### Phase 3: Chrome Removal

- Delete `session.rs` (BrowserSession) and CDP code from `tab.rs`
- Remove `chromiumoxide` / `chromiumoxide_cdp` from workspace
- NativeClient stays as ultra-lightweight HTTP-only fallback
- Binary size drops significantly

### Backward Compatibility

- MCP tool interface unchanged (same `browse_navigate`, `browse_click`, etc.)
- Agent loop unchanged (same observe/think/act cycle)
- `--engine chrome` available through Phase 2
- Rhai scripts and WASM plugins unaffected (above engine layer)
- Stealth evasions: JS-injection evasions still work via SpiderMonkey; some become unnecessary since Sevro doesn't expose Chrome automation markers

## Licensing

- **Servo code:** MPL-2.0 (preserved per-file)
- **Wraith code:** AGPL-3.0
- **Combined work:** AGPL-3.0 (MPL-2.0 Section 3.3 permits combination with GPL/AGPL)
- **SpiderMonkey:** MPL-2.0 (same treatment)

MPL-2.0 file headers retained on all Servo-originated files. New files created by the project use AGPL-3.0.

## Testing Strategy

### Engine Conformance Tests

Single test suite parameterized across all three backends:
- Navigate + snapshot
- Click by ref_id
- Fill form
- Eval JS
- Cookie persistence
- Back/forward navigation
- Sevro must pass every test Chrome passes (except screenshot in Phase 1)

### Sevro-Specific Tests

- `dom_snapshot()` returns same elements as SNAPSHOT_SCRIPT via JS
- `bounding_box()` within 1px of Chrome's `getBoundingClientRect`
- React/Next.js/Vue test pages hydrate correctly
- Request interceptor fires, cookies round-trip
- No memory leaks after 1000 navigate cycles (SpiderMonkey GC stress)

### Regression Suite

- ~50 curated real-world URLs (static, SPA, login, search)
- Snapshot output diffed between Chrome and Sevro
- Run nightly in CI

### Performance Benchmarks

- `dom_snapshot()` latency: <1ms
- Page load: within 2x of Chrome
- Memory: <50MB per page
- Concurrency: 20+ pages on 16GB RAM

## Features Deferred (Re-addable)

All stripped Servo components can be re-added by pulling crates back:
- GPU rendering (WebRender) — if visual regression testing is needed
- Canvas/WebGL — if canvas-based sites become a priority
- Media — if audio/video extraction is needed
- DevTools — if remote debugging is wanted
