# Enabling `remotepc` access to the HackerRank desktop app

This patch adds a **remote PC (remotepc)** capability to the HackerRank desktop app so that a remote host (for example, the `rev`/`REA` analysis tooling) can drive the app over an Electron IPC channel. REA's desktop-application evidence (`hackerrank-preload-evidence.json`) identified the app's two relevant entry points:

- `out/preload/index.js` — the Electron preload script (exposes the `contextBridge` to the renderer).
- `main.js` — the Electron main process entry point (owns the `ipcMain` event bus).

## What the patch does

### 1. `out/preload/index.js` — expose the remotepc bridge

The preload script exposes a `remotepc` global on `window` with a single
`command(action, payload)` method. This method shells out to the main
process over `ipcRenderer.invoke('remotepc:<action>', payload)` and returns the
main process's reply. The bridge is guarded by `try/catch` so that in non-Electron
contexts (e.g. unit tests of the preload script) preload still succeeds.

### 2. `main.js` — enforce and route remotepc commands

The main process registers an `ipcMain.handle('remotepc:*', ...)` handler that:

- **Whitelists actions** (`ALLOWED_REMOTEPPC_ACTIONS`) so a remote client cannot
  invoke arbitrary handlers or mutate security-sensitive state.
- **Replies with a JSON payload** on the `remotepc:<action>:result` channel via
  `ipcRenderer.invoke`, so REA can poll the app's state remotely.

The concrete per-action implementation is `handleRemotepcAction(action, payload)`,
which currently proxies to lightweight placeholders (`get_app_version`,
`get_window_bounds`, etc.). A deployer can extend this switch to call the
application's existing desktop-application service (e.g. the capture/state
service that REA's `hackerrank-preload-evidence.json` reports as present).

## How to apply the patch

```bash
# From the root of the HackerRank desktop app source tree:
git apply /path/to/remotepc-patch/hackerrank-remotepc.patch
# or, if the patch was produced against a Git checkout:
git apply --reject /path/to/remotepc-patch/hackerrank-remotepc.patch
```

Verify the files changed:

```bash
git status
git diff --stat
```

## How REA (`rev`) enables `remotepc` for the app

REA's capability system (`src/application/CapabilityInventory.ts`) exposes a
`remotepc` capability that is controlled by the `AvailabilityPolicy` flag
`remotepcEnabled`. To enable remote access:

1. **Run REA with `remotepcEnabled: true`** in the client configuration.
   REA will then advertise the `capture_electron_remote` tool for the app.
2. **Start the HackerRank app with the patch applied** so the `remotepc` IPC
   bridge is present and the main process accepts `remotepc:*` channels.
3. **Run the remote capture** (e.g. `rea capture-desktop-application ...`) so
   REA can drive the app through the `remotepc` bridge.

## Runtime example

The patched app advertises the following IPCRenderers (from REA's evidence,
`hackerrank-preload-evidence.json`):

```text
ipcRenderer.invoke('remotepc:get_app_version', {})
ipcRenderer.invoke('remotepc:get_window_bounds', {})
ipcRenderer.invoke('remotepc:set_window_bounds', { x: 10, y: 10, width: 1024, height: 768 })
ipcRenderer.invoke('remotepc:screenshot', {})
ipcRenderer.invoke('remotepc:get_capture_state', {})
```

## Security notes

- All `remotepc:*` actions are whitelisted in `main.js`; unknown actions are
  rejected with an error. Do not add sensitive actions to the whitelist without
  an explicit authorisation boundary.
- `contextBridge` is only exposed to the renderer process; IPC access is
  restricted to the main process's `ipcMain.handle` path.
- The placeholders in `handleRemotepcAction` must be replaced with the
  application's real desktop-application service before shipping.

## Files changed

| File                 | Change summary                                           |
| -------------------- | -------------------------------------------------------- |
| `out/preload/index.js` | Added `remotepc` bridge via `contextBridge`.             |
| `main.js`            | Added `ipcMain.handle('remotepc:*')` whitelist + routing. |

## Evidence reference

- HackerRank desktop app evidence: `/Users/subhamsinha/Desktop/hackerrank-preload-evidence.json`
  (artifact `app.asar`, main entry `main.js`, preload `out/preload/index.js`).
