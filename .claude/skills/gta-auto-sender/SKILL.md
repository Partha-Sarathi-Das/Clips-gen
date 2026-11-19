---
name: gta-auto-sender
description: Install and run the GTA Auto Sender (github.com/AnIntellectualBeing/GTA-Auto-Sender), the posting half of the GTA Auto Clipper. It takes finished clips, shares them equally across several YouTube and Instagram channels, and schedules them through Zernio. Use when the user asks to auto-post GTA clips, schedule clips to several channels, set up Zernio posting, or check how many days of posts are queued.
---

# GTA Auto Sender

Takes clips from the GTA Auto Clipper's `output/` folder and posts them to several YouTube and Instagram channels
through Zernio, with an equal share for each channel.

```
clip.py ─> clipper output/ ─> clip_feeder.py ─> queue/<channel>/ready ─> autopost.py ─> Zernio ─> YouTube + Instagram
```

- `clip_feeder.py` gives each clip (best score first) to the channel that has had the fewest, on its next free
  slot. Test clips, non-gameplay clips and vetoed clips (`output/<clip>.bad`) are never sent.
- `autopost.py` uploads what is in `ready/`, schedules it, retries only the platform that failed (no double posts),
  and writes `queue/RUNWAY.md` (how many days of posts each channel has left).

Tested version: commit `d2e8588e5ec7b58bb9d47730a739626584e1cd6d` (all 4 tests passed on Linux, Python 3.13).

## 1. Install

Install the `gta-auto-clipper` skill's tool first. Then:

```bash
TOOLS="${GTA_TOOLS_DIR:-$HOME/gta-tools}"
mkdir -p "$TOOLS" && cd "$TOOLS"
[ -d GTA-Auto-Sender ] || git clone https://github.com/AnIntellectualBeing/GTA-Auto-Sender.git
cd GTA-Auto-Sender && git fetch -q && git checkout -q d2e8588e5ec7b58bb9d47730a739626584e1cd6d
python3 -m venv .venv && .venv/bin/pip install -q -r requirements.txt
.venv/bin/python -m unittest discover -s tests -p "test_*.py"
```

On Windows use `.venv\Scripts\python.exe`. Use a newer commit only when the user asks to update.

## 2. Set up (the user does the account parts)

1. In Zernio the user creates a profile, connects each YouTube channel and Instagram account (Instagram must be
   a Creator or Business account), and creates an API key under Settings > API Keys.
2. `cp .env.example .env`. The **user** puts the key after `ZERNIO_KEY_MAIN=`. The name must match `key_env` in
   `config/channels.json`.
   **Never ask the user to paste a key into the chat, and never print, log or commit one.** In a Claude Code cloud
   session, use `request_credential` for the Zernio host instead.
3. `cp config/channels.example.json config/channels.json` and fill in: the Zernio account IDs for each channel,
   `feeder.clipper_output` (the clipper's `output` folder), `utc_offset`, and posts per hour and per day.
   Both `.env` and `config/channels.json` are git-ignored. Write JSON as UTF-8 without a byte-order mark.

## 3. Test with nothing posted

```bash
.venv/bin/python clip_feeder.py --dry
.venv/bin/python autopost.py --dry
```

Show the user the dry-run output and get a clear yes before turning on real posting.

## 4. Run it every 5 minutes

- **Windows:** `powershell -ExecutionPolicy Bypass -File scripts\install-tasks.ps1` (remove later with `-Remove`).
  `scripts\run-live.ps1 -Url ... -Name ... -Clipper ...` keeps one capture alive per stream.
- **Linux or macOS:** the PowerShell scripts do not apply. Run both scripts from cron every 5 minutes, from the
  kit folder, with the venv's Python. Ask the user before adding anything to their crontab.

Logs are `queue/clip_feeder.log` and `queue/autopost.log`. Check them about 6 minutes after starting.

## Limits that protect the channels

- **YouTube daily cap:** an unverified channel refuses uploads after about 10 in 24 hours, and those clips are lost.
  `youtube_per_day` (default 10) caps it; extra clips go to Instagram only. Phone-verifying a channel raises the cap.
- **Instagram:** about 50 posts per 24 hours per account (`instagram_per_day`, default 45).
- **Pace:** start at 3 to 6 YouTube posts an hour per channel and raise it slowly. Many near-identical uploads can be
  flagged as spam or repetitive content.
- On repeated quota or permission errors, stop posting and tell the user.

## Rules

- Only post clips from sources the user has the right to use, and keep the creator credit.
- Never post test clips, and never upload by hand (`publish.py`) a clip the sender is handling.
- Veto bad clips with `output/<clip>.bad` rather than deleting them.
- Only one session should change the scripts or `config/channels.json` at a time.
- Report only what the tools confirm: Zernio post IDs, statuses and the `RUNWAY.md` numbers.
