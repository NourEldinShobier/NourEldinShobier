# Electron vs Tauri 2.x feature/API gap for a desktop file manager (state: Oct 2026)

Research limits: tauri.app/v2.tauri.app and electronjs.org were blocked by the network proxy, and GitHub search/API was blocked. I could only fetch github.com blob/issue pages and run web searches. Many items on the requested checklist (tray, global shortcuts, menus, notifications, deep links, Playwright, crash reporting, CSP/capabilities, Windows ARM, mobile) could not be verified and are listed under Gaps rather than asserted.

## Windowing and OS integration (drag-out, clipboard, icons, single instance, protocols)

### Takeaway
The biggest file-manager gap is native drag-OUT: Electron has it built in (`webContents.startDrag`), while Tauri core refused it (issue closed "not planned") and relies on a third-party CrabNebula plugin. Tauri's official clipboard plugin documents text only, and Electron has a built-in file-icon API (`app.getFileIcon`) that I found no Tauri equivalent for.

### Cited Findings
- Electron drag-out: call `webContents.startDrag({file, icon})` from the main process in response to the renderer's `ondragstart` (renderer prevents default and sends IPC) — [Electron docs: native file drag & drop](https://github.com/electron/electron/blob/main/docs/tutorial/native-file-drag-drop.md)
- Tauri core issue "Support for dragging files from Tauri window to filesystem" (#6664, opened by someone building a file manager) was closed as "not planned"; Tauri already supports dropping files INTO the window, not out — [tauri #6664](https://github.com/tauri-apps/tauri/issues/6664)
- Workaround: CrabNebula `drag-rs` / `tauri-plugin-drag` (JS `startDrag` with file paths or data buffers plus a preview icon) covers macOS, Windows and Linux (via GTK). README says tested with Tauri v2 windows, tao, wry. On Linux the Rust API needs a `gtk::ApplicationWindow`; winit cannot use it on Linux. The repo showed 16 open issues (contents not read) — [drag-rs README](https://github.com/crabnebula-dev/drag-rs); [lib.rs/CrabNebula docs summary](https://docs.crabnebula.dev/plugins/drag-rs/)
- Tauri official clipboard plugin docs describe only text (`writeText`/`readText`; Rust `write_text`/`read_text`); no image/file clipboard documented on that page — [tauri-docs clipboard](https://github.com/tauri-apps/tauri-docs/blob/v2/src/content/docs/plugin/clipboard.mdx)
- Electron `app.getFileIcon(path, {size})` returns a NativeImage; small=16, normal=32, large=48 on Linux/32 on Windows, "large" unsupported on macOS; on Windows it gives extension icons and embedded exe/dll/ico icons; on Linux/macOS icons depend on the app associated with the MIME type — [Electron app API](https://github.com/electron/electron/blob/main/docs/api/app.md)
- Electron single instance: `app.requestSingleInstanceLock(additionalData)` emits `second-instance` with argv in the primary; on macOS/Linux messages over 32 MB are dropped — [Electron app API](https://github.com/electron/electron/blob/main/docs/api/app.md)
- Electron protocol handlers: `app.setAsDefaultProtocolClient`; on macOS protocols must be declared in Info.plist at build time; in packaged Windows Store (appx) it always returns true but the protocol must be in the manifest — [Electron app API](https://github.com/electron/electron/blob/main/docs/api/app.md)
- Tauri core Cargo features show `tray-icon`, `linux-libxdo` (menus/tray on Linux), `protocol-asset` (asset protocol with HTTP range), `macos-private-api` (marked for removal in v3); tauri crate version on `dev` is 2.12.1 — [tauri Cargo.toml](https://github.com/tauri-apps/tauri/blob/dev/crates/tauri/Cargo.toml)

### Inferences
- A Tauri file manager must depend on a third-party plugin for drag-out (maintained by CrabNebula, not core) and its Linux path is GTK-window-specific, which is a risk on Wayland/other toolkits (not verified).
- Per-file shell icons/thumbnails in Tauri would need custom Rust (or community crates) per OS; I found no official API.

### Gaps
- Could not verify Tauri global-shortcut, native/context menu, notification, deep-link, single-instance, file-association plugin status/limits (docs blocked).
- Could not verify Tauri clipboard image/file support in community plugins (e.g. clipboard-manager variants) or Electron's `clipboard` file-copy behavior.
- Could not read the 16 open drag-rs issues or any Tauri open drag-in/out bug tracker entries (e.g. drag-drop event vs HTML5 DnD conflict on Windows).
- Frameless/custom titlebar, multi-window comparison not verified.

## Process model and extension host isolation

### Takeaway
Electron offers `utilityProcess` (Node.js child via Chromium Services API, MessagePort transfer) as a natural VSCode-style extension host. Tauri's equivalent is sidecar binaries spawned via the shell plugin with explicit capability grants, or Rust threads/processes; there is no built-in Node host.

### Cited Findings
- `utilityProcess.fork` creates a child with Node.js and message ports enabled, launched through Chromium's Services API; supports `env`, `execArgv`, `cwd`, `serviceName`, stdio piping, session/partition for networking, and on macOS `disclaim` (separate TCC identity); main sends messages and transfers `MessagePortMain`; child uses `process.parentPort`; `kill()` is SIGTERM on POSIX without a guarantee — [Electron utilityProcess](https://github.com/electron/electron/blob/main/docs/api/utility-process.md)
- Tauri sidecars: external executables listed under `bundle.externalBin`, each needing a target-triple-suffixed copy per architecture; run via shell plugin `Command.sidecar` (JS) or `app.shell().sidecar` (Rust); capabilities `shell:allow-execute` / `shell:allow-spawn` with `"sidecar": true` and argument validators — [Tauri sidecar docs](https://github.com/tauri-apps/tauri-docs/blob/v2/src/content/docs/develop/sidecar.mdx)
- Multiwebview in Tauri 2 is behind the `unstable` Cargo feature (enables `unstable` in tauri-runtime-wry); the feature still exists in the dev branch (core 2.12.1). Original beta announcement described it as unfinished while the API design was reviewed — [tauri Cargo.toml](https://github.com/tauri-apps/tauri/blob/dev/crates/tauri/Cargo.toml); [v2 beta announcement](https://v2.tauri.app/blog/tauri-2-0-0-beta/)

### Inferences
- An extension host in Tauri would be a Node/Deno/other sidecar (shipped per target triple, adding binary size back) or a Rust-embedded runtime (e.g. WASM) with IPC over stdio/sockets; in Electron it is a utilityProcess with MessagePorts directly to renderers (a renderer-to-extension-host port without routing through main is a documented MessagePort pattern, not confirmed in the pages I read).
- Multiwebview remaining behind `unstable` means layouts of multiple embedded views (e.g. preview panes) carry API-stability risk.

### Gaps
- Did not verify Electron worker threads / renderer-direct MessagePort docs, or Tauri's plugin isolation model beyond sidecars.

## Webview/engine differences

### Takeaway
Electron ships one Chromium, so codecs, WebGPU etc. are consistent; Tauri uses WebView2 (Windows), WKWebView (macOS), WebKitGTK (Linux), so media and web-API support depends on OS version and, on Linux, installed GStreamer plugins. I could not retrieve Tauri's official webview-version table.

### Cited Findings
- Third-party 2026 comparisons state Tauri uses WebView2/WKWebView/WebKitGTK 4.1 and that Linux layout differs and WebKitGTK lags; Electron gives identical rendering. These are blog sources, treat as rough — [TeamDev](https://teamdev.com/mobrowser/blog/top-5-electron-alternatives-in-2026/); [PkgPulse](https://www.pkgpulse.com/guides/electron-vs-tauri-2026)
- WebKitGTK video playback depends on installed GStreamer plugins (e.g. gstreamer1.0-libav for H.264) and has had GPU-specific regressions (NVIDIA, DMABUF env var workarounds). These are WebKit bug-tracker reports for Epiphany/WebKitGTK, not Tauri-specific, but apply to Tauri on Linux. A Manjaro forum user reported Tauri HLS video breaking on seek — [WebKit bug 227121](https://bugs.webkit.org/show_bug.cgi?id=227121); [WebKit bug 260654](https://www2.webkit.org/show_bug.cgi?id=260654); [Manjaro forum](https://forum.manjaro.org/t/webkit2gtk-doesnt-work-well-with-m3u8-streams/168103)
- Tauri Cargo feature `devtools` exists in core (devtools available via feature flag) — [tauri Cargo.toml](https://github.com/tauri-apps/tauri/blob/dev/crates/tauri/Cargo.toml)

### Inferences
- For a file manager previewing arbitrary media (HEVC/AV1/MKV), Linux Tauri previews depend on user-installed codecs; Electron's bundled FFmpeg/Chromium set is fixed (exact Electron codec set, e.g. proprietary codecs build flag, not verified).

### Gaps
- No verified data on WebGPU, WebCodecs, SharedArrayBuffer/cross-origin isolation, Service Workers, File System Access API, Web Audio, PDF viewing for either side. The Tauri webview-versions page (404 on GitHub path; v2.tauri.app blocked) was not read.

## Auto-update, signing, packaging, crash reporting, platforms

### Takeaway
Tauri 2 official distribution docs list Debian, Snap, AppImage, Flatpak, RPM, AUR, Mac App Store, DMG, Microsoft Store and a Windows installer, but historical Flatpak and MSIX issues show these were gaps; the Tauri updater mandates its own signature scheme and has a Windows quit-before-install limitation.

### Cited Findings
- Tauri distribute page lists: Linux (Debian, Snap, AppImage, Flatpak, RPM, AUR), macOS (Mac App Store, DMG), Windows (Microsoft Store + installer), Android (Google Play), iOS (App Store) — [tauri-docs distribute](https://github.com/tauri-apps/tauri-docs/blob/v2/src/content/docs/distribute/index.mdx)
- Tauri Flatpak bundling request #3619 (bundle as Flatpak) appears open (opened Mar 2022, labels bundler, platform: Linux); the page captured showed no comments. A Flathub forum thread asks for help implementing Flatpak support — [tauri #3619](https://github.com/tauri-apps/tauri/issues/3619); [Flathub discourse](https://discourse.flathub.org/t/help-tauri-implement-flatpak-support/5993)
- A developer write-up claims Tauri does not natively support MSIX packaging (Store submission needed a custom pipeline); possibly dated, single source — [scour.ing/dev.to](https://scour.ing/@hello/p/https://dev.to/octasoft-ltd/building-wsl-ui-the-microsoft-store-journey-2428)
- Tauri updater: signature mandatory and cannot be disabled; lost private key means no updates to existing installs; artifacts: AppImage (Linux), `.app.tar.gz` (macOS), NSIS setup.exe/MSI (Windows); Windows installMode passive/basicUi/quiet and the app must quit before install; TLS enforced — [Tauri updater docs](https://github.com/tauri-apps/tauri-docs/blob/v2/src/content/docs/plugin/updater.mdx)
- Electron crash dumps: `app.getPath('crashDumps')` and a Crash Reporting tutorial exist — [Electron app API](https://github.com/electron/electron/blob/main/docs/api/app.md)

### Inferences
- Conflict to resolve: the distribute page lists Flatpak while issue #3619 looks open; likely means docs guidance (manual flatpak-builder) rather than first-party bundler output. Not verified.

### Gaps
- Not verified: Electron auto-update (Squirrel/electron-updater, no Linux built-in), electron-builder/Forge target matrix, MSIX status in Electron, Windows ARM64 and Linux support matrices for both, Tauri crash reporting (likely community/Sentry), debugging experience.

## Security models

### Takeaway
Not researched successfully; only Tauri's Cargo feature flags (`isolation`, `dynamic-acl`) and sidecar capability scoping were verified.

### Cited Findings
- Tauri core has `isolation` and `dynamic-acl` features; shell/sidecar execution is gated by capability entries with argument validators — [tauri Cargo.toml](https://github.com/tauri-apps/tauri/blob/dev/crates/tauri/Cargo.toml); [Tauri sidecar docs](https://github.com/tauri-apps/tauri-docs/blob/v2/src/content/docs/develop/sidecar.mdx)
- Electron utilityProcess docs say nothing about sandboxing the child — [Electron utilityProcess](https://github.com/electron/electron/blob/main/docs/api/utility-process.md)

### Inferences
- A file manager needs broad filesystem scope in both; in Tauri that means wide fs-plugin scopes/custom commands, in Electron it means careful preload/IPC design (neither compared with sources).

### Gaps
- Electron context isolation/sandbox/fuses and Tauri capabilities/permissions/CSP details not fetched.

## Mobile support and testing

### Takeaway
Tauri WebDriver testing does not cover macOS (tauri-driver issue #7068 open, "help wanted"); Tauri 2 targets Android and iOS but mobile relevance to a desktop file manager is minimal.

### Cited Findings
- Tauri's WebDriver guide limits desktop support to Windows and Linux because macOS has no WKWebView driver; Linux uses WebKitWebDriver, Windows uses Edge Driver — [Tauri WebDriver docs via search summary](https://v2.tauri.app/develop/tests/webdriver/)
- Issue #7068 "MacOSX Support for tauri-driver": open, labels help wanted / platform: macOS, no assignee or PR, opened May 2023 — [tauri #7068](https://github.com/tauri-apps/tauri/issues/7068)
- WebdriverIO documents alternatives: an "embedded" provider (needs tauri-plugin-wdio-webdriver in the app, all three OSes) and a "crabnebula" provider (requires a paid API key) — [WebdriverIO Tauri platform support](https://webdriver.io/docs/desktop-testing/tauri/platform-support)
- Tauri distribute docs list Google Play and App Store as targets — [tauri-docs distribute](https://github.com/tauri-apps/tauri-docs/blob/v2/src/content/docs/distribute/index.mdx)

### Inferences
- Tauri E2E on macOS needs the embedded/CrabNebula route; Electron's Chromium is drivable by Playwright (not verified here).

### Gaps
- Playwright's Electron support (`_electron`) and its status not verified; Tauri mobile maturity not assessed.
