# Driving the Steam client via CEF debugging

- Launch with `steam -cef-enable-debugging` → DevTools protocol on
  `http://localhost:8080/json`. The target titled **SharedJSContext** hosts the
  full client JS API. Evaluate with `Runtime.evaluate`
  (`awaitPromise: true, returnByValue: true`); wrap awaits in an async IIFE.
  A ~50-line stdlib-Python WebSocket client is enough (no TLS on localhost).
- Never leave the flag on for daily use — anything local can drive the client.
  The client setting `cef_remote_debugging_enabled` also exists.

## Useful surfaces (2026 client)
- `SteamClient.Apps`: `AddShortcut`, `RemoveShortcut`, `SetShortcutExe/Name/
  Icon/StartDir/LaunchOptions`, `RegisterForAppDetails(appid, cb)` (one-shot
  details incl. `strShortcutExe`), `InstallFlatpakAppAndCreateShortcut`,
  `ListFlatpakApps`, `SetCustomArtworkForApp`.
- `appStore`: `allApps`, `GetAppOverviewByAppID(id)`; overview has `appid`,
  `display_name`, `BIsShortcut()`, `per_client_data` (clientid/installed),
  `selected_clientid`. `app_type == 1073741824` = shortcut.
- `collectionStore`: `GetCollection('type-games')`, `.visibleApps`,
  `BIsHidden(appid)`, `SetAppsAsHidden(appids, bool)` (see remote-shortcuts.md
  for the shortcut-vs-store storage split).
- `SteamClient.RemotePlay`: `SetRemotePlayEnabled(bool)`, pairing/unpairing.
- `settingsStore.m_ClientSettings` — flat key dump is the fastest way to hunt
  for a setting (264 keys on this build).
- Steam only flushes `shortcuts.vdf`/`localconfig.vdf` on clean exit
  (`steam -shutdown`). Edit those files only while Steam is fully down, or your
  changes are overwritten.
