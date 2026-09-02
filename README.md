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
| YouTube search | Type a query instead of hunting for a link; results show channel and duration |
| Autoplay radio | When the queue empties, offers similar tracks with a countdown — pick one, or the top one plays itself |
| Shared queue | Anyone in the room can add videos; the host's player runs the show |
| Audio-only mode | Hides the video, keeps the sound — for music listening |
| Chat | Plus join/leave and "X added …" activity messages |
| People list | Live presence with the current host marked |
| Host handover | If the host closes their tab, someone else can take over the room |
| Per-person volume | Volume and mute are local — they don't affect the room |
| Control modes | Host-only by default, or open playback control to everyone |

Titles for pasted links are resolved through YouTube's public oEmbed endpoint, so
no API key is needed for those.

### How autoplay picks the next track

Not from YouTube's recommendations — those aren't reachable here. The Data API
dropped `relatedToVideoId` in 2023, and the keyless mirrors return nothing, an
error, or generic filler for music videos (measured: 0 related items for one
music video, HTTP 500 for another, and unrelated clickbait for a third).

So the next track comes from a **search seeded off the current one's artist**.
The channel name is the most reliable artist signal — `Ex Battalion Music`, or
`Florante - Topic` — because title word order is inconsistent (`Artist - Song`
vs `Song - Artist`). Record-label channels are the exception, so those fall back
to the part of the title before the dash. Candidates exclude the current track,
anything already played this session, anything queued, and anything under 30s or
over 25 minutes. Four are offered with a 12-second countdown; the top one plays
if nobody picks. Turn it off under **People → room settings**.

### How search works

Search needs a data source, and the official YouTube Data API requires a key that
has no safe hiding place in a public static page. So:

1. **With your own key** (recommended if search matters to you) — paste it under
   **People → search**. It is stored in that browser's `localStorage` only, never in
   this repo. Get one from Google Cloud → *APIs & Services* → enable **YouTube Data
   API v3** → create an API key, then restrict it to your site's domain. The free
   quota is 10,000 units/day and a search costs 100, so roughly 100 searches a day.
2. **Without a key** — the app falls back to public keyless API mirrors
   (Piped / Invidious). This works with no setup at all, but those instances are
   volunteer-run and most are unreachable at any given time, which is why the app
   tries a list of them and remembers the last one that answered.

## Usage

1. Open the page, enter a name, hit **Create a room**.
2. Hit **Copy invite** and send the link to whoever is joining.
3. Type what you want to hear and press **Search**, then click a result to add it.
   Pasting a link (`youtube.com/watch?v=…`, `youtu.be/…`, `/shorts/…`, or a bare
   11-character video ID) also works — the button switches to **Add** when it
   recognises one.
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
- **Playlist URLs** (`?list=…`) aren't expanded; add individual videos instead.
- **Keyless search is unreliable** by nature — if the dropdown says search is
  unavailable, every mirror in the list was down. Add an API key, or paste a link.
- **Videos whose owners disable embedding can't play in any embedded player** —
  common for record-label uploads, which play on youtube.com but refuse to run
  inside other sites. The stage shows why, with a link to open it on YouTube; the
  host skips ahead automatically only when something else is queued. Searching the
  same song and picking a different channel's upload usually works.
- The sync relay is a **public MQTT broker**: anyone who knows your room code can
  join, and room state passes through a third party. Treat codes as semi-private
  and don't put anything sensitive in chat.
- Mobile browsers may throttle a backgrounded tab, which shows up as drift — the
  **Re-sync** button snaps you back.
