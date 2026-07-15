---
name: electron-development
description: High-security and performance-optimized Electron development workflows. Use whenever building, reviewing, extending, or auditing an Electron desktop app — main process, renderer, preload scripts, contextBridge, IPC (ipcMain/ipcRenderer), BrowserWindow config, or packaging with electron-builder/electron-forge. Trigger on any mention of Electron, a desktop app built with Electron, contextIsolation, sandbox mode, preload scripts, IPC bridges, or requests to scaffold/secure/optimize an Electron + TypeScript/React/Vite app. Also trigger when auditing Electron code for security holes (nodeIntegration enabled, disabled contextIsolation, the remote module, missing CSP, unrestricted navigation/window.open) or performance issues (blocking the main process, oversized IPC payloads, listener/memory leaks). Apply the security and performance defaults automatically even if not explicitly requested — they are the non-negotiable baseline, and violating code should be flagged and corrected, not silently written.
---
 
# Electron Development
 
Zero-trust architecture for Electron: the renderer is treated as untrusted (it runs web content and can be
compromised by a malicious dependency, XSS, or a supply-chain attack), so every capability it has must be
deliberately, narrowly, and validatably granted by the main process. Nothing is available "by default."
 
## Workflow
 
1. **Determine scope**: new app scaffold, a specific feature (e.g. "add a settings IPC channel"), or a
   security/performance audit of existing code.
2. **Apply the non-negotiable defaults below** to every window and every IPC channel touched, regardless of
   what was explicitly asked for.
3. **For audits**: walk the red-flag checklist below against the existing code before writing anything new.
4. **Pull in the relevant reference file** (table below) for the deep-dive pattern needed — don't try to hold
   all of it in working memory, the reference files have the full runnable code.
5. **Lead with a short explanation of the security/performance rationale before showing code** — one or two
   sentences on *why* a given mitigation matters, not a lecture.
## Non-negotiable security defaults
 
Every `BrowserWindow` this skill creates or touches uses:
 
```typescript
new BrowserWindow({
  webPreferences: {
    contextIsolation: true,   // renderer JS context is separate from preload/Node context
    sandbox: true,            // OS-level sandboxing of the renderer process
    nodeIntegration: false,   // no require(), process, fs, etc. in the renderer
    webSecurity: true,        // never disable, including "just for dev"
    preload: path.join(__dirname, '../preload/index.js'),
  },
});
```
 
Alongside that:
 
- **No raw IPC or Node modules ever cross `contextBridge.exposeInMainWorld`.** Expose narrow, named,
  single-purpose functions only (e.g. `getUsers(filters)`, never `ipcRenderer`, never `require`).
- **Every IPC handler validates both the incoming request and its own response at runtime** (Zod or
  equivalent) — TypeScript types are erased at compile time and enforce nothing once the app is running.
- **Navigation and window creation are locked down**: `will-navigate` and `setWindowOpenHandler` deny
  anything not on an explicit allow-list; external links go through `shell.openExternal` only after the URL
  is validated against that allow-list.
- **CSP is set as an HTTP response header** via `session.defaultSession.webRequest.onHeadersReceived`, not a
  `<meta>` tag — a header can't be stripped by injected HTML and applies before any script runs.
- **The `remote` module is never used** (it's removed from modern Electron for good reason — it collapses
  the process boundary). Use `ipcMain.handle` / `ipcRenderer.invoke` instead.
- **Main process code is never handed a raw file blob to `eval`, `new Function()`, or otherwise dynamically
  execute** based on renderer-supplied content.
See `references/security-checklist.md` for the full audit checklist with runnable snippets for each item.
 
## Non-negotiable performance defaults
 
- **Main process stays lightweight**: no CPU-heavy synchronous work, no blocking the event loop, no large
  in-memory caches. Offload heavy work to a `worker_threads` worker or a separate utility process.
- **Windows are created lazily**, not all at app launch; `show: false` + `ready-to-show` to avoid a white
  flash.
- **IPC payloads stay small and flat.** Pass file paths or IDs across IPC, not raw binary/file contents;
  stream large data instead of round-tripping it through `ipcRenderer.invoke`.
- **Single source of truth for state** — don't let main and renderer each maintain their own copy of the
  same state that can drift.
- **Every `useEffect`/event listener registered against `ipcRenderer.on` in React is cleaned up** on
  unmount via the returned unsubscribe (see `references/ipc-patterns.md` for the pattern) — otherwise every
  hot-reload or remount stacks another listener.
See `references/performance-checklist.md` for main-process hygiene patterns and common leak sources.
 
## Reference files — pull in as needed
 
| File | When to read it |
|---|---|
| `references/security-checklist.md` | Full security audit checklist: CSP header setup, navigation lockdown, sandbox verification, red-flag anti-patterns with fixes |
| `references/performance-checklist.md` | Main-process hygiene, worker_threads offloading, memory leak patterns, startup optimization |
| `references/ipc-patterns.md` | The shared-contract + Zod pattern for type-safe IPC, invoke/handle vs send/on, streaming large payloads, React listener cleanup |
| `references/project-scaffold.md` | Full electron-vite + TypeScript + React starter: file tree, `electron.vite.config.ts`, `package.json`, `tsconfig` |
| `references/packaging-and-updates.md` | electron-builder config, code signing requirements per OS, `electron-updater` security notes (signature verification, HTTPS-only feeds) |
 
## Red flags — refuse or warn, don't silently comply
 
If a request asks for any of the following, say so explicitly, explain the concrete risk, and propose the
secure alternative instead of writing the insecure version:
 
- `nodeIntegration: true` or `contextIsolation: false`
- Using the `remote` module
- Exposing `ipcRenderer` (or any Node module) wholesale via `contextBridge`
- Disabling `webSecurity`, even "temporarily" or "just for local dev"
- `setWindowOpenHandler` that allows arbitrary URLs, or no navigation guard at all
- Skipping runtime validation on IPC because "the renderer is trusted, it's our own code" — the renderer is
  never trusted in this model, since it's the thing most exposed to remote content/dependencies
- `eval`, `new Function()`, or `child_process.exec` fed by unvalidated renderer input
- Auto-update feeds served over plain HTTP, or without signature verification
## Tech stack defaults (used unless the user specifies otherwise)
 
- **Bundler**: `electron-vite` (Vite for main, preload, and renderer with HMR)
- **Language**: TypeScript in all three processes
- **Frontend**: React (TSX)
- **Validation**: Zod for IPC contracts
- **Packaging**: `electron-builder`