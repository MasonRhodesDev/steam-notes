# Remote Play LAN discovery, wake-on-LAN, and host identity

Static read of the Linux client (manifest 1791918634: `steamclient.so`,
`steamui.so`, `steam`, `streaming_client`) and Steam Link 1.4.2.321, nothing
executed, plus field tests in October 2026. Offsets are file offsets: `SC` is
`steamclient.so`, `SL` is the Steam Link binary. Full evidence (decompiles, asm
excerpts) was kept with the original research, not here. For what the host
does once a stream starts, see [remote-play-host.md](remote-play-host.md).

## Wire format
- UDP 27036, both directions. Packet: signature `FF FF FF FF 21 4C 5F A0`,
  LE `uint32` header length, `CMsgRemoteClientBroadcastHeader`, LE `uint32`
  body length, body chosen by `msg_type` (`steammessages_remoteclient_discovery.proto`).
- Client broadcasts Discovery (type 0) with `client_ids[]` of hosts it knows.
  Host answers Status (type 1) in a randomized burst of three (`SC 0x1728d10`).
  Steam Link polls every 12 s plus jitter (`SL 0x18f710`).
- Port overrides exist as convars: `uRemoteDiscoveryPortOverride`,
  `uRemoteDiscoveryPortTransmit` (`SC 0x170e1af`). Steam Link itself always
  targets 27036.
- **If 27036 can't be bound when Steam starts, Steam does not retry:** it logs
  `Couldn't bind to broadcast port %d, RemoteClientManager disabled` and turns
  off Remote Play, content caching and remote downloads for that whole session.
  Anything squatting the port breaks all of them.

## What makes Steam Link show a host as tappable
Decided entirely in Steam Link (`SL 0x191460`, labels from `SL 0x158cf0`).
Needs: a `client_id` it already knows, `hostname`, `connect_port`, GameStreaming
(bit `2`) in `enabled_services`, and GameStreaming in `supported_services`, or
when that field is absent a `version` from 9 to 199. No auth happens before a
host is listed.

| Condition | Label | Button |
|---|---|---|
| not supported | Unsupported | off |
| not enabled | Disabled | off |
| `screen_locked` | Locked | off |
| `vr_active` / `remoteplay_active` / `games_running` | VR Mode / Remote Play active / Game in progress | **on** |
| otherwise | Start Playing | on |

Discovery carries no game list at all, only the `games_running` bool. Per-app
status for same-account Steam clients rides the PSK-authenticated TCP channel
(`CMsgRemoteClientAppStatus`), and the streamable list is also brokered through
Steam's servers (see [remote-shortcuts.md](remote-shortcuts.md)). Either way it
needs the host's Steam running; Steam Link uses neither.

## Steam Link sends wake-on-LAN; desktop Steam and the Deck don't
- **Trigger:** opening the connect screen for a saved host that isn't answering
  discovery (`SL 0x146800`, from the panel constructor `SL 0x14a2b0`). The UI
  just says `Connecting...`; there is no "waking" text.
- **Packet:** standard 102-byte magic packet (6 x `0xFF`, MAC x 16), sent twice,
  to **UDP 9** at each saved host address and to `255.255.255.255`
  (`SL 0x1961d0`).
- **MAC source:** learned, never configured. Steam Link saves the host's own
  `mac_addresses` field from an earlier Status, per host under
  `/hardware_address`. A host that never advertised to that install gets no
  wake.
- **Then it finishes by itself:** it resends the streaming request every second
  and gives up only after 300 s (`SL 0x149f59`, `cmp rax,0x493e0`); as soon as
  the host reappears in discovery the connect continues.
- **Desktop Steam / Deck:** no WOL strings or code in `steamclient.so`,
  `steamui.so` or `steam`. `streaming_client` carries Steam Link's sender
  (`0x7933f0`) but nothing calls it. Same-account clients also connect to a
  host automatically on discovery, so their TCP connect is not a "user tapped
  Play" signal.
- **Field-tested (2026-10):** from a phone on the LAN, Steam Link woke a host
  that was (a) sitting at a greetd login screen, via a userspace listener on
  UDP 9 that starts the game session
  ([greetd_game_mode](https://github.com/MasonRhodesDev/greetd_game_mode)
  `game-mode-wol`), and (b) in S3 suspend from Big Picture's `-steamos3` idle
  sleep, via the NIC's own WOL (`ethtool` `Wake-on: g`). Both connected and
  streamed with no further input. With the host asleep, the Deck showed no
  Stream option and had no way to wake it.

## client_id: what a host is, and the clone trap
- Clients key a host on header `client_id` (`uint64`). Not hostname, IP, or
  `instance_id` (which changes per Steam process). Host stores it in config as
  `streaming/ClientID` (or `sitelicense/ClientID`).
- **If a host hears its own `client_id` from another machine** (a Discovery or
  Status carrying it, or a ClientIDDeconflict, type 11, listing it), it
  generates a fresh random id and writes it to config (`SC 0x172a370`,
  `Generated client id: %llu`). Every device paired to the old id stops
  recognizing it.
- Consequence: **machines cloned from one Steam config** (copied home dir, VM
  image, imaged Deck) share a `client_id` and will knock each other's pairings
  out the first time both are on the LAN. Delete `streaming/ClientID` on the
  clone before first launch, or expect one re-pair.
- Faking a host's presence (answering discovery as it) runs into the same trap
  and was rejected for that reason.

## Steam Link quick launch
The streaming request (type 5) can carry `gameid` (field 19) and
`stream_interface` (field 20: Default, RecentGames, BigPicture, Desktop,
SteamVR). With no `gameid` the host opens its configured interface. When it
can't, Steam Link says `Quick launch isn't possible. Start streaming normally
and then launch the game from Steam.`

## Not built, for the record
- **Deck "Wake Desktop" shortcut:** a non-Steam shortcut running a script that
  sends the magic packet (MAC from a local config file). Cheap and robust; the
  known answer if the Deck ever needs to wake a host.
- **Decky plugin that wakes on Stream:** possible, but it hooks the library UI
  and breaks with Steam updates.
- **Discovery responder:** viable only for Steam Link and fragile because of the
  `client_id` trap; adds nothing the wake packet doesn't already do.
