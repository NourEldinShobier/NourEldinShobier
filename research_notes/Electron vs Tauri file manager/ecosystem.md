# Electron vs Tauri: Ecosystem, Maturity, Community Health (as of Oct 2026)

Research caveat: egress proxy blocked GitHub API, npm API, v2.tauri.app, electronjs.org and desktopinsights.com fetches; only WebSearch snippets and one crates.io fetch worked. Many requested stats (stars, npm downloads, contributor counts, app lists) could NOT be verified and are listed under Gaps. Nothing below is from memory unless labeled.

## Ecosystem size (stars, downloads, apps, plugins, libraries)

### Takeaway
Only one hard number was obtained: the `tauri` crate had ~35.0M all-time and ~13.6M recent (90-day) crates.io downloads on 2026-10-01-ish. Electron/Tauri star counts, npm downloads and app lists could not be verified with the available tools.

### Cited Findings
- crates.io `tauri` crate: 34,992,422 total downloads, 13,649,659 recent downloads (crates.io "recent" = last 90 days), max version 3.0.0-alpha.4, updated 2026-10-01 (fetched 2026-10-08) — [crates.io API](https://crates.io/api/v1/crates/tauri)
- Tauri plugin model: plugins are Cargo crates plus optional NPM API package (`tauri-plugin-{name}` / `tauri-plugin-{name}-api`), may include Android/Swift mobile code; core deliberately omits non-universal features — [Tauri docs](https://v2.tauri.app/develop/plugins/), [architecture](https://v2.tauri.app/concept/architecture/)
- Tauri uses OS webviews (per search summary of docs); Electron bundles Chromium+Node — [Tauri Architecture](https://v2.tauri.app/concept/architecture/)
- Migration examples: Vikunja desktop moved from an Electron wrapper to Tauri (build moved from electron-builder to `cargo tauri build`) — [Vikunja commit](https://kolaente.dev/CL0Pinette/desktop/commit/18ccae4f71d49e2333f85e58494182f6b698da79); Aptabase blog on choosing Tauri over Electron — [Aptabase](https://aptabase.com/blog/why-chose-to-build-on-tauri-instead-electron); "Why I chose Tauri instead of Electron" — [DEV](https://dev.to/goenning/why-i-chose-tauri-instead-of-electron-34h9). Motivations cited: size, memory, free OS-webview security updates.
- Counter-evidence: "We tried Tauri, and we failed" post cites webview variation as the main trade-off (details not retrieved) — surfaced via [search results](https://dev.to/best_codes/why-i-ditched-electron-for-tauri-588k/comments); a commenter says needing to test many runtimes is the main reason most devs don't leave Electron. Single anecdotal sources.
- Discord community has a feature-request thread asking Discord to switch to Tauri; this is a user suggestion, not a Discord decision — [Discord support](https://support.discord.com/hc/de/community/posts/18470216016919-Switch-from-Electron-to-Tauri-for-the-desktop-app)

### Inferences
- The ~13.6M/90-day crate downloads includes CI and transitive pulls; not comparable to npm `electron` downloads without the same methodology.
- No evidence found of a notable Tauri-to-Electron reversal; absence may reflect search limits.

### Gaps
- GitHub stars/forks/contributors for electron/electron, tauri-apps/tauri, plugins-workspace (GitHub blocked).
- npm `electron` download counts (api.npmjs.org blocked).
- Verified lists of production apps (VSCode, Slack, Discord, Figma, Obsidian, Notion, Postman, 1Password; Spacedrive etc.) with sources; Electron-to-Tauri migrations by big names.
- xterm.js/node-pty vs Rust pty crates, Monaco/CodeMirror, electron-builder/forge vs tauri-plugin-updater comparisons: not researched successfully.
- State of JS / Stack Overflow survey figures.

## Maturity, stability, governance, funding

### Takeaway
Tauri 2.0 stable shipped 2024-10-02; by Oct 2026 a Tauri 3 alpha line exists (CEF work appears to be in it). Electron keeps an 8-week major cadence with only the latest 3 majors supported. Tauri is governed as a Programme in the Dutch Commons Conservancy with CrabNebula as commercial partner.

### Cited Findings
- Tauri core v2.0.0 released 2024-10-02 per changelog; RC targeted end of Aug 2024 so stable slipped ~1 month — [Tauri release RC post](https://v2.tauri.app/blog/tauri-2-0-0-release-candidate/), [tauri changelog](https://v2.tauri.app/release/tauri/)
- Tauri v3 alpha: crates.io shows max version 3.0.0-alpha.4 (2026-10-01 fetch); a May 24, 2026 digest reported v3.0.0-alpha.7 (version numbers conflict between sources; possibly different crates, e.g. CLI vs core) — [crates.io](https://crates.io/api/v1/crates/tauri), [Rust daily digest](https://buttondown.com/thewang/archive/rust-daily-digest-2026-05-24/)
- Electron: 8-week major cadence (4-week alpha, 4-week beta before stable); supports latest 3 stable majors, latest minor only; Chromium bumped within ~1-2 weeks of a Chromium stable — [Electron timelines](https://electronjs.org/de/docs/latest/tutorial/electron-timelines)
- Electron 40 stable 2026-01-13 (Chromium M144), EOL 2026-06-30; Electron 39 (M142) EOL 2026-05-05; Electron 38 (M140) EOL 2026-03-10; Electron 37 EOL 2026-01-13; 43.0.0 beta on 2026-06-08 (Chromium 150) — [Electron timelines](https://electronjs.org/de/docs/latest/tutorial/electron-timelines), [releases](https://releases.electronjs.org/)
- Tauri governance: Programme within The Commons Conservancy (Dutch Stichting); Board is the central decision body; domain leads and Working Group — [Statutes](https://www.commonsconservancy.org/dracc/0035), [Governance](https://v2.tauri.app/de/about/governance/)
- Funding: Open Collective donations; CrabNebula is an official partner (distribution platform, free DevTools; Tauri post "Strengthening Tauri: Our Partnership with CrabNebula"); CrabNebula founder Daniel Thompson-Yvetot reportedly chairs the Tauri board — [Tauri blog](https://v2.tauri.app/blog/3/), [CrabNebula Cloud docs](https://v2.tauri.app/distribute/pipelines/crabnebula-cloud/); chair claim from a profile snippet, weakly sourced
- CEF: no official doc found for a Tauri CEF runtime. Indirect: March 2026 task notes CEF webviews lacking back/forward/reload/stop in public API; May 2026 v3 alpha fix about creating CEF data dir — [task](https://hub.harborframework.com/tasks/abundant/tauri-apps__tauri-14661), [digest](https://buttondown.com/thewang/archive/rust-daily-digest-2026-05-24/)

### Inferences
- Electron: with ~8-week majors and 3 supported, an app must upgrade roughly every 6 months to stay supported (3 x 8 wks = 24 wks).
- CEF in Tauri 3 alpha suggests an experimental path to a bundled Chromium, which would narrow the webview-inconsistency gap but remove the size advantage; unconfirmed.
- Single-company dependence (CrabNebula) is a bus-factor risk; not quantified.

### Gaps
- Tauri 2.x release cadence/breaking changes, security advisories; Electron OpenJS Foundation status (not retrieved); security patch lag numbers; contributor counts; sponsor list/amounts; confirmation of CEF runtime status and repo name.

## Extensibility (plugin/extension systems)

### Takeaway
Tauri plugins are compile-time Rust crates, not a runtime third-party extension mechanism. No existing Tauri project running WASM/Extism/Wasmtime or a VSCode-style JS extension API was found.

### Cited Findings
- Tauri plugins are authored as Cargo crates with optional JS bindings, built into the app at compile time — [Tauri plugin docs](https://v2.tauri.app/develop/plugins/)
- Extism is a plugin framework over WASM engines (Wasmtime etc.) with a Rust SDK supporting host functions, designed for third-party code sandboxing; latest crate 1.30.0 — [docs.rs extism](https://docs.rs/crate/extism)
- A search for Tauri+Extism/Wasmtime integration returned no existing project or official guidance — [search summary of docs above]

### Inferences
- A VSCode-class extension host in Tauri would be custom work: e.g. sidecar Node/Deno process, WASM via Extism, or sandboxed iframes/workers. Feasible in principle, no reference implementation found.
- VSCode's extension-host model (separate Node process) maps naturally onto Electron since Node is already bundled; this is background knowledge, not sourced here.

### Gaps
- VSCode extension host architecture sources; Zed/Lapce/other WASM extension examples; projects attempting JS extension API in Tauri; sandbox security evidence.

## Developer experience

### Takeaway
Sources found are anecdotal: Rust backend is a learning hurdle but some say little Rust is needed; Tauri's crate ecosystem is smaller than npm.

### Cited Findings
- Developers in a discussion said they didn't want to learn Rust; reply says Tauri does much of the Rust for you; an article notes Rust libraries are far fewer than NPM — [Search results compiled](https://itnext.io/why-i-chose-tauri-instead-of-electron-e67b34f8857d)
- Tauri offers CrabNebula DevTools for debugging — [docs](https://v2.tauri.app/develop/debug/crabnebula-devtools/)
- Electron upgrade burden: see timelines above (support for only 3 majors).

### Inferences
- Rust compile times and cross-compilation friction are widely reported but not verified here.

### Gaps
- Build times, cross-compilation, hiring pool, docs quality, Discord/Stack Overflow community sizes, bundle size benchmarks: no sourced data.

## Risks and alternatives

### Takeaway
Main Tauri risk is webview variance (esp. Linux WebKitGTK); main Electron risks are size and a fast upgrade treadmill. Alternatives not sourced.

### Cited Findings
- Commenters identify testing across multiple OS webviews as Tauri's key trade-off — [DEV comments](https://dev.to/best_codes/why-i-ditched-electron-for-tauri-588k/comments)
- Search for Linux/WebKitGTK specifics returned nothing citable. Unverified background knowledge (not sourced): WebKitGTK versions vary by distro, with CSS/media/WebRTC gaps.

### Inferences
- A file-manager/VSCode-class app depending on terminal rendering (xterm.js) and Monaco would be most exposed to webview differences on Linux.

### Gaps
- Wails, Neutralino, Electrobun, Dioxus, Flutter desktop, Qt: not researched. Electron security patch burden: no incident data gathered.
