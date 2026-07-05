# SteamOS: how Gaming Mode ↔ Desktop Mode switching works (2026 builds)

Old guides describe shell scripts (`gamescope-session` in `/usr/bin`,
`startplasma-steamos-oneshot`). Gone. Current builds (steamos-manager 26.x) do it
all through **SDDM autologin** plus Valve's Rust daemon **steamos-manager**.

## The engine: SDDM autologin drop-ins

`/etc/sddm.conf.d/`, read lexically, last file wins per key:

| File | Contains | Written by |
|---|---|---|
| `steamos.conf` (OS image) | `Relogin=true`, `Session=gamescope-wayland.desktop`, **`User=deck`** | Valve, static |
| `zz-steamos-autologin.conf` | `Session=` — persistent default mode | steamos-manager root daemon |
| `zzt-steamos-temp-login.conf` | `Session=` — one-shot override (zzt > zz-) | same; auto-deleted after next successful session start, plus `ExecStartPre=-rm` failsafe on sddm.service |

Every switch = end the current session; `Relogin=true` makes SDDM immediately
autologin again, re-reading the drop-ins (confirmed: SDDM logs "Session …
selected" each cycle). **The user is never dynamic** — only the session. `User=`
comes solely from `steamos.conf`.

## Call chain

- `steamos-session-select` is a compat shim for the Steam client → calls `steamosctl`.
- `steamosctl` → session-bus `com.steampowered.SteamOSManager1`
  (`SessionManagement1`: `SwitchToDesktopSession`, `SwitchToLoginMode`,
  `DefaultLoginMode`, `ValidDesktopSessions`) → user daemon relays privileged
  writes to the root daemon on the system bus (`RootManager.SetTemporarySession` /
  `SetDefaultSession`), which writes the drop-ins above.
- Gaming → Desktop: root daemon writes `zzt` with the plasma session, user daemon
  stops `gamescope-session.target`, SDDM relogs into Plasma, then
  `steamos-manager-session-cleanup.service` (user unit) deletes `zzt` — so a crash
  or reboot lands back in Gaming Mode.
- Desktop → Gaming: the "Return to Gaming Mode" icon is just
  `qdbus org.kde.Shutdown /Shutdown org.kde.Shutdown.logout` when the default mode
  is `game` (no privileged call at all); otherwise `steamosctl switch-to-game-mode`.

## Useful facts

- `/etc/pam.d/sddm-autologin` uses `pam_permit` — **any** user with a valid shell
  can autologin passwordless; no `autologin` group gate.
- The Gaming Mode session is `gamescope-session.target` (user manager);
  `start-gamescope-session` (from the wayland-sessions .desktop) just starts it and
  waits. Real env setup lives in `/usr/lib/steamos/gamescope-session`
  (`gamescope-session.service`).
- `steamos-polkit-helpers/*` (reboot, poweroff, **restart-sddm**, update, …) are
  pkexec wrappers pre-authorized `allow_active=yes` for any active console user —
  not deck-specific.
- `/usr/lib/steamos/steamos-create-homedir <uid>` provisions a home for any UID
  (skel + Steam desktop links + gamemoded user unit); stock only runs it for 1000.
- Raw evidence + a multi-user session-switcher design built on these facts:
  `steamos-multiuser` repo (private) — SDDM `zzz-` drop-in with `User=`,
  `family-switch@.service` + polkit rule, `systemctl restart sddm`.
