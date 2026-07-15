
# Type-Safe IPC Patterns
 
## The shared-contract pattern
 
Define the channel name and both request/response shapes in one file imported by both main and preload/
renderer, so they can never silently drift apart.
 
```typescript
// src/shared/ipc-contract.ts
import { z } from 'zod';
 
export const IPC_CHANNELS = {
  GET_USERS: 'db:get-users',
} as const;
 
export const GetUsersRequestSchema = z.object({
  search: z.string().max(100).optional(),
  limit: z.number().int().positive().max(100).default(20),
});
export type GetUsersRequest = z.infer<typeof GetUsersRequestSchema>;
 
export const UserRecordSchema = z.object({
  id: z.number().int(),
  name: z.string(),
  email: z.string().email(),
  createdAt: z.string(), // ISO string — Date objects don't survive structured clone cleanly
});
 
export const GetUsersResponseSchema = z.object({
  users: z.array(UserRecordSchema),
});
export type GetUsersResponse = z.infer<typeof GetUsersResponseSchema>;
```
 
## `invoke`/`handle` vs `send`/`on`
 
- **`ipcRenderer.invoke` + `ipcMain.handle`**: use for request/response — renderer asks for something and
  needs a return value (a query, a file save result, a settings read). This is the default choice.
- **`webContents.send` + `ipcRenderer.on`**: use for one-way push from main to renderer — progress updates,
  streamed log lines, "a background job finished" notifications. Don't try to fake a request/response cycle
  with two `send` calls in each direction; `invoke`/`handle` already does that correctly, including error
  propagation.
## Full request/response flow
 
```typescript
// main/index.ts — register the handler
import { ipcMain } from 'electron';
import { GetUsersRequestSchema, GetUsersResponseSchema, IPC_CHANNELS } from '../shared/ipc-contract';
import { queryUsers } from './db';
 
ipcMain.handle(IPC_CHANNELS.GET_USERS, async (_event, rawRequest) => {
  const request = GetUsersRequestSchema.parse(rawRequest);
  const users = await queryUsers(request);
  return GetUsersResponseSchema.parse({ users });
});
```
 
```typescript
// preload/index.ts — the only bridge
import { contextBridge, ipcRenderer } from 'electron';
import { IPC_CHANNELS, type GetUsersRequest, type GetUsersResponse } from '../shared/ipc-contract';
 
contextBridge.exposeInMainWorld('api', {
  getUsers: (request: GetUsersRequest): Promise<GetUsersResponse> =>
    ipcRenderer.invoke(IPC_CHANNELS.GET_USERS, request),
});
 
// renderer/src/types/window.d.ts — tell TypeScript what window.api looks like
declare global {
  interface Window {
    api: {
      getUsers: (request: GetUsersRequest) => Promise<GetUsersResponse>;
    };
  }
}
```
 
```tsx
// renderer/src/App.tsx
import { useEffect, useState } from 'react';
import type { UserRecord } from '../../shared/ipc-contract';
 
export default function App() {
  const [users, setUsers] = useState<UserRecord[]>([]);
  const [error, setError] = useState<string | null>(null);
 
  useEffect(() => {
    let cancelled = false;
    window.api.getUsers({ limit: 20 })
      .then((res) => { if (!cancelled) setUsers(res.users); })
      .catch((err) => { if (!cancelled) setError(String(err)); });
    return () => { cancelled = true; }; // avoid setting state after unmount
  }, []);
 
  if (error) return <p role="alert">Failed to load users: {error}</p>;
 
  return (
    <ul>
      {users.map((u) => <li key={u.id}>{u.name} — {u.email}</li>)}
    </ul>
  );
}
```
 
## Push notifications (progress, logs) with cleanup
 
```typescript
// preload/index.ts
contextBridge.exposeInMainWorld('api', {
  onProgress: (callback: (pct: number) => void) => {
    const listener = (_event: unknown, pct: number) => callback(pct);
    ipcRenderer.on('task:progress', listener);
    return () => ipcRenderer.removeListener('task:progress', listener);
  },
});
```
 
```tsx
// renderer — always unsubscribe on unmount
useEffect(() => {
  const unsubscribe = window.api.onProgress(setPercent);
  return unsubscribe;
}, []);
```
 
## Streaming large payloads instead of one big IPC message
 
For large files, don't read the whole file into a buffer and send it over `invoke`. Pass the file path and
let the renderer request chunks, or better, write large-data flows to disk/temp files and hand back a path:
 
```typescript
ipcMain.handle('export:large-report', async (_event, rawRequest) => {
  const request = ExportRequestSchema.parse(rawRequest);
  const outputPath = await writeReportToTempFile(request); // writes to disk, doesn't buffer in memory
  return { path: outputPath }; // small, flat payload
});
```