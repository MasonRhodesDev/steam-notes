# SteamOS: what survives updates

SteamOS updates atomically replace the read-only rootfs (`/usr`, …); `/home`
persists. `/etc` and `/var` are **more subtle than "they persist"** — see below.

Safe (persistent) locations:
- `~/.local/bin` — launcher wrappers, static binaries (gh CLI lives here)
- `~/Applications` — AppImages (ES-DE)
- `/var/lib/flatpak` — system flatpaks (offloaded to `/home`; wiped only by factory reset)
- `~/.var/app` — flatpak app data
- `~/.config/systemd/user` — user services
- `~/.steam/steam/userdata` — per-account Steam config/artwork
- distrobox/podman containers (`~/.local/share/containers`) — the build toolchain

Avoid:
- `/usr`, `/usr/local` — wiped every update; requires `steamos-readonly disable`
- pacman packages — same
- root-level dirs like `/games` — read-only rootfs, can't even be created

## /etc and /var: the real rules (verified July 2026)

- A/B slots each have their own tiny var partition (`var-A`/`var-B`, 256M).
  `/etc` is an overlayfs whose upper dir lives on the **active slot's** var
  (`/var/lib/overlays/etc/upper`) — so raw `/etc` edits are per-slot.
- On each atomic update, `/etc` changes are **dropped except a whitelist**:
  `/usr/lib/rauc/atomic-update-keep.conf`. Dropped state is backed up to
  `/var/lib/steamos-atomupd/etc_backup` and the previous kept files appear in
  `/etc/previous/`.
- Kept by default (highlights): `/etc/sddm.conf.d/*`;
  `/etc/passwd|group|shadow|gshadow|subuid|subgid` (→ **created users survive**);
  `/etc/systemd/system/*.service`, `*.service.d/**`, `*.wants/**`, `*.socket`,
  `*.mount`; NetworkManager connections; ssh host keys; hostname/localtime/
  machine-id; `/etc/environment`.
- **Not** kept: `/etc/polkit-1/rules.d/*`, pam.d, udev rules, most everything else.
- Escape hatch: drop a `*.conf` into `/etc/atomic-update.conf.d/` listing extra
  globs to preserve (that dir is itself whitelisted). `*` matches within a path
  segment, `**` crosses `/`.
- `/var` proper is per-slot, but the big dirs are offloaded to the shared home
  partition: `/var/lib/flatpak`, `/var/lib/docker`, `/var/log`, `/var/cache/pacman`,
  `/var/tmp`, plus `/opt`, `/root`, `/srv`, `/nix`.

Other notes:
- No compiler on the base image (`ld` exists, no gcc/cc). Build in distrobox;
  match glibc: host is 2.41 (July 2026), so a debian:12 (2.36) container's
  binaries run on the host.
- `sudo` needs a password the user must set (`passwd`); flatpak, distrobox,
  systemd --user all work without it.
- Host has python3 (3.13), git, rsvg-convert, socat, podman, distrobox on base.
