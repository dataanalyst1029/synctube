# SyncTube — watch & listen to YouTube together

A single HTML file that lets two or more people watch the same YouTube video, or
listen to the same music, **in sync**, from anywhere. One person hosts; everyone
else follows their playhead automatically.

No account. No server to run. No API key.

## How it works

- **Media source:** the official YouTube IFrame Player API — every client plays the
  real YouTube player, so nothing is downloaded or re-hosted.
- **Sync transport:** MQTT over WebSocket to a free public broker
  (`broker.emqx.io`, with `hivemq` and `mosquitto` as fallbacks). The host
  publishes `{videoId, position, playing, at, queue}` to a **retained** topic every
  2.5s, so even someone who joins late instantly lands at the right moment.
- **Drift correction:** each guest estimates the host's clock offset from the
  median of recent messages, computes where the host *should* be right now, and
  seeks only when drift exceeds the tolerance (1.2s normally, 0.45s in "Tight
  sync" mode).
- **Room codes** are 6 characters and live in the URL hash (`#ABC123`), so the
  invite link is all anyone needs.

## Features

| | |
|---|---|
| Synced playback | Play, pause, seek and track changes propagate to everyone |
| Shared queue | Anyone in the room can add videos; the host's player runs the show |
| Audio-only mode | Hides the video, keeps the sound — for music listening |
| Chat | Plus join/leave and "X added …" activity messages |
| People list | Live presence with the current host marked |
| Host handover | If the host closes their tab, someone else can take over the room |
| Per-person volume | Volume and mute are local — they don't affect the room |
| Control modes | Host-only by default, or open playback control to everyone |

Titles are resolved through YouTube's public oEmbed endpoint, so no API key is needed.

## Usage

1. Open the page, enter a name, hit **Create a room**.
2. Hit **Copy invite** and send the link to whoever is joining.
3. Paste a YouTube link (`youtube.com/watch?v=…`, `youtu.be/…`, `/shorts/…`, or a
   bare 11-character video ID) and press **Add**.
4. Everyone taps once to enable sound — browsers block audible autoplay until a
   real click happens — and playback stays locked together from then on.

## Running it locally

The YouTube player needs a real origin, so serve the file over http rather than
opening it via `file://`:

```bash
python -m http.server 8787
```

Then visit `http://127.0.0.1:8787/`.

## Limitations

- The **host's tab must stay open** — it is the source of truth. If it closes,
  another person can take over as host.
- **Playlist URLs** (`?list=…`) aren't expanded; paste individual videos instead.
- Videos whose owners disable embedding can't play in any embedded player. The
  host automatically skips to the next queued track when that happens.
- The sync relay is a **public MQTT broker**: anyone who knows your room code can
  join, and room state passes through a third party. Treat codes as semi-private
  and don't put anything sensitive in chat.
- Mobile browsers may throttle a backgrounded tab, which shows up as drift — the
  **Re-sync** button snaps you back.
