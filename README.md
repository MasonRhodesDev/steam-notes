# Steam Notes

Knowledge base for the Steam Deck / Steam client internals learned while setting up
the multi-tenant family Deck (July 2026). Each note is self-contained.

## Index

- [shortcut-appids.md](shortcut-appids.md) — how non-Steam shortcut app IDs are derived, when Steam re-keys them
- [remote-shortcuts.md](remote-shortcuts.md) — why desktop shortcuts appear as "Stream" tiles, and the per-device hide that actually works
- [cef-debugging.md](cef-debugging.md) — driving the Steam client via `-cef-enable-debugging` (SharedJSContext, `SteamClient.*`, stores)
- [flatpak-pitfalls.md](flatpak-pitfalls.md) — sandbox path rules that bit us (config symlinks, RPC sockets)
- [multi-tenant-deck.md](multi-tenant-deck.md) — per-Steam-account app profiles on a shared single-user device
- [steamos-persistence.md](steamos-persistence.md) — what survives SteamOS updates and where to install things
