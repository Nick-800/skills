# Electron Performance Checklist
 
## 1. Keep the main process light
 
The main process runs your app's single UI-adjacent event loop for window management, menus, and IPC
dispatch. Blocking it stalls window resizing, menu clicks, and every pending IPC call across all windows.
 
```typescript
// BAD — synchronous heavy work blocks the whole main process
ipcMain.handle('process-image', (_e, buffer) => {
  return heavySyncImageProcessing(buffer); // blocks main for the whole duration
});
 
// GOOD — offload to a worker thread, main process just dispatches
import { Worker } from 'node:worker_threads';
 
ipcMain.handle('process-image', (_e, buffer) => {
  return new Promise((resolve, reject) => {
    const worker = new Worker(path.join(__dirname, 'image-worker.js'));
    worker.postMessage(buffer, [buffer]); // transfer, don't copy, for large ArrayBuffers
    worker.once('message', (result) => { resolve(result); worker.terminate(); });
    worker.once('error', (err) => { reject(err); worker.terminate(); });
  });
});
```
 
For CPU-bound work that needs to stay a genuinely separate OS process (e.g. isolating a crash), spawn a
`utilityProcess` (Electron's replacement for ad-hoc `child_process` use) instead of doing it inline.
 
## 2. Lazy window creation + fast first paint
 
```typescript
const win = new BrowserWindow({ show: false, /* ...webPreferences */ });
win.once('ready-to-show', () => win.show()); // avoids white flash, and shows only once content is ready
```
 
Don't create secondary windows (settings, about, etc.) at app launch — create them on first use and reuse
the existing instance rather than creating a new `BrowserWindow` every time the user reopens them:
 
```typescript
let settingsWindow: BrowserWindow | null = null;
 
function openSettingsWindow() {
  if (settingsWindow) {
    settingsWindow.focus();
    return;
  }
  settingsWindow = createSettingsWindow();
  settingsWindow.on('closed', () => { settingsWindow = null; });
}
```
 
## 3. IPC payload hygiene
 
- Pass file **paths**, not file **contents**, across IPC. Let the renderer request a stream or a chunked
  read if it needs the bytes, rather than serializing megabytes through `ipcRenderer.invoke`.
- Keep payload shapes flat — deeply nested objects cost more to structured-clone across the IPC boundary.
- For genuinely large or continuous data (e.g. log tailing, progress updates), use `webContents.send` for
  one-way push notifications rather than polling with repeated `invoke` calls.
## 4. React-side listener cleanup
 
Every `ipcRenderer.on` registered from a preload-exposed subscribe function needs a matching `removeListener`
on unmount, or every hot-reload/remount during development (and every mount/unmount in production) stacks
another duplicate listener.
 
```typescript
// preload/index.ts — expose a subscribe function that returns its own unsubscribe
contextBridge.exposeInMainWorld('api', {
  onProgress: (callback: (pct: number) => void) => {
    const listener = (_event: unknown, pct: number) => callback(pct);
    ipcRenderer.on('task:progress', listener);
    return () => ipcRenderer.removeListener('task:progress', listener);
  },
});
```
 
```tsx
// renderer/src/ProgressBar.tsx
useEffect(() => {
  const unsubscribe = window.api.onProgress(setPercent);
  return () => unsubscribe(); // critical — without this, listeners accumulate across remounts
}, []);
```
 
## 5. Single source of truth for state
 
Don't let the main process cache a copy of state (e.g. "current user settings") that can drift from what's
persisted on disk or what the renderer displays. Read from the persistence layer (file, DB) on demand or
push explicit invalidation events — don't maintain a shadow copy in main-process memory that has to be kept
in sync by hand.
 
## 6. Startup profiling
 
Use `app.getAppMetrics()` and Chromium's built-in `--trace-startup` flag to profile cold start if startup
time becomes a concern; the two most common culprits are large synchronous `require()` chains in the main
process entry point and eagerly creating windows/webviews that aren't needed yet.