---
name: gopro-overlay
description: >
  Overlay GPS, power, heart rate, and cadence data from a Garmin FIT file onto GoPro video footage
  using gopro-dashboard-overlay. Use this skill whenever the user wants to add a data overlay to
  cycling/running/activity video, combine FIT/GPX data with GoPro footage, create race videos with
  power/speed/HR data, or mentions gopro-dashboard-overlay. Also triggers for "overlay my ride data",
  "add power data to my video", "combine my Garmin data with my GoPro", or any mention of FIT files
  and video together.
---

# GoPro Dashboard Overlay

Overlay ride data (power, speed, HR, cadence, maps) from a Garmin FIT file onto GoPro video.

## Prerequisites

- `uv` (Python package manager)
- `ffmpeg` and `ffprobe`
- macOS (for `mac_hevc` hardware encoding profile)

## Workflow

### Step 1: Find files

Look for `.MP4` video files and `.fit` files in the working directory. Match them by name or ask
the user which pairs go together.

### Step 2: Set up the environment

If not already installed, create a venv and install `gopro-overlay`:

```bash
cd <working-directory>
uv venv .venv
uv pip install gopro-overlay
```

The tool binary will be at `.venv/bin/gopro-dashboard.py`.

Also verify `ffmpeg` is installed (`which ffmpeg`). If not, install via `brew install ffmpeg`.

### Step 3: Determine the correct video start time

This is the critical step. GoPro cameras often store an incorrect `creation_time` in their MP4
metadata due to timezone misconfiguration. The embedded SMPTE timecode is far more reliable because
it comes directly from the camera's wall clock at recording time.

#### 3a: Extract the GoPro timecode

```bash
ffprobe -v quiet -print_format json -show_streams <VIDEO>.MP4
```

Look for the `timecode` field in the stream tags (e.g., `"timecode": "13:45:07;06"`). This is the
camera's local wall-clock time when recording started. The frame number after the semicolon can be
ignored.

#### 3b: Determine the timezone from the FIT file

Read the FIT file's `activity` message, which contains both a UTC `timestamp` and a `local_timestamp`:

```python
import fitdecode
with fitdecode.FitReader('<FILE>.fit') as f:
    for frame in f:
        if isinstance(frame, fitdecode.FitDataMessage) and frame.name == 'activity':
            utc_ts = frame.get_value('timestamp')
            local_ts = frame.get_value('local_timestamp')
            offset_hours = (utc_ts.replace(tzinfo=None) - local_ts).total_seconds() / 3600
            print(f"UTC offset: {offset_hours} hours")
```

For example, if UTC timestamp is `20:53:12` and local is `13:53:12`, the offset is +7 hours (PDT).

#### 3c: Convert timecode to UTC

Add the timezone offset to the timecode. For example:
- Timecode: `13:45:07` local
- Offset: +7 hours (PDT)
- UTC start time: `20:45:07`

#### 3d: Set the file modification time

gopro-dashboard doesn't accept a custom datetime, so we use `touch` to set the file's modification
time and then tell the tool to use it:

```bash
TZ=UTC touch -t <YYYYMMDDHHmm.ss> <VIDEO>.MP4
```

Verify with:
```bash
TZ=UTC stat -f "%Sm" -t "%Y-%m-%dT%H:%M:%S UTC" <VIDEO>.MP4
```

#### 3e: Quick alignment check before full render

Before rendering the full video, render a short clip and verify alignment. The timecode-derived
estimate is usually within a few seconds, but the GoPro's clock often drifts 4–8s from the Garmin's
GPS-synced clock and that drift will be very visible in the final video.

Render a 1–2 minute test clip (covering at least one full lap if it's a circuit race) with the
gopro-dashboard command from Step 6. This takes ~60–90s instead of 25+ minutes for the full video.

```bash
ffmpeg -y -ss <OFFSET_SECONDS> -i <VIDEO>.MP4 -t 120 -c copy <VIDEO>-trim.mp4
TZ=UTC touch -t <ADJUSTED_TIMESTAMP> <VIDEO>-trim.mp4
```

The touch timestamp for the trimmed clip must account for the trim offset:
`trim_timestamp = original_video_start_utc + trim_offset_seconds`

#### 3f: Verify alignment using map dot vs camera at corners

**This step requires Claude to be able to read images** (the Read tool on .jpg files, or computer
vision via screenshots). If you can't read images, skip to **3f-manual** below.

The rendered overlay includes a map with a position dot drawn from FIT GPS data. When the dot
reaches a circuit corner, the camera should also be turning that corner. Any mismatch is your
alignment offset, measured directly from the rendered video.

Why this works (and why naive cross-correlation doesn't): for a circuit race, ACCL/speed signals
are highly periodic across laps, so cross-correlating GoPro accelerometer with FIT speed produces
multiple equally-strong peaks at lap intervals. Matching a *spatial* event (dot position on the
circuit map) is unambiguous regardless of which lap you're on.

**Procedure:**

1. **Find sharp corners from FIT GPS.** Detect moments where the heading changes >60° (smoothed
   over ±5s). Take the sharpest 2–3 corners that fall within the test clip's time window:

   ```python
   import fitdecode, datetime, numpy as np
   video_start_utc = datetime.datetime(...)  # touch timestamp of trimmed clip, as UTC
   clip_duration = 120  # seconds

   positions = []
   with fitdecode.FitReader('<FILE>.fit') as f:
       for frame in f:
           if isinstance(frame, fitdecode.FitDataMessage) and frame.name == 'record':
               ts = frame.get_value('timestamp')
               lat_raw = frame.get_value('position_lat', fallback=None)
               lon_raw = frame.get_value('position_long', fallback=None)
               if ts and lat_raw is not None and lon_raw is not None:
                   t = (ts.replace(tzinfo=datetime.timezone.utc) - video_start_utc).total_seconds()
                   if 0 <= t <= clip_duration:
                       positions.append((t, lat_raw * (180.0/2**31), lon_raw * (180.0/2**31)))

   headings = []
   for i in range(1, len(positions)):
       dlat = positions[i][1] - positions[i-1][1]
       dlon = positions[i][2] - positions[i-1][2]
       headings.append((positions[i][0], np.arctan2(dlon, dlat) * 180/np.pi))

   corners = []
   for i in range(5, len(headings)-5):
       hb = np.mean([h for _,h in headings[i-5:i]])
       ha = np.mean([h for _,h in headings[i+1:i+6]])
       diff = abs(ha - hb); diff = 360-diff if diff > 180 else diff
       if diff > 60:
           corners.append((headings[i][0], diff))

   unique = []
   for c in corners:
       if not unique or c[0] - unique[-1][0] > 5:
           unique.append(c)
   for t, d in sorted(unique, key=lambda x: -x[1])[:3]:
       print(f"corner candidate: t={t:.0f}s, turn={d:.0f}°")
   ```

2. **Extract a frame at each corner time, cropped to show both the map and the camera view.**
   For Max Lens Mod (3840x3360), the map is in the upper-right; crop with `crop=1200:1400:2640:0`.
   For 1920x1080 layouts, adjust the crop to capture the map widget along with enough of the camera
   image to read body lean and the road ahead:

   ```bash
   for t in <corner_t1> <corner_t2> <corner_t3>; do
     ffmpeg -y -ss $t -i <VIDEO>-trim-overlay.mp4 -frames:v 1 \
       -vf "crop=1200:1400:2640:0" /tmp/align_corner_${t}s.jpg
   done
   ```

3. **Read each frame and check that the dot and camera agree.** For each frame:
   - Find the position dot inside the zoomed circuit map (typically a small map with a red rectangle
     showing the circuit boundary). Note where the dot sits on the rectangle.
   - Find the camera view. A bike mid-corner shows: visible frame tilt, road curving in the
     direction of travel, the rider leaning, often a speed dip and high power on the way out.
   - If the dot is at a corner of the rectangle AND the camera is mid-turn → aligned.
   - If the dot has already passed the corner while the camera is still on the straight before it
     → overlay is **ahead** → decrease the touch timestamp.
   - If the camera is already past the corner while the dot is still on the straight before it
     → overlay is **behind** → increase the touch timestamp.

4. **Estimate the offset by extracting more frames around the corner.** If misaligned, extract
   additional frames at ±2s, ±4s, ±6s around the corner time and find the offset where dot and
   camera agree:

   ```bash
   CORNER_T=22
   for d in -6 -4 -2 0 2 4 6; do
     t=$((CORNER_T + d))
     ffmpeg -y -ss $t -i <VIDEO>-trim-overlay.mp4 -frames:v 1 \
       -vf "crop=1200:1400:2640:0" /tmp/align_${CORNER_T}_off${d}s.jpg
   done
   ```

   The offset `d` of the frame where the camera is mid-corner (while the FIT-driven dot was at the
   corner at `t=CORNER_T`) is the correction in seconds. Apply it to the touch timestamp:
   - **Camera reaches corner *later* than dot** (overlay ahead) → subtract `d` from the timestamp.
   - **Camera reaches corner *earlier* than dot** (overlay behind) → add `d` to the timestamp.

5. **Re-touch and re-render the test clip** with the adjusted timestamp. Repeat verification until
   dot and camera agree at multiple corners.

When applying the correction to the full video, use the same offset on the original (untrimmed)
video's UTC start time.

#### 3f-manual: Fallback when image reading isn't available

If you can't read images, render the test clip and ask the user to verify alignment. The GoPro's
clock often drifts several seconds from the Garmin's GPS-synced clock; offsets of 4–8s are common.
If the user reports the overlay is ahead or behind:
- **Overlay ahead of video**: decrease the touch timestamp (shift earlier)
- **Overlay behind video**: increase the touch timestamp (shift later)

After adjusting, re-render the short test clip and ask the user to verify before committing to the
full render. When applying the correction to the full video, subtract the same offset from the
original (untrimmed) video's UTC start time.

> **Don't try cross-correlating GoPro ACCL with FIT speed/power for circuit races.** It looks
> attractive (both signals exist, both reflect the same physical activity) but lap-by-lap
> periodicity creates many equally-strong correlation peaks at multiples of the lap period, and the
> reported "best" peak is often wrong by tens of seconds. Use the spatial corner-matching approach
> above instead.

### Step 4: Check video resolution

```bash
ffprobe -v quiet -print_format json -show_streams <VIDEO>.MP4
```

Read `width` and `height` from the video stream. Common GoPro resolutions: 3840x2160, 1920x1080,
3840x3360 (Max Lens Mod), 5312x2988, etc.

### Step 5: Create or scale the layout XML

If the user has a layout XML file, check if its resolution matches the video. Layout files are
typically named like `power-1920x1080.xml`.

If the resolutions don't match, create a scaled version. The overlay size must match the video
resolution exactly, otherwise the overlay will appear in just one corner of the frame:
- Scale all absolute x positions by `(video_width / layout_width)`
- Scale all absolute y positions by `(video_height / layout_height)`
- Scale font `size`, icon `size`, `width`, `height` attributes uniformly by
  `(video_width / layout_width)` to maintain proportions
- Scale relative positions within composites (child x/y offsets) by the same uniform factor

Save as `power-<width>x<height>.xml`.

If no layout XML exists, omit `--layout-xml` and the tool will use its default layout. The user
can customize later.

### Step 6: Run gopro-dashboard

```bash
.venv/bin/gopro-dashboard.py \
  --use-fit-only \
  --fit <FILE>.fit \
  --font "/System/Library/Fonts/Supplemental/Arial Bold.ttf" \
  --video-time-start file-modified \
  --profile mac_hevc \
  --overlay-size <WIDTH>x<HEIGHT> \
  --layout-xml <LAYOUT>.xml \
  <VIDEO>.MP4 \
  <VIDEO>-overlay.mp4
```

Key flags:
- `--use-fit-only`: Use only the FIT file for data (no GoPro GPS needed)
- `--video-time-start file-modified`: Align using the file modification time we set via touch
- `--profile mac_hevc`: Apple hardware H.265 encoding (fast, good quality). Use `mac` for H.264
- `--overlay-size`: Must match the video resolution exactly
- `--font`: Required on macOS since the default Roboto font isn't installed

The render runs at ~15 fps with `mac_hevc`. A 5-minute video takes ~3-4 minutes. A 35-minute
video takes ~25 minutes. Run long renders in the background.

### Step 7: Verify

Ask the user to review the output video, checking:
1. **Alignment**: Does the data match what's happening in the video? (e.g., power spikes when
   sprinting, speed drops in corners)
2. **Layout**: Are overlay elements properly positioned and readable?
3. **Data**: Do values look reasonable for the activity?

If alignment is off, go back to Step 3f (or 3f-manual) — re-render a short test clip with the
adjusted timestamp before re-rendering the full video.

## Available profiles

| Profile | Codec | Speed | Transparency | Use case |
|---------|-------|-------|-------------|----------|
| `mac_hevc` | H.265 (hardware) | Fast | No | Final output on Mac |
| `mac` | H.264 (hardware) | Fast | No | Final output on Mac |
| `mov` | PNG (lossless) | Very slow | Yes | Transparent layer for KDENLive compositing |
| `vp9` | VP9 | Slow | Yes | Transparent layer for web |

Avoid `mov` profile unless transparency is specifically needed — it produces enormous files
(100+ GB for a 5-minute video) and renders at ~0.6 fps vs ~15 fps for `mac_hevc`.

## Troubleshooting

- **"Unable to load font"**: Add `--font "/System/Library/Fonts/Supplemental/Arial Bold.ttf"`
  (or any installed .ttf)
- **Overlay in wrong corner**: `--overlay-size` doesn't match the video resolution — must be exact
- **Overlay data ahead/behind**: Adjust the `touch` timestamp by a few seconds (see Step 3f).
  Use the test-clip workflow to iterate quickly
- **Very slow render**: Use `--profile mac_hevc` instead of `--profile mov`
- **No FIT data in time window**: The timecode-to-UTC conversion may be wrong; double-check the
  timezone offset from the FIT activity message
- **Standing still at start of video**: The FIT recording and video may have started before the
  actual race. Check when speed first exceeds ~15 kph to find the action

## Multiple videos

When processing multiple videos with the same FIT file, each video needs its own timecode-to-UTC
conversion since they were recorded at different times during the activity. Each video may also
need its own fine-tuning offset — the GoPro clock drift can vary.

Process them sequentially (not in parallel) for best render performance. Always do a test-clip
alignment check for each video before committing to the full render.
