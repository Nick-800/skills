# Packaging and Auto-Updates
 
## electron-builder config (`electron-builder.yml`)
 
```yaml
appId: com.yourcompany.yourapp
productName: Your App
directories:
  output: release
files:
  - out/**/*
  - package.json
mac:
  hardenedRuntime: true       # required for notarization
  gatekeeperAssess: false
  notarize: true
  entitlements: build/entitlements.mac.plist
win:
  target: nsis
  signingHashAlgorithms: [sha256]
linux:
  target: AppImage
publish:
  provider: github            # or your own update server — must be HTTPS
```
 
## Code signing — required per OS, not optional
 
- **macOS**: unsigned/unnotarized apps are quarantined by Gatekeeper and most users can't open them without
  digging into System Settings. `hardenedRuntime: true` + notarization is the baseline for distribution
  outside the Mac App Store.
- **Windows**: unsigned executables trigger SmartScreen warnings ("Windows protected your PC"). A code
  signing certificate (EV or standard) is needed to avoid this for any real distribution.
- **Linux**: no OS-level signing gate, but still sign your AppImage/deb if you publish a repository, so
  users can verify provenance.
## Auto-update security (`electron-updater`)
 
```typescript
import { autoUpdater } from 'electron-updater';
 
autoUpdater.autoDownload = false; // don't silently download and swap binaries — ask the user first
autoUpdater.checkForUpdates();
 
autoUpdater.on('update-available', () => {
  // prompt the user, then autoUpdater.downloadUpdate() on confirmation
});
```
 
Non-negotiable:
 
- **The update feed URL must be HTTPS.** An HTTP feed can be MITM'd to serve a malicious binary as a
  "legitimate" update.
- **electron-updater verifies code-signing signatures on downloaded updates by default** — don't disable
  this verification even if it's inconvenient during development; disable it only for a genuinely separate
  dev-only update channel that never ships.
- Pin the update feed to your own signed release channel (GitHub Releases, S3 bucket you control, or a
  dedicated update server) — never point it at a third-party mirror you don't control.
## Common pitfalls at build time
 
- Forgetting to externalize native modules (`better-sqlite3`, etc.) so they get rebuilt against Electron's
  Node ABI, not the system Node ABI — use `electron-builder`'s `npmRebuild` (on by default) or
  `@electron/rebuild` manually if bypassing electron-builder's install step.
- Shipping devDependencies inside `asar` by not scoping `files` correctly in the builder config — keep the
  `files` allow-list tight (as in the example above) rather than defaulting to "everything."
- Leaving source maps in a public release build if the app contains anything sensitive in comments/strings
  — decide deliberately whether to ship them.