# Transistor

**TypeScript engine, native I/O, every platform.**

## The Problem

There is no single framework that cleanly lets you write application logic in TypeScript and ship it as a native app on desktop, Android, and iOS with all of the following at once:

- Native socket and filesystem access from a JS runtime
- A straightforward UI/windowing story
- A practical mobile lifecycle/background story
- A coherent distribution and update workflow

The existing landscape:

| Framework | Desktop | Android | iOS | JS Engine | Raw Sockets | Real Filesystem |
|-----------|---------|---------|-----|-----------|-------------|-----------------|
| **Electron** | Yes | No | No | Node/V8 | Yes | Yes |
| **Tauri** | Yes | Yes (v2) | Yes (v2) | Rust backend + WebView | Yes (Rust) | Yes (Rust) |
| **Electrobun** | Yes | No | No | Bun | Yes | Yes |
| **Capacitor** | No | Yes | Yes | WebView | No | Plugin only |
| **React Native** | Partial | Yes | Yes | Hermes/JSC | No | Third-party |
| **Bare** | Yes | Yes | Yes | V8 via libjs | Via modules | Via modules |

Most frameworks still force a major tradeoff. Capacitor is closest in spirit (web code + native shell) but stays webview-centric. Electrobun has the right desktop architecture (JS runtime + native FFI) but no mobile story. Tauri has broad platform support, but backend/runtime logic is Rust-first.

Bare is the notable runtime exception for cross-platform embedding and native I/O primitives, but it intentionally does not solve UI/windowing, picker UX, app lifecycle policy wiring, or product-level update UX by itself.

## The Insight

The native binding surface for a useful MVP is surprisingly small. Web APIs already cover rendering, layout, input, audio, video, crypto, workers, and storage. There is still a long tail of native capabilities the web cannot reliably handle (GPU compute, USB/Bluetooth, sensors, strict background execution), but two common gaps are:

1. **Raw sockets** (TCP/UDP)
2. **Real filesystem access**

A large class of "native app" use cases comes down to sockets, files, or both:

- Torrent client -> sockets + files
- Web server -> sockets + files
- FTP client -> sockets + files
- Ad-blocking browser -> sockets
- Database app -> files
- Dev tools -> sockets + files

The binding layer for MVP can still be small (roughly 10-15 core functions per platform).

## The Architecture

Different JS runtimes on each platform, chosen as a natural fit:

| Platform | JS Runtime | Native Bindings | Why |
|----------|-----------|-----------------|-----|
| **Desktop** (Mac/Win/Linux) | Bun | FFI via `dlopen` | Full runtime, Electrobun-style packaging |
| **Android** | QuickJS | JNI to Kotlin | Tiny, in-process embedding, practical for services |
| **iOS** | JavaScriptCore | System framework | Zero extra engine bundle cost |

The key property is in-process runtime execution, not subprocess IPC. On Android this matters for service lifecycle and power policy integration. QuickJS via JNI is one practical approach; embedded V8 (for example via Bare) is another.

One TypeScript API surface across runtimes: app code should not care which engine is underneath.

## Prior Art & Proof Points

### JSTorrent (jstorrent.com)

This architecture is proven in production. JSTorrent ships a TypeScript engine on multiple targets:

- Desktop: Tauri webview + Rust io-daemon sidecar
- Android: QuickJS + Kotlin JNI bindings
- ChromeOS: Chrome extension + Android companion server
- CLI: Node.js

The engine implements `IFileSystem` / `IFileHandle` interfaces with multiple backend adapters (Node fs, HTTP-to-daemon, JNI-to-Kotlin, in-memory, null). Same TypeScript code runs everywhere. This proves:

- QuickJS embedded in Android can work reliably (foreground service, background downloads)
- A unified I/O abstraction can cover radically different native backends
- Files + sockets can be enough for a complex real-world app

### Electrobun (electrobun.dev)

Desktop framework: Bun runtime + native C++/ObjC++ wrappers loaded via FFI. Strong ideas worth adopting:

- `dlopen()` for direct native calls from JS (no mandatory IPC hop)
- bsdiff patching for small application updates
- System webview for UI (no bundled Chromium)
- Small distributables compared to Electron-class bundles

Desktop-only today, but architecturally strong for that leg.

### Bare - What It Actually Solves

Bare is a small runtime for embedding JS across desktop and mobile. It is close to the runtime core Transistor describes, but with important scope boundaries.

What Bare does provide:

- Cross-platform runtime support (desktop, Android, iOS)
- In-process embedding APIs for host applications
- Runtime lifecycle primitives (`suspend` / `wakeup` / `resume`)
- Ecosystem modules for native I/O (`bare-fs`, `bare-tcp`, `bare-dgram`)

What Bare does not provide by itself:

- UI framework or windowing model
- File/folder picker UX flows
- Android foreground service implementation and notification wiring
- iOS background entitlement policy handling
- End-to-end desktop/mobile distribution and product update UX

So Bare can significantly reduce runtime + I/O plumbing, but host-platform integration work remains substantial for a product app.

### Bare vs Multi-Runtime Tradeoff

| | Bare (V8 everywhere) | Multi-runtime (Bun + QuickJS + JSC) |
|---|---|---|
| **Engine behavior** | More uniform | Per-engine differences |
| **Binary footprint** | Larger on mobile | Potentially smaller on mobile |
| **Maintenance** | One runtime family | Three runtime families |
| **Integration flexibility** | Strong runtime core | Can optimize per platform |
| **Host app work** | Still required | Still required |

### Demand Signal

People keep asking for "Electrobun on mobile" and still hit the same gap:

- [Bun mobile compilation request](https://github.com/oven-sh/bun/issues/21237)
- [HN discussion](https://news.ycombinator.com/item?id=47069650) asking about Electrobun on Android
- Capacitor users repeatedly hitting raw socket/filesystem limits

## MVP Scope

### First app: Web server

A simple web server is a good proving ground:

- Exercises sockets + files directly
- Natural fit for TypeScript
- Useful standalone artifact
- Small enough to iterate quickly

### Binding API (MVP)

Minimal surface: files and sockets.

**Filesystem:**
- open / close
- read / write
- stat
- mkdir
- readdir
- unlink

**Sockets:**
- TCP: listen, accept, connect, send, recv, close
- UDP: bind, send, recv, close

Roughly 15 core functions if hand-implemented per platform, or mapped onto existing runtime ecosystems where available.

### Scaffolding

CLI tool that generates platform projects from templates:

```sh
transistor init my-app --platforms android,ios,desktop
```

Each platform template is a prebuilt project (Android Studio / Xcode / desktop host) with runtime embedding, native bindings, and a starter UI path wired. The CLI copies templates and rewrites app identifiers.

No dynamic Gradle/Xcode project generation required for MVP.

### Non-goals for MVP

- Broad native API coverage (camera, bluetooth, sensors, etc.)
- Full plugin marketplace
- Universal hot reload across all platforms
- Automatic template regeneration across major framework upgrades

## Name

**Transistor** - three terminals (desktop, Android, iOS), amplifies what JavaScript can do, electronics lineage with Capacitor/Electron.
