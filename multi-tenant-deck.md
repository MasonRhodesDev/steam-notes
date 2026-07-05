# Multi-tenant Deck: per-Steam-account app profiles

All Steam profiles on a Deck share one Linux user (`deck`), so apps like Discord
share state by default. Pattern that works:

1. **Detect the active Steam account at launch** — parse
   `~/.steam/steam/config/loginusers.vdf` and take the block with
   `"MostRecent" "1"`; account id = steamid64 − 76561197960265728.
   (This client build does **not** write `ActiveUser` to `~/.steam/registry.vdf`;
   `AutoLoginUser` there is a login *name*, not an id. MostRecent flips on every
   profile login, so it's correct at shortcut-launch time.)
2. **Symlink-swap the app's config dir** to a per-account profile dir in a
   launcher wrapper, then `exec` the app. Keep profile dirs *inside*
   `~/.var/app/<app-id>/` for flatpaks (see flatpak-pitfalls.md).
3. Point every profile's Steam shortcut at the same wrapper
   (`~/.local/bin/game-mode-discord` on this device).

Notes:
- Steam shortcuts + grid artwork are per-account (`userdata/<accountid>/config/`),
  so each profile needs its own `shortcuts.vdf` entries; identical wrapper exes
  give identical crc appids across profiles (fine — grids are per-account dirs).
- Same pattern extends to Chromium (`--user-data-dir`), Kodi (built-in profiles
  or `KODI_DATA`), Prism (has native multi-account).
- The heavyweight alternative — one Linux user per person with their own Gaming
  Mode session — is viable on current builds; internals in
  [session-switching.md](session-switching.md), design work in the private
  `steamos-multiuser` repo.
