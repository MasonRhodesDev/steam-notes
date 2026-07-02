# Remote ("Stream from …") shortcut tiles

## What they are
Every online Steam client on the same account publishes its playable list —
including non-Steam shortcuts — via Remote Play's streamable-games mechanism
(server-brokered since the Sept 2025 client, so it works cross-network). Other
devices render those as stream-only tiles. There is **no** official "shortcut
cloud sync" feature; the aggressive 2026 variant (tiles persisting with Remote
Play off, duplicates) was a bug, fixed in client ~1779155395 / stable 2026-05-27
(steam-for-linux #13141, #12315, SteamOS #2418).

## Facts verified on-device (client 1782861641)
- Synced records live **server-side**, not in local `shortcuts.vdf`; they carry
  the *desktop's* appids and appear in `appStore.allApps` with
  `per_client_data[].clientid` = the remote client id.
- They are stream-only: local Play requires a separate local shortcut; Steam
  re-keys any appid collision, so local+remote can never merge into one tile.
  (The "merged tile" some users see for identically-defined shortcuts is the
  buggy path where the remote entry steals the tile — avoid.)
- Disabling Remote Play (`SteamClient.RemotePlay.SetRemotePlayEnabled(false)`,
  localconfig `EnableStreaming 0`) removes all synced tiles — but kills streaming.

## The per-device hide that actually works
`collectionStore.SetAppsAsHidden([appids], true)` — for **shortcut** appids the
hidden state is stored in per-device local storage
(`localconfig.vdf → "WebStorage" → "user-collections"`), **not** the account's
cloud-synced hidden list (`cloudstorage/cloud-storage-namespace-1.json`, which
only carries store appids). Verified: cloud list byte-identical before/after,
persists across restarts, desktop unaffected, Remote Play stays on. The hidden
tiles remain streamable under Library → Hidden. Community "hiding syncs
everywhere" lore is true only for store games.

UI equivalent: long-press tile → Manage → Hide this game.
New shortcuts added on the desktop appear unhidden and need a one-time hide.

## Secondary knob
`ready_to_play_includes_streaming=false` (Settings → Library → "Ready to Play
should include streamable games") — per-device, only affects that filter, not
All Games. Was broken for non-Steam entries 2023→2026-05.
