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
│   ├── scripts/
│   │   └── setup-build.sh     # Build environment setup
│   └── Cargo.toml
├── Cargo.toml                 # Workspace root includes sevro/
└── BLUEPRINT.md
```

### Key: `sevro/ports/headless/`

This is the new code — a minimal embedding port that exposes the engine to Wraith without windowing, GPU, or UI. It owns the `SevroEngine` struct and the agent-specific APIs.

## Fork Point

Fork from Servo's latest stable tag at implementation time (currently v0.0.4, commit TBD). Pin the exact commit hash in `sevro/SERVO_UPSTREAM_COMMIT`. Strategy for upstream:

- Security fixes: cherry-pick from upstream Servo as needed
- Feature updates: evaluate quarterly, merge if beneficial
- SpiderMonkey upgrades: track Servo's SM version, don't independently upgrade

## Engine Abstraction Layer

All three backends implement a unified trait:

```rust
/// Concurrency model: the engine is stored behind Arc<Mutex<dyn BrowserEngine>>.
/// Callers must acquire the lock before any operation. This matches the
/// existing MCP server pattern (Arc<Mutex<NativeClient>>).
///
/// Async boundary: all methods are async to accommodate Chrome's CDP round-trips.
/// NativeClient wraps sync operations in async. SevroEngine wraps sync DOM access
/// in async (zero-cost — no actual suspension). This is a lowest-common-denominator
/// choice that avoids trait bifurcation.
pub trait BrowserEngine: Send + Sync {
    async fn navigate(&mut self, url: &str) -> Result<()>;
    async fn snapshot(&self) -> Result<DomSnapshot>;
    async fn execute_action(&mut self, action: BrowserAction) -> Result<ActionResult>;
    async fn eval_js(&self, script: &str) -> Result<String>;
    async fn page_source(&self) -> Result<String>;
    async fn screenshot(&self) -> Result<Vec<u8>>;
    fn capabilities(&self) -> EngineCapabilities;
    async fn shutdown(&mut self) -> Result<()>;
}

pub struct EngineCapabilities {
    pub javascript: bool,
    pub screenshots: ScreenshotCapability,
    pub layout: bool,
    pub cookies: bool,
    pub stealth: bool,
}

pub enum ScreenshotCapability {
    None,
    ViewportOnly,
    FullPage,
}
```

The `execute_action` method is the single dispatch point for all 17 `BrowserAction` variants. The individual trait methods (`navigate`, `snapshot`, `eval_js`, `screenshot`) exist as convenience shortcuts for the most common operations that the MCP server and agent loop call directly. All other actions (Click, Fill, Select, KeyPress, TypeText, Scroll, Hover, GoBack, GoForward, Reload, Wait, WaitForSelector, WaitForNavigation, ExtractContent) route through `execute_action`.

`shutdown()` handles resource cleanup. For SevroEngine, this triggers SpiderMonkey GC shutdown and mozjs `Runtime` destruction in the correct order. For ChromeEngine, this closes the browser process. For NativeClient, this is a no-op. `Drop` implementations call `shutdown` as a safety net but log a warning if the async shutdown wasn't called explicitly.

### Backend Capabilities

| Capability | NativeClient | ChromeEngine | SevroEngine |
|-----------|-------------|-------------|-------------|
| JavaScript | No | Yes | Yes |
| Screenshots | None | FullPage | None (Phase 1), ViewportOnly (Phase 3) |
| Layout/BBox | No | Yes | Yes |
| Cookies | Yes | Yes | Yes |
| Stealth evasions | No | Yes | Yes |

The MCP server and agent loop switch from hardcoded `NativeClient` to `Arc<Mutex<dyn BrowserEngine>>`, selected at startup. Auto-fallback: Sevro first, NativeClient if Sevro is not compiled (feature-gated).

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
| `constellation/` | See "Constellation Replacement" section below |
| `script/` | Add direct Rust API to read DOM tree without JS roundtrip |
| `net/` | See "Networking Layer" section below |

### Constellation Replacement

Servo's constellation manages navigation, security, iframe isolation, and script scheduling via multi-process IPC. We replace it with a single-process `PageController`:

- **Navigation:** Direct function calls instead of IPC messages. `PageController::navigate(url)` triggers resource loading, HTML parsing, script execution, and layout in sequence on the current thread (or a dedicated engine thread).
- **Page lifecycle events:** `PageController` emits `DomContentLoaded`, `Load`, and `NetworkIdle` events via a channel. The `BrowserEngine::navigate()` impl awaits `DomContentLoaded` by default. Callers can configure which event to wait for.
- **Cancellation:** Navigation can be cancelled by dropping a `CancellationToken`. Long-running JS evaluation has a configurable timeout (default 30s) enforced by SpiderMonkey's interrupt mechanism.
- **Iframes:** Supported. Each iframe gets its own script context but shares the same process. Same-origin iframes share a script thread; cross-origin iframes get isolated script contexts (matching Servo's existing security model, minus the process boundary).
- **Same-origin policy:** Enforced by default for correctness. Can be relaxed globally via `SevroConfig::disable_cors` for agent use cases that need cross-origin DOM access.
- **`window.open()` / popups:** Captured and redirected to a new in-process page context. The agent can inspect popup URLs without spawning new windows.
- **DOM mutation during snapshot:** `dom_snapshot()` acquires a read-lock on the DOM tree. SpiderMonkey script execution is paused (via JS interrupt) for the duration of the snapshot. This guarantees a consistent view. The pause is <1ms for cached snapshots, so it doesn't affect page behavior.

### Networking Layer

Two options for replacing Servo's hyper-based networking:

**Option A (chosen): Wrap reqwest behind Servo's `net_traits`.**

Servo's `net/` component uses a `ResourceThreads` interface defined in `net_traits`. We implement this interface using Wraith's existing `reqwest` client (which already has TLS fingerprint profiles, cookie jars, and gzip/brotli support). This preserves Servo's internal resource loading pipeline (CSP checks, CORS enforcement, cache headers) while routing actual HTTP through reqwest.

Trade-off: more glue code upfront, but CORS/CSP correctness is maintained and TLS fingerprints work automatically.

**Option B (rejected): Gut Servo's net entirely, route all fetches through reqwest.**

Simpler but loses CORS/CSP enforcement. Since some agent tasks involve authenticated browsing on real sites, losing CORS could cause subtle breakage.

### Size Estimate

Servo: ~400K lines Rust + ~200K lines SpiderMonkey C++.
Sevro after strip: ~80-120K lines Rust + SpiderMonkey.

Note: The 80-120K estimate is for Rust code under our control. SpiderMonkey C++ remains unchanged (~200K lines) but is compiled as a static library and rarely touched.

## Agent-Specific APIs

The core value of Sevro over headless Chrome — direct Rust access to engine internals:

```rust
impl SevroEngine {
    /// Read the live DOM tree as a Wraith DomSnapshot — tree walk only, no layout.
    /// Target: <1ms. Acquires DOM read-lock, pauses JS execution briefly.
    fn dom_snapshot_fast(&self) -> DomSnapshot;

    /// Read DOM tree WITH computed layout/bounding boxes.
    /// Triggers style recalculation + layout pass if dirty.
    /// Target: <1ms if layout is cached, 5-50ms if layout recomputation needed.
    fn dom_snapshot_with_layout(&self) -> DomSnapshot;

    /// Computed style for any element (Stylo query, no JS).
    fn computed_style(&self, node_id: u64, property: &str) -> String;

    /// Bounding box from layout engine (no getBoundingClientRect JS call).
    /// May trigger layout recomputation if styles are dirty.
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
    fn set_request_interceptor(
        &mut self,
        handler: Box<dyn Fn(&Request) -> RequestAction + Send + Sync>,
    );
}
```

### Performance Targets

| Operation | Chrome/CDP (current) | Sevro (target) |
|-----------|---------------------|----------------|
| DOM snapshot (no layout) | 50-200ms | <1ms |
| DOM snapshot (with layout) | 50-200ms | 5-50ms (first), <1ms (cached) |
| Page load to DOM-ready | baseline | within 2x |
| Memory per page | 100-200MB | <50MB |
| Concurrent pages (16GB) | 6-8 | 20+ |

## Stealth & Detection Surface

With Sevro replacing Chrome, the detection landscape changes:

**Problems eliminated:**
- `navigator.webdriver` is never set (no CDP)
- No `cdc_*` Chrome DevTools variables
- No Chrome-specific automation markers

**New risks:**
- Missing Web APIs that sites feature-detect (WebGL, WebRTC, etc.)
- SpiderMonkey's JS behavior may differ subtly from V8 on edge cases
- User-Agent must claim to be Chrome but engine is SpiderMonkey

**Mitigation strategy:**
- Sevro's `script/` component stubs out missing APIs to return plausible values (e.g., `navigator.mediaDevices` returns empty array rather than throwing)
- `stealth_evasions.rs` JS injection still works — SpiderMonkey executes the evasion scripts just like Chrome would
- User-Agent configured via `TlsProfile` to match a real Chrome version
- Feature-detection tests: maintain a list of APIs that bot detectors commonly probe, ensure Sevro returns compatible responses

## Build System & Distributed Compilation

SpiderMonkey C++ compilation is the bottleneck.

### Platform Requirements

**Linux (primary):** GCC 10+ or Clang 14+, Python 3.8+, autoconf 2.13, `pkg-config`. Standard package manager installs.

**macOS:** Xcode Command Line Tools, Python 3.8+, autoconf 2.13 via Homebrew. Straightforward.

**Windows:** This is the hard one. SpiderMonkey requires:
- Visual Studio 2022 with C++ workload (MSVC)
- Mozilla's `mozbuild` system (the `mozjs` crate handles this via a 14K-line `build.rs`)
- Python 3.8+ with `pip` packages: `mach`, `mozbuild`
- Specific LLVM/Clang version for certain optimizations
- Clean build on Windows: **15-30 minutes** (not 5 minutes as on Linux)

The `sevro/scripts/setup-build.sh` (and `setup-build.ps1` for Windows) will detect the platform, verify prerequisites, and configure `.cargo/config.toml`.

### Build Tiers

**Tier 1: Solo Dev (default)**

```toml
# .cargo/config.toml
[build]
rustc-wrapper = "sccache"
```

- `sccache` caches Rust compilation
- Clean build: ~5 min (Linux/Mac), ~15-30 min (Windows)
- Incremental: ~15-30s (all platforms)

**Tier 2: Team / CI**

```bash
CC="distcc gcc" CXX="distcc g++"
DISTCC_HOSTS="localhost/8 @buildcluster"
```

- SpiderMonkey C++ distributes across N machines via `distcc`
- Clean build: ~90s with 4 nodes

**Tier 3: CI Cache**

- GitHub Actions caches `sccache` + `distcc` artifacts
- PRs only rebuild changed crates + touched C++ files
- Typical PR build: ~30-60s

SpiderMonkey only recompiles when SM version or bindings change. Day-to-day Wraith development won't touch it.

### Spike: Windows Build Validation

Before full implementation begins, run a spike to verify the `mozjs` crate builds on Windows 10/11 in isolation. If this fails, evaluate:
- Cross-compiling SM on Linux for Windows targets
- Pre-building SM as a static library and shipping it as a binary artifact
- Falling back to Boa for Windows-only builds (reduced capability)

## Migration Path

### Phase 1: Coexistence

- Add `BrowserEngine` trait
- Implement for `NativeClient`, `ChromeEngine`, `SevroEngine`
- MCP server and agent loop use `Arc<Mutex<dyn BrowserEngine>>`
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
- Rhai scripts continue to call through the `BrowserEngine` trait; no change needed
- Stealth evasions: JS-injection evasions still work via SpiderMonkey; some become unnecessary since Sevro doesn't expose Chrome automation markers

## Licensing

- **Servo code:** MPL-2.0 (preserved per-file)
- **Wraith code:** AGPL-3.0
- **Combined work:** AGPL-3.0 (MPL-2.0 Section 3.3 permits combination with GPL/AGPL)
- **SpiderMonkey:** MPL-2.0 (same treatment)

MPL-2.0 file headers retained on all Servo-originated files. New files created by the project use AGPL-3.0.

Note: MPL-2.0 files remain individually available under MPL-2.0. Third parties can extract Sevro's MPL-2.0 files without AGPL obligations on those specific files. AGPL applies to the combined work and any new files.

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| SpiderMonkey won't build on Windows | Blocks Windows dev | Medium | Spike first. Fallback: cross-compile or pre-built binary. |
| Servo APIs change upstream | Fork diverges, merge conflicts | Medium | Pin fork point. Cherry-pick security only. Quarterly review. |
| Bot detectors flag missing Web APIs | Stealth regression | High | Stub missing APIs with plausible values. Maintain detection test suite. |
| SpiderMonkey memory leaks | Crashes on long sessions | Low | GC stress tests. Explicit `shutdown()` with Runtime destruction order. |
| Maintenance cost of fork | Dev time overhead | Medium | Minimize modifications. Stay close to upstream structure. |
| Layout accuracy differs from Chrome | Agent clicks wrong elements | Medium | Regression suite: diff bounding boxes Chrome vs Sevro on 50 URLs. |
| Build complexity deters contributors | Slower adoption | Medium | Setup scripts. Pre-built SM binaries for common platforms. |

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

- `dom_snapshot_fast()` returns same elements as SNAPSHOT_SCRIPT via JS
- `dom_snapshot_with_layout()` bounding boxes within 1px of Chrome's `getBoundingClientRect`
- React 19 / Next.js 16 / Vue 3 test apps hydrate correctly (pinned versions in test fixtures)
- Request interceptor fires, cookies round-trip
- No memory leaks after 1000 navigate cycles (SpiderMonkey GC stress)
- `shutdown()` cleans up SpiderMonkey Runtime without crash or leak

### Regression Suite

- ~50 curated real-world URLs (static, SPA, login, search)
- Snapshot output diffed between Chrome and Sevro
- Run nightly in CI

### Performance Benchmarks

- `dom_snapshot_fast()` latency: <1ms
- `dom_snapshot_with_layout()` latency: <1ms cached, 5-50ms uncached
- Page load: within 2x of Chrome
- Memory: <50MB per page
- Concurrency: 20+ pages on 16GB RAM

## Features Deferred (Re-addable)

All stripped Servo components can be re-added by pulling crates back:
- GPU rendering (WebRender) — if visual regression testing is needed
- Canvas/WebGL — if canvas-based sites become a priority
- Media — if audio/video extraction is needed
- DevTools — if remote debugging is wanted
