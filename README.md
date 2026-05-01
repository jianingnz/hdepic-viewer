# HD-EPIC Viewer

Interactive 2D-track viewer for HD-EPIC clips. Each card shows the RGB
clip with the 2D track overlay locked to the video frame.

Live site: https://jianingnz.github.io/hdepic-viewer/

## What's in here

- `index.html` — single-page viewer (Three.js available, currently 2D-only).
- `static/data/manifest.json` — list of 705 clips with metadata.
- `static/data/hdepic/{cid}.json` — per-clip bundle (point cloud, poses,
  2D + 3D tracks, visibility).
- `videos/hdepic/{cid}.mp4` — re-encoded 480p, 15 fps video.
- `.nojekyll` — disables Jekyll on GitHub Pages.

## Clip selection

705 clips picked from ~25 k HD-EPIC clips that have full ViPE
(pose / depth / intrinsics) plus AllTracker 2D + lifted 3D tracks.

**Balanced length sampling.** To cover the full distribution rather than
just the longest clips, pool by clip length and take the top N by score
within each bucket:

| Bucket | Frame range (T) | Duration @ 15 fps | Pool | Picked |
|--------|------------------|-------------------|------|--------|
| short  | 60 ≤ T < 150    | 4–10 s            | 889  | 300    |
| medium | 150 ≤ T < 350   | 10–23 s           | 492  | 300    |
| long   | 350 ≤ T < 1100  | 23–73 s           | 105  | 105    |

Quality scoring per clip:

- `long`     — `log(T) − log(60)` (reward longer clips, hard cut at T = 60)
- `track`    — `mean visibility · (visible_pts / N)` (dense, reliable tracks)
- `motion`   — `tanh(camera_path_length / 0.5 m)` (camera moves enough)
- `spread`   — `tanh(world-space 3D track extent / 1 m)` (avoid degenerate
               clips where every track collapses to a single point)
- `caption`  — caption present and ≥ 3 words
- Combined: `2·long + 3·track + 1.5·motion + 1.5·spread + 1·caption`

Clips with fewer than 60 frames, fewer than 30 raw tracks, fewer than 20
reliably-visible 3D tracks, or degenerate intrinsics are dropped.

All 9 participants (P01–P09) are represented; captions cover 100 % of
clips.

## Alignment

Source video and tracks are both at 15 fps with one track entry per video
frame. The viewer keeps **all** time samples in JSON (no time-axis
subsampling) so every video frame has a matching track frame. The
`requestAnimationFrame` loop redraws at the display refresh rate (~60 Hz)
with linear interpolation between consecutive track samples for smooth
playback even on long clips.

## 2D-only layout

The 3D Scene card is hidden — RGB + 2D tracks fill a single 720 px-max
column. The rAF loop spends all its time on the 2D draw (no Three.js
render per frame) so playback stays smooth on long clips.

2D track styling: 3 px lines, 4 px current dot at 95 % alpha, fading
trail of the last 18 frames.

## Size optimizations

The site is ~895 MB so it fits under GitHub Pages limits:

- Up to 90 query tracks per clip (subsampled along the N axis).
- Point cloud capped at 6 000 points per clip.
- Float values written with `%.4g` formatting (matches float16 precision
  without the FP-noise digits Python's `tolist()` produces).
- Booleans written as `0`/`1` (~3× smaller than `true`/`false`).
- MP4 re-encoded at CRF 32 with `+faststart` (~3.5× smaller than source).

## Running locally

```bash
python3 -m http.server 8000
# open http://localhost:8000/
```

## Source pipeline

Tracks come from the GenTraj pipeline (ViPE poses + monocular depth +
AllTracker 2D points → lifted 3D), with HD-EPIC action annotations as
the text labels.
