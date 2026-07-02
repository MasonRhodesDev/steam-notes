# Non-Steam shortcut app IDs

- A shortcut's app ID is `crc32(Exe + AppName) | 0x80000000`, where `Exe` is the
  *quoted* executable string exactly as stored in `shortcuts.vdf`
  (e.g. `"/usr/bin/kodi"` + `Kodi` → `2443196458`). Community docs:
  Steam-Shortcut-Manager wiki.
- `shortcuts.vdf` (binary VDF, `userdata/<accountid>/config/`) stores the appid
  explicitly. Steam **respects a stored appid** as long as it doesn't collide with
  another record the client already knows (including remote/streamable shortcut
  records synced from other devices).
- On collision Steam **re-keys the local entry to a fresh opaque id** — observed
  ids match no crc32 variant of exe/name/startdir; treat them as arbitrary.
  `SteamClient.Apps.AddShortcut` also mints non-crc ids when a collision exists.
- Editing an existing shortcut (`SetShortcutExe`, name, etc.) does **not** re-key it.
- Practical rules:
  - Grid artwork is keyed by appid (`grid/<appid>p.png`, `<appid>_hero.png`,
    `<appid>_logo.png`, `<appid>_icon.png`, wide capsule `<appid>.png`).
  - If you hand-write `shortcuts.vdf`, pick appids that can't collide with any
    synced record for that account, or Steam orphans your artwork on first load.
  - Two devices can never share one shortcut tile by forcing equal appids —
    Steam re-keys the local one (verified experimentally; see remote-shortcuts.md).
