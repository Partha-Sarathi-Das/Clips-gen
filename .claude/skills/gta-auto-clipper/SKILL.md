---
name: gta-auto-clipper
description: Install and run the GTA Auto Clipper (github.com/AnIntellectualBeing/GTA-Auto-Clipper), which finds live GTA gameplay streams on YouTube, records them, picks the best moments from chat, audio and speech, and renders full-screen 9:16 clips with pop-up word captions. Use when the user asks to find GTA streams, clip GTA gameplay, make GTA shorts or reels, or set up the GTA clipper.
---

# GTA Auto Clipper

A free, MIT-licensed Python tool. It runs on the user's own computer and **never posts anything by itself**.
Posting is done by the separate `gta-auto-sender` skill, or by hand with `zernio_publish.py`.

Tested version: commit `6047fb862516800f0f768a43347cf8a9b06b6ea3` (all 30 tests passed on Linux, Python 3.13).

## 1. Install

Needs Python 3.10+, `ffmpeg` on the PATH, and Node.js or Deno (yt-dlp uses it to read YouTube).

```bash
TOOLS="${GTA_TOOLS_DIR:-$HOME/gta-tools}"
mkdir -p "$TOOLS" && cd "$TOOLS"
[ -d GTA-Auto-Clipper ] || git clone https://github.com/AnIntellectualBeing/GTA-Auto-Clipper.git
cd GTA-Auto-Clipper && git fetch -q && git checkout -q 6047fb862516800f0f768a43347cf8a9b06b6ea3
python3 -m venv .venv && .venv/bin/pip install -q -r requirements.txt
.venv/bin/python -m unittest discover -s tests -p "test_*.py"
```

On Windows use `.venv\Scripts\python.exe` instead of `.venv/bin/python`.
Use a newer commit only when the user asks to update. Run the tests again after updating.

The first real run downloads a speech model, so it needs a steady connection.

## 2. Keys

- `YOUTUBE_API_KEY` (YouTube Data API) is needed for stream search and live-chat scoring. Without it the clipper
  can still clip a known URL using audio and speech.
- Zernio keys are only needed for posting (see `gta-auto-sender`).
- **Never ask the user to paste a key into the chat, and never print, log or commit one.** Ask them to set it as an
  environment variable themselves. In a Claude Code cloud session, use `request_credential` for the host the key is
  for, or ask them to add it to the environment's secrets.

## 3. Find streams

```bash
.venv/bin/python find_streams.py "GTA 6" --max 25 --json
```

The order is YouTube's view-count search order, **not** the current live viewer count. Say so when showing results.

## 4. Before recording: two checks

1. **Gameplay check.** Look at a real frame of the stream (a screenshot or a short recording) and confirm actual GTA
   gameplay is on screen. A title, thumbnail, discussion stream or facecam is not enough.
   **Never pass `--gameplay-confirmed` unless you actually saw gameplay.** If you cannot see the video, ask the user
   for a screenshot instead of guessing.
2. **Rights check.** Only clip footage the user owns or has permission to use (for example, a creator who allows
   clipping, or a source list the user set up and covers). If rights are unclear, ask once and do not capture.
   Keep the creator credit on every clip.

## 5. Make clips

Live stream (runs until Ctrl+C or the stream ends):

```bash
.venv/bin/python clip.py "https://www.youtube.com/watch?v=VIDEO_ID" --gameplay-confirmed \
  --max-seconds 0 --clip-len 30 --lead-in 6 --top 2
```

Try a local file first (goes to `test_out/`, which can never be published):

```bash
.venv/bin/python clip.py --file /path/to/gameplay.mp4 --test --clip-len 30
```

Useful options: `--moment-source chat|combined|audio` (default `chat`, falls back to audio when chat is empty),
`--layout fill|fit` (default `fill`; keep it unless the user asks), `--spoken-language auto|hi|en`, `--no-chat`,
`--min-score 2.5`, `--chunk 90`, `--top 1`, `--output-dir output/`.

Clips land in `output/` as an MP4 and a JSON file.

## 6. Review every clip

Check that: the moment includes its lead-in and payoff, the player is centred, no webcam or menu fills the frame,
captions are readable, in sync and on one line, the audio is clear, and the credit and metadata are correct.
To veto a bad clip so the sender never posts it, create `output/<clip name>.bad` with one line saying why.

Write titles and captions only from what is really in the clip. Do not make up reactions, events or view counts.

## 7. Posting by hand (optional)

Preview first, then publish only a clip the user approved (`"approved": true` in its JSON):

```bash
.venv/bin/python zernio_publish.py "output/<clip>.mp4"            # preview, nothing is posted
.venv/bin/python zernio_publish.py "output/<clip>.mp4" --publish  # only with the user's OK for this clip
```

Unattended posting also needs `GTA_UNATTENDED_POSTS=1`, both Zernio account IDs and `--unattended`, and only for
sources covered by the user's standing rights policy. For posting to many channels, use `gta-auto-sender` instead.
After posting, report the Zernio post ID and status. If a request fails after the upload, check Zernio before retrying.

## Be honest

Do not say you watched a video, read chat, found a stream or made a clip unless a command actually did it.
