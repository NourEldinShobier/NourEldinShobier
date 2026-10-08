# Electron vs Tauri for media/editor extensions in an extensible file manager (state as of late 2026)

Research caveat: the search tools returned thin results and electronjs.org was blocked by the egress proxy. Many Electron-side and Monaco/CodeMirror/LSP claims could not be sourced and are listed under Gaps or flagged "unverified background" in Inferences. The Tauri-side video, libmpv, PDF-via-PDFium, pty and Spacedrive findings below are sourced.

## Text/code editing, terminal, language servers

### Takeaway
Little was found that directly sourced Monaco/CodeMirror behavior per webview. The terminal path is sourced for Tauri (xterm.js + portable-pty, hand-rolled or via an immature plugin). Electron's node-pty path is not sourced here.

### Cited Findings
- xterm.js is the front-end terminal component used by VS Code, Hyper and Theia. — [search summary of xterm.js description](https://git.iohub.dev/dany/antosdk-apps/raw/tag/v1.x/xTerm/README.md)
- Tauri terminal pattern: xterm.js frontend + Rust `portable-pty` backend, streaming over Tauri events. The reference project marc2332/tauri-terminal is a proof of concept, last commit Nov 2023, no license specified (secondary write-up, anecdotal). — [write-up](https://prompts.brightcoding.dev/blog/marc2332tauri-terminal-build-a-terminal-emulator-in-tauri)
- `tauri-plugin-pty` (npm `tauri-pty`) exists, repo created 2026-01-08, described as in development; docs.rs build of 0.1.0 failed (last good 0.0.8). I could not confirm it uses portable-pty. — [docs.rs](https://docs.rs/tauri-plugin-pty), [Gitee mirror](https://gitee.com/web/tauri-plugin-pty)

### Inferences
- Unverified background (not sourced): Monaco and CodeMirror 6 are pure JS and generally work in WKWebView/WebView2/WebKitGTK, with CM6 lighter and better-suited to many instances and to older WebKitGTK; Monaco needs web-worker setup. Electron guarantees a single known Chromium. Test on the oldest WebKitGTK you support.
- Unverified background: LSP servers are child processes in both. Electron has Node `child_process`; Tauri uses the shell/sidecar plugin or Rust `std::process`. Neither is meaningfully harder.
- Production-grade pty on Tauri means owning a small Rust wrapper; on Electron node-pty is the established route but needs native-module rebuilds per Electron version (unverified background).

### Gaps
- No sourced data on Monaco/CM6 quirks per webview, large-file handling, or LSP in either framework. No sourced node-pty details.

## Video/audio playback

### Takeaway
On Tauri, video relies on the OS webview's codec stack, which is fragile on Linux (GStreamer plugin dependence, version-specific regressions) and platform-dependent on Windows. True native embedding (libmpv) is possible but hacky. Electron's Chromium codec situation could not be sourced here.

### Cited Findings
- WebKitGTK playback depends on installed GStreamer plugins; missing "bad"/libav plugin sets cause H.264/HLS/AAC failures; maintainers ask for `GST_DEBUG="3,webkit*:6"` logs. — [WebKit bug 245850](https://bugs.webkit.org/show_bug.cgi?id=245850), [WebKit bug 192281](https://bugs.webkit.org/show_bug.cgi?id=192281)
- Hardware vs software decoder choice matters (vah264dec worked where libav errored); WebKitGTK 2.36.3 avoided legacy GStreamer VA-API plugins (re-enable via `WEBKIT_GST_ENABLE_LEGACY_VAAPI=1`). — [WebKitGTK 2.36.3 release](https://webkitgtk.org/2022/05/28/webkitgtk2.36.3-released.html)
- HLS was disabled in WebKitGTK 2.39.4, breaking HLS-dependent sites. — [WebKit bug 255495](https://onbugs.webkit.org/show_bug.cgi?id=255495) (via search summary)
- A Tauri app using WebKit2GTK + hls.js failed on Linux with a missing-AAC-plugin warning while working on Windows/Android/Chrome (forum anecdote). — [Manjaro forum](https://forum.manjaro.org/t/webkit2gtk-doesnt-work-well-with-m3u8-streams/168103)
- An MP4 playback regression in one WebKitGTK release was apparently fixed in 2.46.1; an Arch case was an ffmpeg version change, not a WebKit bug. — [WebKit bug 271229](https://bugs.webkit.org/show_bug.cgi?id=271229)
- Tauri discussion (latest comments May and Aug 2026): libmpv in a child WebviewWindow via tauri-plugin-libmpv, with the window repositioned to track an HTML div (not true embedding); native player windows draw above the webview; transparency works only on non-macOS; Rust libmpv bindings poorly maintained; texture rendering via wgpu/canvas is untested; one user streams live-transcoded video to the browser `<video>`; blurymind (Aug 2026) reports WebKitGTK trouble with MP4 streaming (videos not playing to the end even with buffer headers). — [tauri discussion 6343](https://github.com/orgs/tauri-apps/discussions/6343)
- tauri-plugin-libmpv-api v0.3.2: Windows setup downloads libmpv, macOS/Linux need system libmpv; last updated 2025-11-24; low maintenance score. — [package listing](https://classic.yarnpkg.com/en/package/tauri-plugin-libmpv-api)
- MaxVideoPlayer (Tauri v2) has its own tauri-plugin-mpv embedding libmpv in the native window (NSOpenGLView on macOS, EGL with X11/Wayland subsurfaces on Linux; Windows planned). — [MaxVideoPlayer](https://gitblind.noratr.app/MaxMB15/MaxVideoPlayer)
- Tauri's wry custom protocol moved to a version supporting partial content (range) in 2021; current v2 asset-protocol range behavior not confirmed. — [tauri-runtime-wry 0.2.1](https://v2.tauri.app/release/tauri-runtime-wry/v0.2.1/)
- Windows: HEVC requires Microsoft HEVC Video Extensions (paid/OEM, anecdotal sources); AV1 needs the free AV1 Video Extension. Not WebView2-specific. — [Microsoft Q&A](https://learn.microsoft.com/en-us/answers/a/1180572), [VRChat forum](https://ask.vrchat.com/t/hevc-h265-video-playback/17150)
- Chromium docs warn proprietary codec builds may need patent licensing; this applies to custom builds. — [oss.kr thread quoting Chromium docs](https://www.oss.kr/oss_license_qna/show/c4bd70fa-f19a-43ab-bc89-a016a071dbc4)

### Inferences
- A robust Tauri extension for video needs a tiered strategy: native `<video>` for H.264/AAC/VP9 where the platform supports it; fallback to ffmpeg transcode/remux served through a range-capable custom protocol; or libmpv in a native child window for power users.
- Unverified background: Electron ships Chromium with ffmpeg including H.264/AAC in official builds (Chrome-branded ffmpeg codecs), but AC3/DTS/HEVC support varies and MKV is partial; confirm in Electron docs. This gives uniform behavior across OSes, the main Electron advantage for media.
- Electron also faces the overlay problem for libmpv, so native-quality playback is hard in both.

### Gaps
- Electron codec/licensing facts, WebView2 codec specifics, WebKitGTK's current handling of AC3/DTS/MKV/AV1, hardware decode status per platform: not found.

## PDF viewing

### Takeaway
Tauri can render PDFs either with pdf.js in the webview or natively with pdfium-render (needs a bundled PDFium binary; IPC cost per page). Electron's built-in Chromium PDF viewer was not verified.

### Cited Findings
- pdfium-render binds to the PDFium library at run time; the PDFium binary must be supplied by the app; PDFium is not thread-safe (a `thread_safe` feature serializes calls). — [pdfium-render docs](https://docs.rs/crate/pdfium-render)
- A Tauri PDF reader developer reported pdf.js too slow for comic archives and graphics-heavy PDFs and moved to native PDFium (anecdote). A separate project measured Rust/PDFium 2-6x faster on construction PDFs, with pdf.js ahead on near-empty pages (~3 ms vs ~60 ms IPC overhead). — [search summary of project notes](https://gitcode.com/gh_mirrors/op/open-pdf-studio/tree/main/mcp-server)
- PDFium is the open-sourced Chrome PDF engine (BSD-style license). — [InfoQ](https://infoq.com/news/2014/06/google-chrome-pdf-engine-free)
- Electron projects exist using pdf.js and a third-party PDF window module; an old forum post claims Electron 1.6.4+ supports PDF viewing. — [electron-pdf-viewer](https://github.com/hhy5277/electron-pdf-viewer), [MagicMirror forum](https://forum.magicmirror.builders/post/21192)

### Inferences
- Unverified background: Electron includes the Chromium PDF viewer (PDFium) for in-window PDFs, giving search, forms and print with little work; Tauri webviews have uneven native PDF embedding (WKWebView has PDFKit, WebView2 has Edge's viewer, WebKitGTK none), so pdf.js or PDFium is needed for consistency. Slight Electron edge for effort.

### Gaps
- Current Electron PDF/print docs, WebView2 PDF viewer availability, printing in Tauri: not found.

## Image/thumbnail/preview pipelines

### Takeaway
No sourced evidence was gathered for this question beyond the Rust-native-vs-JS PDF anecdote, which suggests native pipelines pay off for heavy content.

### Cited Findings
- See PDF performance anecdotes above (native Rust rendering beats JS on heavy documents but adds IPC overhead). — [open-pdf-studio notes](https://gitcode.com/gh_mirrors/op/open-pdf-studio/tree/main/mcp-server)

### Inferences
- Unverified background: Tauri's Rust core makes `image`, libvips, ffmpeg bindings and libraw/libheif natural for thumbnails and RAW/HEIC with off-main-thread work; Electron can do the same via sharp/native modules or utility processes. Roughly equal, with Tauri slightly more natural if the core is already Rust.

### Gaps
- RAW/HEIC support, memory behavior of large images in each webview, specific benchmarks: not found.

## Platform architecture and extensibility

### Takeaway
Spacedrive (Tauri-based desktop, Rust core) now has a WASM extension system with explicit manifest permissions, but it is early and partly unimplemented. The VSCode-style extension host for Electron was not sourced here.

### Cited Findings
- Spacedrive extensions are Rust compiled to wasm32-unknown-unknown with a `spacedrive-sdk`; `manifest.json` declares permissions (`read_records`, `read_sidecars`, `write_sidecars`, `use_models`); unknown manifest fields rejected; AI inference, tags and custom fields return `NotAvailable`; some declarable permissions have no host implementation; `test-extension` is the reference. — [Spacedrive extensions README](https://raw.githubusercontent.com/spacedriveapp/spacedrive/main/extensions/README.md)
- WASM extension integration commit dated around Oct 2025; Tauri 2.0 upgrade May 2024; a mobile app in React Native added Dec 2025. Sourced via a third-party mirror. — [mirror](https://code.morphllm.com/spacedriveapp/spacedrive/commits/branch/main/extensions/README.md)
- General: the WASM Component Model/WIT is the main 2026 WASM change; Zed uses WASM for plugins (secondary blog sources). — [WebAssembly in 2026](https://devstarsj.github.io/2026/07/10/webassembly-2026-complete-guide/)

### Inferences
- WASM sandboxed extensions suit data/job-type extensions (thumbnailers, indexers) and fit a Rust core, but UI-heavy extensions (editor, video) need a separate UI-extension mechanism (iframes/webview per extension or host-provided components).
- Electron permits the VSCode model: a separate Node extension-host process plus webview-based UI contributions; this is more capable but gives extensions full Node power unless sandboxed (unverified background).

### Gaps
- Whether Spacedrive's Tauri-based UI is still current in 2026, lessons learned from Tabby/other Electron file managers, VSCode extension host details: not found.

## Net assessment (provisional)

- Text/code editor: roughly equal; Electron is slightly safer for engine consistency (unverified).
- Terminal/LSP: Electron easier (node-pty mature, unverified); Tauri workable via portable-pty with custom glue (sourced).
- Video/audio: Electron clearly easier for in-page playback uniformity (unverified on codec detail); Tauri hard on Linux (sourced), with libmpv/ffmpeg workarounds that are hacky (sourced).
- PDF: Electron easier out of the box (unverified); Tauri good with pdfium-render or pdf.js, more work (sourced).
- Images/heavy previews: equal to slight Tauri advantage with a Rust core (unverified).
- Extension platform: Tauri plus WASM is a sandboxed model with a young ecosystem (sourced); Electron plus an extension host is proven but heavier (unverified).
