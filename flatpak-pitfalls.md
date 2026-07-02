# Flatpak sandbox pitfalls

- **Symlinks must resolve inside the sandbox.** Pointing an app's config dir
  (e.g. `~/.var/app/com.discordapp.Discord/config/discord`) at a path outside
  the app's mounted filesystems leaves a dangling symlink inside the sandbox —
  Discord hangs on a blank window at startup. Keep swap targets inside
  `~/.var/app/<app-id>/…`, which is always mounted rw.
- **Discord RPC socket**: the flatpak exposes it on the host at
  `$XDG_RUNTIME_DIR/app/com.discordapp.Discord/discord-ipc-0` (a socat bridge
  inside the sandbox forwards it). Host-side RPC clients that scan only
  `$XDG_RUNTIME_DIR/discord-ipc-N` need a symlink:
  `ln -sf $XDG_RUNTIME_DIR/app/com.discordapp.Discord/discord-ipc-0 $XDG_RUNTIME_DIR/discord-ipc-0`
  (runtime dir is a tmpfs — recreate per session, e.g. systemd `ExecStartPre`).
- App icons for artwork: `/var/lib/flatpak/app/<id>/current/active/{files,export}/share/icons/…`
  (some apps only ship `scalable/` SVG — render with `rsvg-convert`, which is on
  the SteamOS base image).
- Steam may auto-install a flatpak when a shortcut carries `FlatpakAppID`
  (observed with Google Chrome via SteamOS's `InstallFlatpakAppAndCreateShortcut`
  mechanism) — a flatpak "appearing on its own" usually traces back to this.
- `flatpak install` can partially fail while still printing the "Installing …"
  plan — always verify afterwards with `flatpak info <id>` / `flatpak list --app`.
