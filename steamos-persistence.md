# SteamOS: what survives updates

SteamOS updates atomically replace the read-only rootfs (`/usr`, …); `/home` and
`/var` persist, `/etc` persists via overlay.

Safe (persistent) locations:
- `~/.local/bin` — launcher wrappers, static binaries (gh CLI lives here)
- `~/Applications` — AppImages (ES-DE)
- `/var/lib/flatpak` — system flatpaks (survive updates; wiped only by factory reset)
- `~/.var/app` — flatpak app data
- `~/.config/systemd/user` — user services
- `~/.steam/steam/userdata` — per-account Steam config/artwork
- distrobox/podman containers (`~/.local/share/containers`) — the build toolchain

Avoid:
- `/usr`, `/usr/local` — wiped every update; requires `steamos-readonly disable`
- pacman packages — same
- root-level dirs like `/games` — read-only rootfs, can't even be created

Other notes:
- No compiler on the base image (`ld` exists, no gcc/cc). Build in distrobox;
  match glibc: host is 2.41 (July 2026), so a debian:12 (2.36) container's
  binaries run on the host.
- `sudo` needs a password the user must set (`passwd`); flatpak, distrobox,
  systemd --user all work without it.
- Host has python3 (3.13), git, rsvg-convert, socat, podman, distrobox on base.
