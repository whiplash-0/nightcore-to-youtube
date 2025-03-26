# Nightcore to YouTube

A personal automation pipeline that takes a raw audio track, generates slowed/nightcore variants through a browser, renders them into videos, and publishes them to YouTube — all from a single command.

```
Artist - Song.mp3  +  24_1_w.jpg
          ↓
    [browser → nightcore.studio]
          ↓
   85_8.mp3  100_0.mp3  115_3.mp3
          ↓
    [ffmpeg + Pillow]
          ↓
   85_8.mp4  100_0.mp4  115_3.mp4
          ↓
    [YouTube Data API]
          ↓
  "Artist - Song (Slowed)" ✓
  "Artist - Song (Slightly Sped Up)" ✓
  "Artist - Song (Sped Up)" ✓
```

## How it works

The pipeline has three steps, each independently runnable:

**Step 1 — `create-nightcore`**  
Opens [nightcore.studio](https://nightcore.studio/) via Playwright, uploads the source track, sets the speed and reverb sliders via their ARIA values, and downloads the generated audio. Multiple variants are downloaded concurrently in a single persistent browser context.

**Step 2 — `nightcore-to-video`**  
Pairs each generated audio file with the cover image and renders an `.mp4` using `ffmpeg`. The cover is letterboxed to the requested aspect ratio (default `16:9`, up to `32:9`) and encoded with `libx264`. Video rendering runs in a `multiprocessing.Pool`.

**Step 3 — `upload-to-youtube`**  
Reads the filenames, derives human-readable speed labels ("Deeply Slowed", "Sped Up", etc.), builds YouTube titles/descriptions/tags from those and from the cover metadata, then uploads via the YouTube Data API with OAuth. Handles resumable uploads and gracefully stops on daily quota limits.

## The filename contract

Everything the pipeline needs to know lives in filenames. No manifest, no database.

| File | What the name encodes |
|---|---|
| `Artist - Song.mp3` | artist and track title |
| `24_1_w.jpg` | `year_season_playlist` cover metadata |
| `85_8.mp3` / `85_8.mp4` | `speed_reverb` for a generated variant |

This makes every step **restartable**: if Step 1 already ran, Step 2 picks up the existing `.mp3` files without re-downloading. If videos exist, you can upload only a subset. The pipeline recovers intent purely from what's on disk.

A working directory before and after Step 1:

```
my-track/
├── Artist - Song.mp3      ← source track (kept)
├── 24_1_w.jpg             ← cover art
├── 85_8.mp3               ← slowed + reverb
├── 100_0.mp3              ← standard speed
└── 115_3.mp3              ← sped up + light reverb
```

Cover metadata (`24_1_w`) is parsed as discovery year / season / playlist and embedded into the YouTube description. Valid values as currently configured:

- **year**: `23`–`99`
- **season**: `1`–`4`
- **playlist**: `w`, `p`, `e`, `s`

## Installation

This project uses Conda (Python 3.11).

```bash
conda env create -f environment.yml
conda activate nightcore_to_youtube
```

To install the `nightcore-to-youtube` shell alias:

```bash
./setup.sh
```

`setup.sh` also copies a Chrome profile into your Edge config, which the Playwright step needs for its persistent browser context. It assumes:

- Conda at `~/apps/miniconda3`
- A Chrome profile named `Profile 34` under `~/.config/google-chrome/`

> The Playwright step currently hardcodes the Edge user-data dir and profile name in `src/steps/create_nightcore.py`. These are the first things to change if you're running this on a different machine.

### YouTube credentials

Create a `.credentials/` directory at the project root and place your OAuth client secret JSON there. The expected filename pattern is:

```
.credentials/client_secret_<...>.apps.googleusercontent.com.json
```

On first upload, a browser window will open for OAuth consent. The token is then cached as `.credentials/token.pickle` for future runs.

## Usage

```bash
python src/main.py <working-directory> [<speed> [reverb]]...
```

Or via the alias after setup:

```bash
nightcore-to-youtube <working-directory> [<speed> [reverb]]...
```

Speed is a percentage of original tempo (range `50`–`200`). Reverb is optional and defaults to `0` (range `0`–`49`). Pairs are interleaved in the argument list.

### Examples

```bash
# Full pipeline: generate 3 variants, render videos, upload all
nightcore-to-youtube ./my-track 85 8 100 0 115 3

# Step 1 only, keep browser visible
nightcore-to-youtube ./my-track 115 3 125 5 --step 1 --gui

# Resume from existing audio, render in ultrawide 21:9
nightcore-to-youtube ./my-track --step 2 --ratio 21:9

# Upload only the last 2 already-rendered videos
nightcore-to-youtube ./my-track --step 3 --uploaded-video-count -2

# Run steps 1 and 2, skip upload
nightcore-to-youtube ./my-track 85 0 100 0 --steps 1:2
```

### All options

| Option | Short | Description |
|---|---|---|
| `--step N` | `-s` | Run a single step (1, 2, or 3) |
| `--steps N:M` | `-ss` | Run an inclusive step range (default: `1:3`) |
| `--gui` | `-g` | Show browser window during Step 1 |
| `--preset PRESET` | `-p` | `ffmpeg` encoding preset for Step 2 |
| `--ratio W:H` | `-r` | Video aspect ratio for Step 2 (default: `16:9`, max: `32:9`) |
| `--uploaded-video-count N` | `-u` | Upload first N (positive) or last N (negative) videos in Step 3 |

## Project structure

```
src/
├── main.py                    # CLI entry point, validation, step orchestration
├── config.py                  # file conventions, ratio limits, credential paths
├── steps/
│   ├── create_nightcore.py    # Playwright automation against nightcore.studio
│   ├── nightcore_to_video.py  # ffmpeg + Pillow video rendering
│   └── upload_to_youtube.py   # OAuth flow + YouTube upload
└── utils/
    ├── working_directory.py   # directory contract, file discovery
    ├── metadata.py            # cover filename → metadata parsing and formatting
    └── param_types.py         # custom Click parameter types (ratios, ranges)

nightcore_to_youtube.sh        # wrapper: sets PYTHONPATH, activates conda, runs CLI
setup.sh                       # installs alias, copies browser profile
environment.yml                # conda environment spec (Python 3.11)
```
