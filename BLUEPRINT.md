# BLUEPRINT.md — God-Tier Feature Roadmap

This document captures the full architectural vision for OpenClaw Browser.
It is the output of deep research into the AI browser agent landscape as of March 2026.
Use this as the long-term feature roadmap when planning what to build after the core is working.

## Architecture Pillars

OpenClaw combines five pillars no single project has unified:

1. **Hybrid DOM-and-Vision Perception Engine** — Three modes: accessibility tree snapshots (primary, ~200-400 tokens), DOM distillation with task-aware filtering, and vision-based grounding for canvas/PDF/non-standard UIs
2. **MCTS-Based Self-Improving Action Planning** — Monte Carlo Tree Search over web action sequences (from AgentQ research), with trajectory storage for iterative improvement
3. **Browser-as-API / Automatic API Discovery** — Passively capture XHR/Fetch/WebSocket traffic via CDP, reverse-engineer underlying APIs, generate typed wrappers, skip browser rendering on repeat visits
4. **WASM/Rhai/MCP Plugin Ecosystem** — WASM plugins via Wasmtime for domain-specific extractors, Rhai scripts for quick userscripts, MCP bridge so every plugin is an LLM tool
5. **Browser-Native RAG Knowledge Base** — Every page visited gets embedded, cached, and indexed. Cross-session knowledge graph via petgraph. The browser gets smarter over time.

## Competitive Landscape (March 2026)

| Project | Benchmark Score | Architecture | Key Innovation |
|---------|----------------|-------------|----------------|
| OpenAI CUA | 87% WebVoyager | Vision-primary, pixel-level actions | Trained on massive interaction data |
| Google Mariner | 83.5% | Hybrid DOM+vision | Multi-tab reasoning |
| Agent-E | 73.2% | Text-only DOM | Hierarchical planner-navigator split |
| Claude Computer Use | 72.5% | Screenshot-based | Desktop-level control |
| Stagehand v3 | N/A | CDP-native, model-agnostic | act()/extract()/observe() primitives |
| AgentQ | 81.7% (booking) | MCTS + DPO self-play | 340% improvement via self-training |
| Dia Browser | N/A (acquired $610M) | Full browser fork | Cross-tab synthesis, persistent memory |

## Feature Wishlist (Prioritized)

### Tier 1: Core Differentiators (Build These First After MVP)

**Network Intelligence / API Discovery**
- CDP `Fetch.enable` + `Network` domain for traffic capture
- URL template inference (/users/123 → /users/{id})
- Request/response JSON schema extraction
- Auth pattern detection (Bearer, cookies, API keys)
- GraphQL introspection detection
- Auto-generated typed API wrappers
- "Direct mode" — skip browser, call API directly on cached sites
- Reference: Unbrowse, Notte Anything API, reverse-api-engineer

**Adaptive Self-Healing Selectors**
- Cascade: CSS → XPath → text-content → a11y tree → visual similarity → LLM
- DOM Historian: store tree snapshots, use tree-diff on failure
- Agent-E's "change observation" pattern post-action
- Reference: Healenium (stores DOM trees, finds closest match)

**Parallel Browser Swarm**
- `Target.createBrowserContext` for lightweight isolated sessions (~50MB each)
- tokio Semaphore-based connection pool
- Orchestrator/synthesizer pattern for multi-page research
- Shared KnowledgeStore across all sessions
- Practical limit: 6-8 agents on 16GB RAM, 15-20 on 32GB

### Tier 2: Intelligence Layer (Build After Core Works)

**Vision-Based Grounding (Mode 3)**
- OmniParser v2: YOLOv8 + Florence-2 (0.6s on A100)
- Moondream 0.5B for local sub-second UI understanding
- ShowUI 2B (CVPR 2025, 75.1% ScreenSpot)
- Run via candle or ort in Rust — no Python dependency
- Reference: Crane project (Qwen3-VL 2B at 50x PyTorch speed)

**MCTS Action Planning (from AgentQ)**
- Monte Carlo Tree Search over action sequences
- UCB1 heuristics for exploration/exploitation
- AI critic for branch scoring
- Trajectory storage (success/fail pairs)
- DPO fine-tuning loop for local models
- Reference: AgentQ paper (Llama-3 70B: 18.6% → 81.7%)

**Semantic Page Diffing**
- Compute page embeddings via fastembed (Rust, ONNX)
- Compare embeddings between visits for meaningful changes
- Ignore noise (ads, timestamps, random IDs)
- Generate summaries: "Price dropped from $99 to $79"

**Cross-Site Entity Resolution**
- Extract entities from visited pages
- petgraph knowledge graph for relationships
- Merge information about same entity across sites
- "Tell me everything we know about Product X" queries the graph

### Tier 3: Stealth & Anti-Detection (Critical for Production Use)

**TLS Fingerprint Matching**
- Use `wreq` crate (BoringSSL, 100+ browser profiles)
- Chrome/Firefox/Safari JA3/JA4 fingerprint matching
- HTTP/2 SETTINGS frame order matching
- ALPN negotiation matching
- Header case sensitivity matching

**Behavioral Simulation**
- Bézier mouse curves with Fitts's Law timing
- Overshoot simulation at >500px distances
- Gaussian micro-adjustments
- Bigram-modeled typing (50-200ms inter-key)
- Momentum-based scroll deceleration
- Reference: ghost-cursor library patterns

**Canvas/WebGL Spoofing at Render Level**
- Not JS injection (detectable) — render-level noise
- Reference: Camoufox (C++ implementation)
- Consistent fingerprint profiles (UA + GPU + platform must match)
- 17 evasions from puppeteer-extra-stealth-plugin

### Tier 4: Plugin Ecosystem (Build When Community Forms)

**WASM Plugins via Wasmtime**
- WIT interface files for browser primitives
- Plugins compile from Rust/Go/C/JS to wasm32-wasip2
- Sandboxed, hot-reloadable
- Reference: Zed editor extension system

**Rhai Scripting Layer**
- Embedded scripting for quick extraction rules
- Sandboxed by default, never panics host
- "Userscripts for AI browsers"

**MCP Bridge**
- Every plugin auto-registers as MCP tools
- Domain-specific plugins: Amazon, GitHub, email
- Community plugin registry

### Tier 5: Dream Features (Long-Term Vision)

**Time-Travel Debugging**
- Record full browser state at each step (DOM diffs + network + actions)
- Branch from any point to explore alternatives
- Replay.io-style deterministic replay
- Critical for debugging long multi-step workflows

**Predictive Pre-Fetching**
- Parse agent task plan for likely next URLs
- Pre-fetch in background contexts
- Pre-extract markdown and cache DOM snapshots
- Navigation becomes instant

**Workflow Recording & Replay**
- Record human browser sessions (Workflow Use approach)
- Compile to parameterized, self-healing workflows
- LLM-based selector repair when sites change
- Export as reusable task templates

**Declarative Task DAGs**
- Define complex tasks as directed acyclic graphs
- Independent subtasks run in parallel
- Dependency resolution between steps
- Browser becomes a task execution engine

**Website Capability Fingerprinting**
- On first visit: map login forms, search, APIs, nav structure
- Cache per domain
- Choose optimal strategy: browser UI vs API vs workflow template

**Tor Integration**
- arti-client (official Rust Tor, reached 1.0)
- Per-profile proxy configuration
- DNS-over-HTTPS via hickory-resolver

## Rust Crate Dependencies for Advanced Features

| Feature | Crate | Notes |
|---------|-------|-------|
| TLS stealth | `wreq` | BoringSSL, 100+ browser profiles |
| Local ML | `candle` | HuggingFace, CPU/CUDA/Metal |
| ONNX inference | `ort` | 3-5x faster than Python |
| Embeddings | `fastembed` | ONNX, BGE/E5 quantized |
| OCR | `oar-ocr` | PaddleOCR via ONNX |
| Knowledge graph | `petgraph` | 2.1M+ downloads |
| Plugins | `wasmtime` + `rhai` + `extism` | Multi-layer |
| Tor | `arti-client` | Official Rust Tor |
| DNS | `hickory-resolver` | DoH/DoT support |
| PDF | `lopdf` | Read/write/extract |
| Telemetry | `opentelemetry` | Traces to Grafana/Jaeger |
