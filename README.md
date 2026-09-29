# Video clipping pipeline with the Shotstack API

`clips.py` turns a long recording into five short, vertical, captioned clips:

1. Shotstack's Ingest API imports the recording and transcribes it.
2. Claude picks the five best moments from the transcript and saves them to `moments.json`.
3. Each moment takes two Shotstack renders. The first cuts it out and fits it into a 1080x1920 frame. The second adds captions generated from the clip's own audio.
4. The finished clips are downloaded to a local folder.

It's a reference implementation to take apart and adapt, not a finished tool.

## What you need

- Python 3.10 or later
- A Shotstack API key. Production renders use credits: see [Shotstack pricing](https://shotstack.io/pricing/).
- An Anthropic API key, for picking the moments

## Setup

```bash
python3 -m venv venv
source venv/bin/activate
python -m pip install -r requirements.txt
export SHOTSTACK_API_KEY=your-shotstack-key
export ANTHROPIC_API_KEY=your-anthropic-key
```

The script uses Shotstack's production API. To try it in the sandbox, also set `SHOTSTACK_ENV=stage` and use your sandbox key. Sandbox renders are watermarked.

## Run it

```bash
python clips.py "https://example.com/recording.mp4"
```

Shotstack fetches the recording from its own servers, so the URL must return the video file itself, not a web page or a list of download mirrors.

To check the result before rendering everything, render the first clip only, then run the same command again without `--limit`:

```bash
python clips.py "https://example.com/recording.mp4" --limit 1
```

The demo clips were made from this recording:

```bash
python clips.py "https://ftp.fau.de/fosdem/2025/k1105/fosdem-2025-5235-beyond-the-readme-crafting-a-better-developer-experience-for-open-source-projects.mp4"
```

## What you get

Everything for one recording goes in `clips/<name>/`:

- `transcript.srt`: the transcript from Shotstack.
- `moments.json`: the moments to cut, each with a start and end in seconds and a title. This file is the contract between the picker and the renders. Edit it by hand, or write it with your own picker, and the renders use it as it is.
- `clip-MMmSSs.mp4`: the finished clips, named by where they start in the recording.
- `manifest.json`: what's been done so far.

Running the same command again carries on from where the last run stopped: it reuses the import and the moments, and renders only the clips that aren't finished. Claude picks new moments when the transcript, model, prompt or settings change, or when you add `--repick`.

Shotstack's links to rendered files expire after 24 hours, so the script adds captions to each cut and downloads each finished clip as soon as it's ready.

## When something goes wrong

- A render that fails is tried again the next time you run the script.
- If a request to start a render times out, the script can't tell whether the render started, so it doesn't send the request again. Check your Shotstack dashboard. If nothing started, delete that clip's entry under `"renders"` in `manifest.json` and run the script again.

## Changing the picker

The moments come from Claude Sonnet 5.5 (`claude-sonnet-5-5`), with structured outputs so the reply is JSON that matches a schema. The transcript goes to Claude one sentence per line, and Claude answers with the first and last words of each moment rather than times or line numbers. The script finds those words in the transcript, so every clip starts and ends on a whole sentence. It checks every moment itself (15 to 60 seconds, inside the recording, no overlaps) and asks Claude once to fix any that break the rules.

To use another model, change `MODEL` and check its thinking settings: Sonnet 5.5 can't turn thinking off, and `{"type": "between_tools"}` is its lowest setting, while Claude Sonnet 5 uses `{"type": "disabled"}`. To use a different picker altogether, write `moments.json` yourself.

## Demo footage

"Beyond the README: Crafting a Better Developer Experience for Open Source Projects" by Lorna Mitchell, FOSDEM 2025. Licensed CC BY 2.0 BE.
