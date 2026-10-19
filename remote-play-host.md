# Remote Play: the host side

What the Steam client does on the **host** when a Remote Play client connects,
learned while making a Wayland compositor react correctly to streaming
(October 2026). Everything here is from `~/.local/share/Steam/logs/streaming_log.txt`
on a Linux host.

## A client-initiated desktop stream raises Big Picture on the host

Connecting from a client and streaming the Steam UI (rather than launching a
title) brings up a real Big Picture window on the host. It is an ordinary
window: on Hyprland it has class `steam` and title `Steam Big Picture Mode`,
and a compositor rule keyed on exactly that pair fires within a second of the
stream starting.

This matters if the host has any automation keyed to Big Picture, because a
remote connection will trigger it even though nobody touched the machine.

Not yet confirmed for the other path: a client that launches a specific title
may go straight to the game without raising Big Picture. The behaviour above
was observed only for `Game: UNKNOWN (0)` plus `>>> Starting desktop stream`,
which is what the Steam UI path logs.

## Capture follows the host's foreground window

The host captures whatever is foreground, and the capture size is that
window's geometry, not the monitor's. A tiled Big Picture window on a
3440x1440 display produced:

```
SynchronizeClientState(): setting capture size 3430x1390
```

3430x1390 is the window inside its gaps. Fullscreening the intended window
changed it to 3440x1440 on the next state sync. So "the wrong thing is being
streamed" is almost always "the wrong window is foreground on the host", and
the capture size line is the quickest way to prove which.

## There is no heartbeat in the log: use the boundary pair

This one cost real time. A natural approach to "is a stream live right now" is
to tail the log and look for a recent session marker. **It does not work,
because nothing in this log is periodic.**

- `Connected UDP@`, `Maximum capture:` and `SynchronizeClientState` all fire at
  session start and then stop.
- Per-frame and per-event chatter buries them immediately. Measured during a
  live stream: scanning the last 8KB of the log found **zero** of those
  markers, with the newest one 426 lines further back.
- `dynamicbitrate:` is not a substitute. It appeared 112 times during a
  struggling stream and **zero** times during a healthy one, so it detects
  congestion, not streaming.
- `CLIENT: Video rect:` and `CLIENT: Texture rect:` fire roughly once every two
  minutes, far too sparse for a short freshness window.

The reliable signal is the boundary pair:

```
>>> Starting desktop stream
>>> Stopped desktop stream
```

Across six sessions in one log these were perfectly paired and strictly
alternating. So scan backwards for the most recent of the two and whichever
came last **is** the current state. No freshness window, no timestamp parsing,
and it stays correct through a session of any length.

Both lines say "desktop" even when the activity is a game
(`k_EStreamActivityGame`), so there is no separate game-stream variant to
match.

## The capture method decides how much the stream costs the host

The method is logged once per stream and the difference is large:

| Logged method | What it does | Host cost |
|---|---|---|
| `Game Vulkan NV12 + VAAPI H264` | takes frames from the game directly | low |
| `Desktop OpenGL NV12 + VAAPI HEVC` | reads back the whole desktop, converts, encodes HEVC | high |
| `Desktop Black Frame + VAAPI H264` | transient, seen while switching sources | n/a |

A session on the desktop path at 2856x1196 and 60 FPS produced input lag
**on the host itself**, felt while typing locally, because the readback plus
HEVC encode competes with the compositor for the same GPU. The game path at
1920x1080 on the same machine was smooth at both ends.

So if the host feels sluggish during a stream, check the capture method before
suspecting the network. Streaming a title rather than the Steam UI is what
selects the cheap path.

## Client quality settings are charged to the host too

It is tempting to read quality and bitrate as client-side preferences. They are
not:

```
Streaming quality: k_EStreamQualityBeautiful
Streaming bitrate: Unlimited
```

With those, the host drove its target above 250 Mbit/s and the log filled with
`client video decoder overflow` and `data loss`, while a `Balanced` plus
`Automatic` session on the same host and network logged none of either. One
session recorded 11 decoder overflows, 29 data-loss events and 547 dropped
frames against 0, 0 and 1 for the healthy one.

`Unlimited` costs the host encode time and the client decode headroom at once,
so a phone-class client should not be left on it.

## `Bringing streamed game to foreground` fires too early and never retries

The host logs this exactly once per attempt, and it logged
`Bringing streamed game to foreground - failed` on every attempt recorded.
Timeline from one launch:

```
17:11:31  Bringing streamed game to foreground - failed
17:11:34  Adding window ... for gameID <id>
```

It fires about three seconds **before** the game's windows exist, so there is
nothing to raise, and nothing tries again. Any host-side window rule relying on
Steam to foreground the game will therefore not fire. Something on the host has
to do it, and it has to keep trying as windows appear, including through splash
windows that briefly steal foreground.

## The `-pipewire` warning is not a fault indicator

```
WARNING: Desktop capture unavailable, try running Steam with -pipewire
```

This appears at the start of sessions that then stream perfectly well,
including a smooth game-capture session with zero dropped frames. It is worth
knowing about, but it does not correlate with any fault observed here, so do
not treat its presence as the cause of a problem.
