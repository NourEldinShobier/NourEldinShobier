# Electron vs Tauri: Performance and Resource Usage (as of 2026)

Research caveats: web access was limited (betterstack.com and v2.tauri.app were blocked for direct fetch, so several claims come via search-result summaries, flagged "via search summary"). Independent, reproducible, versioned benchmarks are scarce; most published numbers are blog-grade. No Tauri-vs-Electron file-manager-specific benchmark was found.

## 1. Independent benchmarks: RAM, startup, bundle size, per OS

### Takeaway
Bundle/installer size strongly favors Tauri (roughly 3-60 MB vs 85-320 MB). RAM and startup are much less clear-cut: on Windows both use Chromium (WebView2) so memory is similar; on macOS/Linux Tauri's idle base is lower but real-world web content can cost as much or more (WebKit). Startup is not a reliable Tauri win in the one controlled test found.

### Cited Findings
- Better Stack built the same screen-recorder app in four frameworks. macOS bundle: Tauri 57 MB vs Electron 323 MB. Startup: Electron 273 ms, Tauri 311 ms (Tauri slower; webview init), Deno Desktop 242 ms. Idle memory: Tauri 109 MB vs Electron 128 MB. Most controlled comparison found, but versions not retrieved — [Better Stack](https://betterstack.com/community/guides/scaling-nodejs/tauri-vs-electron-vs-deno-vs-electrobun/) (via search summary; page blocked for fetch)
- A July 2026 article claims hello-world 3.2 MB (Tauri) vs 85 MB (Electron 34.x), idle 42 MB vs 168 MB, cold start 380 ms vs 1,420 ms; methodology not verifiable — [tech-insider](https://tech-insider.org/tauri-vs-electron-2026/) (via search summary; treat as anecdotal/marketing-grade)
- Levminer single-app comparison: Tauri installer about 2.5 MB vs Electron about 85 MB — [Levminer](https://www.levminer.com/blog/tauri-vs-electron) (via search summary)
- Startup claims vary wildly: Noroff case study ~2 s (Tauri) vs ~4 s (Electron); NashTech 200-500 ms vs 1-3 s, no setup given — [Noroff](https://library.noroff.dev/frameworks/tauri/tauri-case-study/), [NashTech](https://blog.nashtechglobal.com/why-frontend-developers-should-learn-tauri-a-practical-perspective/) (anecdotal)
- Tauri issue #5889 (Dec 2022, informal, versions not stated) challenged Tauri's own v1 benchmark (Electron ~500 MB on Linux, 2x Tauri) for ignoring shared memory (default mprof/psutil counts shared Chromium pages repeatedly). Rerun with USS/PSS: default hello-world apps on Ubuntu 22.04: Electron 118 MB USS / 207 MB PSS; Tauri 125 MB USS / 185 MB PSS (mixed). Loading real sites (postman.com): Tauri 421 MB macOS / 581 MB Ubuntu / 399 MB Windows vs Electron 337 / 240 / 318 MB. vscode.dev: Tauri 429 / 572 / 370 MB vs Electron 332 / 222 / 312 MB. Author: Tauri on WebView2 uses about the same as Electron (both Chromium); WebKit adds ~90+ MB on real web apps — [tauri#5889](https://github.com/tauri-apps/tauri/issues/5889) (2022 data; flag as old)
- Measurement caveat: macOS system WebKit helper processes run separately and may not be attributed to the app, producing misleadingly low Tauri numbers; Windows msedgewebview2.exe sums overcount shared pages — summaries from search results of dev articles/issue #5889 (e.g. [DEV comments](https://dev.to/akashpattnaik/leaving-electronjs-to-the-past-3mkl/comments))
- Blog claim of Tauri 2 at 172 MB across six windows vs Electron 409 MB, and idle 30-50 MB for Tauri; no method given — [noqta](https://noqta.tn/en/blog/tauri-2-desktop-apps-rust-web-technologies-2026) (anecdotal)

### Inferences
- Per-OS expectation: Windows ≈ parity on RAM (same Chromium engine; Electron adds Node + own copy, Tauri shares/updates WebView2 runtime); macOS/Linux Tauri lower base footprint but WebKit memory can grow with heavy pages. Installer size is the only consistently large win.
- Startup: Tauri avoids loading Node/Electron main but waits on webview creation; expect same order of magnitude, app-dependent.

### Gaps
- No peer-reviewed or reproducible (scripted, versioned) 2025-2026 benchmark covering Tauri 2.x vs Electron 3x/4x on all three OSes. CPU usage comparisons not found at all.
- Could not read Better Stack page (blocked) for versions/methodology.

## 2. IPC mechanisms and overhead

### Takeaway
Tauri 2 added raw-bytes IPC (serialization-free) but it still rides the webview's fetch-like path; large-payload latency is OS-dependent and was reported poor on Windows. Electron offers MessagePort/utilityProcess, but cross-process ArrayBuffer transfer is generally copy-based; no solid Electron-vs-Tauri IPC benchmark found.

### Cited Findings
- Tauri v2.0.0-rc.3 added an InvokeResponseBody type for binary responses, for performance — [Tauri release notes](https://v2.tauri.app/fr/release/tauri/v2.0.0-rc.3/) (via search summary)
- Tauri discussion #11915: informal Discord test, 10 MB binary IPC about 5 ms on macOS vs about 200 ms on Windows. Maintainer (FabianLars): v2 serialization-free IPC still uses fetch API; unlikely to improve much with system webviews; only WebView2 supports shared memory and it was reported "weirdly slow"; recommends ArrayBuffer returns (Rust to JS), raw requests (JS to Rust), maybe Channels. Sep 2025 follow-up complained 200 ms/10 MB too slow, unanswered — [discussion 11915](https://github.com/orgs/tauri-apps/discussions/11915) (non-scientific)
- Third-party tauri-conduit benchmarks (Rust-side dispatch only, excludes webview bridge): 64 KB payload: Tauri invoke 2.27 ms, its JSON path 834 us, its binary path 202 us — [tauri-conduit](https://github.com/userFRM/tauri-conduit) (vendor-of-alternative numbers)
- Tauri IPC payloads are either raw bytes or JSON, not both; JSON encodes each byte as a number, so send binary frames as raw buffers (connectrpc-tauri uses Channel with protobuf) — [connectrpc-tauri](https://libraries.io/cargo/connectrpc-tauri)
- Electron/Chromium background: ArrayBuffer transfer across processes is implemented as copy-and-neuter; true zero-copy needs shared memory (architectural complexity) — [Chromium code review](https://codereview.chromium.org/1287203003) (old, background only; not Electron-specific)

### Inferences
- For a file manager, avoid sending 100k-entry listings as JSON per call in either framework: paginate/stream chunks (Tauri Channel; Electron MessagePort from utilityProcess) and keep heavy work off the UI thread. Large thumbnail bytes in Tauri: prefer custom URI protocol / asset protocol (browser-native image loading) over invoke, though I found no benchmark confirming.
- Electron's main advantage: Node can run in the renderer-adjacent utilityProcess with MessagePort direct to renderer, bypassing main process; no measurement found.

### Gaps
- No benchmark of Electron contextBridge/ipcRenderer.invoke/MessagePort throughput; no benchmark of Tauri Channel throughput or custom protocol throughput in 2025-2026. Official Tauri docs fetch blocked.

## 3. Rust backend advantages for a file manager

### Takeaway
Rust has real advantages for parallel recursive traversal, search, hashing and a long-lived index (ripgrep/fd lineage, Spacedrive), but a flat single-directory listing is syscall-bound and Rust gains are small; Node + native addons (Rust/napi) can capture most of the benefit. Evidence is thin and mostly project self-reports.

### Cited Findings
- Rush-FS (Rust napi addon for Node) self-reports about 12x faster recursive readdir than Node on a ~30k-entry node_modules tree (281 ms vs 23 ms, Apple Silicon, release build), via jwalk/Rayon — [hyper-fs/rush-fs PR](https://github.com/CoderSerio/hyper-fs/pull/1) (author's benchmark)
- Rust forum: 600,000-file directory, naive Rust read_dir loop 17 s vs Node recursive readdir 8 s (likely non-release build or implementation difference; walkdir was reported faster) — [Rust forum](https://users.rust-lang.org/t/why-is-rust-traversing-directories-slower-than-node/80916)
- jwalk README: parallel walking helps deep trees, not a single directory with many files; same project reports switching to plain std::fs::read_dir for non-recursive calls to avoid overhead — [jwalk](https://codegraph.jelmer.uk/rust-jwalk/0.8.1-1/README.md), [hyper-fs PR](https://github.com/CoderSerio/hyper-fs/pull/1)
- Spacedrive: Rust core + Tauri (migrated to Tauri v2 RC mid-2024; 0.4.2 changelog notes more stable Tauri v2); commits Dec 2025-Jan 2026 added ephemeral index cache, snapshot persistence, enhanced fs watching; development paused for funding in early 2025 per press — [Spacedrive changelog](https://www.spacedrive.com/docs/changelog/alpha/0.4.2), [AlternativeTo](https://alternativeto.net/news/2025/3/spacedrive-pauses-development-due-to-funding-challenges). No indexing benchmarks found.
- ripgrep/fd as Rust examples were not independently benchmarked here (known from background; no source retrieved).

### Inferences
- Rust's win is parallelism, memory control (no GC for millions of entries), and mature crates (ignore, notify, blake3, rayon); Node comparably fast for single-dir readdir and can call the same code via napi-rs. Choice of framework matters less than whether heavy work runs off the UI/main thread.
- Tauri lets the same Rust crate be called in-process without N-API glue; Electron requires native addon packaging per platform/ABI.

### Gaps
- No head-to-head file-manager benchmark (100k+ dirs, watching, hashing, search) of Node vs Rust found; no Tauri-based file manager retrospectives (other than Spacedrive commit logs) found; Spacedrive postmortem not found.

## 4. Chromium vs system webviews: rendering, GPU, consistency

### Takeaway
Electron gives one engine everywhere; Tauri inherits WebView2 (Chromium, good), WKWebView (good but different, some frame-rate/jitter reports) and WebKitGTK (weakest, with documented GPU/driver pitfalls).

### Cited Findings
- Tauri docs: on Linux, WebKitGTK/NVIDIA mismatches (DMABUF renderer) cause blank windows, resize flicker/crashes; workarounds WEBKIT_DISABLE_DMABUF_RENDERER=1 (loses faster path), WEBKIT_DISABLE_COMPOSITING_MODE=1 (disables accelerated compositing); WebGL2 context may silently be software-rasterized and the renderer string is masked, causing low FPS in terminals/editors/maps/charts that are fast in a regular browser; tracked in tauri#9394 — [Tauri Linux Graphics](https://v2.tauri.app/develop/debug/linux-graphics/) (via search summary)
- WebKit bugzilla: GTK async scrolling slow scrolling/CSS animations bug reopened, still reproducible Jan 2023 — [WebKit 221738](https://bugs.webkit.org/show_bug.cgi?id=221738)
- HN anecdote: WebKitGTK single-digit fps scrolling daisyUI components page — [HN item](https://hn.nuxt.dev/item/46082291) (anecdotal)
- macOS: developer reports Tauri/WKWebView capped at 60 fps on ProMotion displays; Babylon.js forum user saw jitter in Tauri vs Chrome/Safari at same reported 60 fps — [Apple forums](https://developer.apple.com/forums/tags/webkit?page=7), [Babylon forum](https://forum.babylonjs.com/t/performance-between-safari-and-wkwebview-tauri/60811) (anecdotal)
- System webview can regress with OS updates (iOS 17.4.1 report: list render 1 s to 3 min) — Apple forums above (iOS, not macOS)
- Linux webview version tracks distro; WebView2 auto-updates — [DEV article](https://dev.to/shrsv/exploring-system-webviews-in-tauri-native-rendering-for-efficient-cross-platform-apps-9hl)

### Inferences
- For a file manager UI (virtualized lists, thumbnails grid), DOM virtualization works on all engines; the risk is Linux scrolling/compositing and GPU path, so test on WebKitGTK with NVIDIA/Wayland early. Video codec support differs (WebKitGTK depends on GStreamer plugins; Electron bundles Chromium codecs with licensing caveats) — not sourced here.

### Gaps
- No large-DOM / virtualized-list benchmark across engines; no WebGPU support matrix sourced; no video codec sources retrieved.

## 5. Real-world heavy apps and pain points

### Takeaway
Electron dominates heavy apps (VS Code, Slack, Discord) with known memory complaints that are largely app-level; Tauri production use at VS Code scale is rare, and some teams moved off Electron to native (Zed) or WebView2 (Teams).

### Cited Findings
- 2026 audit: Teams moved from Electron to WebView2 (Oct 2023); Zed walked away from Electron; VS Code, Slack, Claude Desktop remain — [codenote.net](https://codenote.net/en/posts/famous-electron-apps-2026-research/) (blog audit)
- Discord admitted excessive RAM on Windows and is testing auto-restart above 4 GB when idle over 30 min (Dec 2025) — [AlternativeTo](https://alternativeto.net/news/2025/12/discord-admits-excessive-ram-usage-on-windows-app-and-tests-auto-restarts-as-temporary-fix)
- Slack rewrite: about half the memory, 33% faster startup (date not stated, pre-2026) — [AppleInsider forum](https://forums.appleinsider.com/discussion/212114)
- Wu benchmark (Linux ARM64 VM, software rendering, Wu 1.0.10, Zed 1.21.0, VS Code 1.139.1): idle VS Code about 744 MiB vs Wu about 383 MiB; during whole-repo search Zed 1,395 MiB vs VS Code 824 MiB (Zed higher) — [chatgate.ai](https://chatgate.ai/post/wu) (vendor of Wu; caution)
- Other Zed vs VS Code figures (idle 142 MB vs 730 MB; 222 MB vs 3,549 MB) are from blogs with unclear method — [tech-insider](https://tech-insider.org/zed-vs-vscode-2026/), [DEV](https://dev.to/truongandev/zed-vs-vs-code-in-2026-is-it-worth-switching-17gm) (anecdotal)
- Electron version context: Electron 39 (2025) ships Chromium 142, Node 22.20 and stable ASAR integrity — [Electron blog](https://electronjs.org/de/blog/electron-39-0); a 2026 audit cites Electron 42.0.1 in GitHub Desktop (unverified)
- Conductor (2026) reported choosing Tauri for smaller bundle, faster cold start, snappier UI — [performance.dev](https://performance.dev/the-conductor-rewrite) (single anecdote)

### Inferences
- Memory complaints about Slack/Discord stem from app architecture (leaks, multi-workspace, DOM/layer issues) as well as Chromium baseline; Tauri on Windows would still run Chromium.

### Gaps
- No Obsidian data (Electron) retrieved; no list of large Tauri apps with pain-point reports beyond Spacedrive; no Tauri heavy-app (VS Code class) case studies found. CPU/battery comparisons unsourced.
