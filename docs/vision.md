# Transistor

**TypeScript engine, native I/O, every platform.**

## The Problem

There is no framework that lets you write application logic in TypeScript and ship it as a native app on desktop, Android, and iOS — with real native bindings for sockets and filesystem.

The existing landscape:

| Framework | Desktop | Android | iOS | JS Engine | Raw Sockets | Real Filesystem |
|-----------|---------|---------|-----|-----------|-------------|-----------------|
| **Electron** | Yes | No | No | Node/V8 | Yes | Yes |
| **Tauri** | Yes | Experimental | Experimental | Rust backend | Yes (Rust) | Yes (Rust) |
| **Electrobun** | Yes | No | No | Bun | Yes | Yes |
| **Capacitor** | No | Yes | Yes | Webview | No | Plugin only |
| **React Native** | Partial | Yes | Yes | Hermes/JSC | No | Third-party |

Nobody covers all platforms with a real JS runtime and native I/O primitives. Capacitor is closest in spirit (web code + native shell) but it's webview-only with no real system access. Electrobun has the right architecture (JS engine + native FFI) but is desktop-only. Tauri requires Rust for the backend logic.

## The Insight

The native binding surface needed is surprisingly small. Web APIs already cover rendering, layout, input, audio, video, crypto, workers, storage. The two things the web platform can't do are:

1. **Raw sockets** (TCP/UDP)
2. **Real filesystem access**

That's it. Every "native app" use case that the web can't handle comes down to sockets, files, or both:

- Torrent client → sockets + files
- Web server → sockets + files
- FTP client → sockets + files
- Ad-blocking browser → sockets
- Database app → files
- Dev tools → sockets + files

The binding layer is maybe 10-15 functions total per platform.

## The Architecture

Different JS runtimes on each platform, chosen to be the natural fit:

| Platform | JS Runtime | Native Bindings | Why |
|----------|-----------|-----------------|-----|
| **Desktop** (Mac/Win/Linux) | Bun | FFI via dlopen | Full runtime, Electrobun-style packaging |
| **Android** | QuickJS | JNI to Kotlin | Tiny, embeds in-process, works in foreground services |
| **iOS** | JavaScriptCore | System framework | Zero bundling cost, Apple's own engine |

The key property: the JS runtime runs **in-process**, not as a subprocess. On Android this is essential — the OS lifecycle (foreground services, battery optimization) requires the engine to live inside the app process. QuickJS linked via JNI is the only practical approach. JSC on iOS is similar — it's a system library you call from Swift.

One TypeScript API surface across all three runtimes. App developers write once, don't care which runtime they're on.

## Prior Art & Proof Points

### JSTorrent (jstorrent.com)

This architecture has been proven in production. JSTorrent ships a TypeScript engine on every platform:

- Desktop: Tauri webview + Rust io-daemon (sidecar)
- Android: QuickJS + Kotlin JNI bindings
- ChromeOS: Chrome extension + Android companion server
- CLI: Node.js

The engine implements `IFileSystem` / `IFileHandle` interfaces with 6 backend adapters (Node fs, HTTP-to-daemon, JNI-to-Kotlin, in-memory, null). Same TypeScript code runs everywhere. This proves:

- QuickJS embedded in Android works reliably (foreground service, background downloads)
- A unified I/O abstraction can paper over radically different native backends
- The binding surface (files + sockets) is sufficient for a complex real-world app

### Electrobun (electrobun.dev)

Desktop framework: Bun runtime + native C++/ObjC++ wrappers loaded via FFI. Key ideas worth adopting:

- `dlopen()` for zero-overhead native calls from JS (no IPC)
- bsdiff patching for tiny (~14KB) application updates
- System webview for UI (no bundled Chromium)
- Single ~12MB distributable

Desktop-only. No mobile story. But the right architecture for the desktop leg.

### Demand Signal

People keep asking for "Electrobun on mobile" and nobody has an answer:

- [Bun mobile compilation request](https://github.com/oven-sh/bun/issues/21237) — someone explicitly asks for Electrobun on mobile, and there's a working iOS fork of Bun as of Feb 2026
- [HN discussion](https://news.ycombinator.com/item?id=47069650) — "Can Electrobun compile to Android?" — user wants to build an ad-blocking browser, ends up on Ionic which can't do it
- Capacitor users routinely hit the wall of "I need raw sockets / real filesystem" and have no answer

## MVP Scope

### First app: Web server

A simple web server is the perfect proving ground:

- Exercises the exact primitives (sockets + files)
- Natural fit for TypeScript (the entire history of simple web servers is JS)
- Useful standalone product
- Simple enough to ship quickly

### Binding API (MVP)

Minimal surface — files and sockets only:

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

~15 functions, implemented three times (Bun FFI, Kotlin JNI, Swift JSC).

### Scaffolding

CLI tool that generates platform projects from templates:

```
transistor init my-app --platforms android,ios,desktop
```

Each platform template is a pre-built project (created once by hand in Android Studio / Xcode) with the JS runtime embedded, bindings wired up, and a webview ready. The CLI copies the template and does string replacement for app name / bundle ID.

No programmatic Gradle/Xcode project generation. Just good templates. Same approach as Capacitor (`npx cap add android`).

### Non-goals for MVP

- Comprehensive native API surface (camera, bluetooth, sensors, etc.)
- Plugin system
- Hot reload across platforms
- Framework upgrade mechanism (template regeneration)

## Name

**Transistor** — three terminals (desktop, Android, iOS), amplifies what JavaScript can do, electronics lineage with Capacitor/Electron. Short, brandable, easy to search.
