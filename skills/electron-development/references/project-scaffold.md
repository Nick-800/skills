# electron-vite + TypeScript + React Scaffold
 
## File tree
 
```
my-app/
├── electron.vite.config.ts
├── package.json
├── tsconfig.json
├── tsconfig.node.json
├── tsconfig.web.json
├── src/
│   ├── main/
│   │   ├── index.ts
│   │   └── db.ts
│   ├── preload/
│   │   ├── index.ts
│   │   └── index.d.ts
│   ├── renderer/
│   │   ├── index.html
│   │   └── src/
│   │       ├── main.tsx
│   │       └── App.tsx
│   └── shared/
│       └── ipc-contract.ts
```
 
## `package.json` (relevant excerpt)
 
```json
{
  "name": "my-app",
  "version": "1.0.0",
  "main": "./out/main/index.js",
  "scripts": {
    "dev": "electron-vite dev",
    "build": "electron-vite build",
    "build:win": "electron-vite build && electron-builder --win",
    "build:mac": "electron-vite build && electron-builder --mac",
    "build:linux": "electron-vite build && electron-builder --linux"
  },
  "dependencies": {
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "electron": "^31.0.0",
    "electron-vite": "^2.3.0",
    "electron-builder": "^24.13.0",
    "typescript": "^5.5.0",
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.0"
  }
}
```
 
## `electron.vite.config.ts`
 
```typescript
import { resolve } from 'path';
import { defineConfig, externalizeDepsPlugin } from 'electron-vite';
import react from '@vitejs/plugin-react';
 
export default defineConfig({
  main: {
    plugins: [externalizeDepsPlugin()],
  },
  preload: {
    plugins: [externalizeDepsPlugin()],
  },
  renderer: {
    resolve: {
      alias: {
        '@shared': resolve('src/shared'),
      },
    },
    plugins: [react()],
  },
});
```
 
## `src/renderer/src/main.tsx`
 
```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
 
ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
);
```
 
## `src/preload/index.d.ts`
 
Ambient typing so both preload build and renderer TypeScript agree on `window.api`'s shape without either
side needing to import from the other's build output.
 
```typescript
import type { GetUsersRequest, GetUsersResponse } from '../shared/ipc-contract';
 
declare global {
  interface Window {
    api: {
      getUsers: (request: GetUsersRequest) => Promise<GetUsersResponse>;
    };
  }
}
```
 
## Notes
 
- `externalizeDepsPlugin()` in main/preload configs keeps `node_modules` out of the bundle for those
  processes — they run in Node/Electron directly and don't need bundling the way renderer code does.
- Use path aliases (`@shared`) instead of long relative `../../../shared/...` imports once the shared
  contract file is referenced from three different process trees.
- See `references/security-checklist.md` and `references/ipc-patterns.md` for what actually goes inside
  `src/main/index.ts` and `src/preload/index.ts`.