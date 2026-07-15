# Electron Security Checklist
 
Full audit checklist. Walk this against any existing codebase before adding new features, and apply it by
default to any new code.
 
## 1. Window creation defaults
 
```typescript
import { BrowserWindow, app } from 'electron';
import path from 'node:path';
 
function createWindow(): BrowserWindow {
  const win = new BrowserWindow({
    show: false, // paired with 'ready-to-show' below — avoids white flash AND lets CSP/handlers attach first
    webPreferences: {
      contextIsolation: true,
      sandbox: true,
      nodeIntegration: false,
      webSecurity: true,
      preload: path.join(__dirname, '../preload/index.js'),
    },
  });
 
  win.once('ready-to-show', () => win.show());
  return win;
}
```
 
**Verify, don't assume**: `contextIsolation` and `sandbox` default to `true` in modern Electron, but always
set them explicitly. A codebase that has ever set them to `false` "for debugging" and forgotten to revert is
the single most common Electron vulnerability in the wild.
 
## 2. CSP via response header, not `<meta>`
 
```typescript
import { session } from 'electron';
 
session.defaultSession.webRequest.onHeadersReceived((details, callback) => {
  callback({
    responseHeaders: {
      ...details.responseHeaders,
      'Content-Security-Policy': [
        "default-src 'self'; " +
        "script-src 'self'; " +          // no 'unsafe-inline', no 'unsafe-eval'
        "style-src 'self' 'unsafe-inline'; " + // relax only if you truly need inline styles
        "img-src 'self' data:; " +
        "connect-src 'self'; " +
        "object-src 'none'; " +
        "base-uri 'none';"
      ],
    },
  });
});
```
 
Why a header and not a `<meta http-equiv="Content-Security-Policy">` tag: a `<meta>` CSP only takes effect
after the parser reaches it, so any script above that point in the HTML — including one injected via a
markup-injection bug — runs unrestricted. The header applies to the whole response before any HTML parses.
 
## 3. Navigation and window-creation lockdown
 
```typescript
import { shell } from 'electron';
 
const ALLOWED_NAVIGATION_ORIGINS = new Set(['https://your-app-domain.com']);
 
win.webContents.on('will-navigate', (event, url) => {
  const { origin } = new URL(url);
  if (!ALLOWED_NAVIGATION_ORIGINS.has(origin)) {
    event.preventDefault();
  }
});
 
win.webContents.setWindowOpenHandler(({ url }) => {
  const { origin } = new URL(url);
  if (ALLOWED_NAVIGATION_ORIGINS.has(origin)) {
    // still don't let renderer-controlled navigation spawn a full Electron window —
    // hand it to the OS default browser instead
    shell.openExternal(url);
  }
  return { action: 'deny' };
});
```
 
**Never** return `{ action: 'allow' }` from `setWindowOpenHandler` for arbitrary URLs — that creates a new
`BrowserWindow` with (by default) the same webPreferences as the parent, potentially handing a remote page
capabilities it should never have.
 
## 4. contextBridge — the only door between renderer and main
 
```typescript
// preload/index.ts
import { contextBridge, ipcRenderer } from 'electron';
 
contextBridge.exposeInMainWorld('api', {
  getUsers: (request: unknown) => ipcRenderer.invoke('db:get-users', request),
  // one named function per capability — never expose ipcRenderer itself
});
```
 
Anti-pattern to flag immediately if seen:
 
```typescript
// NEVER DO THIS — hands the renderer the entire ipcRenderer object,
// which means it can invoke ANY channel, not just the ones you intended.
contextBridge.exposeInMainWorld('electron', { ipcRenderer });
```
 
## 5. Runtime-validate every IPC handler, both directions
 
```typescript
// main/ipc/users.ts
import { ipcMain } from 'electron';
import { GetUsersRequestSchema, GetUsersResponseSchema } from '../../shared/ipc-contract';
import { queryUsers } from '../db';
 
ipcMain.handle('db:get-users', async (_event, rawRequest) => {
  const request = GetUsersRequestSchema.parse(rawRequest); // throws on bad input
  const users = await queryUsers(request);
  return GetUsersResponseSchema.parse({ users }); // catches DB-layer bugs before they reach the renderer
});
```
 
If `parse` throws, let it — `ipcMain.handle` will reject the renderer's promise, which is the correct
behavior for malformed input. Don't swallow the error and return a default value; that hides bugs and
potential probing attempts.
 
## 6. Things to grep for in an audit
 
| Pattern | Why it's a problem |
|---|---|
| `nodeIntegration: true` | Renderer gets `require`, `process`, `fs` — full Node access from web content |
| `contextIsolation: false` | Preload's JS context merges with the page's — a page script can reach into preload internals |
| `require('@electron/remote')` or `require('electron').remote` | Collapses the process boundary; deprecated for this exact reason |
| `webSecurity: false` | Disables same-origin policy and other web security checks |
| `contextBridge.exposeInMainWorld('x', ipcRenderer)` or similar wholesale exposure | Renderer can call any IPC channel, not just intended ones |
| `shell.openExternal(userControlledUrl)` without a scheme/origin check | Can be used to launch arbitrary protocol handlers |
| `eval(...)` / `new Function(...)` on any renderer- or network-derived string | Arbitrary code execution in whichever process runs it |
| Auto-updater feed URL using `http://` | Update payloads can be MITM'd and swapped for malicious binaries |
| Missing `app.commandLine.appendSwitch('disable-features', 'SitePerProcess')` | Allows cross-origin frames to access parent's Node APIs when site isolation is otherwise enabled | 
