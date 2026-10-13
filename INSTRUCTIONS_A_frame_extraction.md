# INSTRUCTIONS A: New notebook `extract_drop_frames.ipynb`

Read `CONTEXT.md` first (especially §3 and §4b). This notebook turns recorded videos into
**lossless PNG frames that contain a drop**, together with the background each frame should be
compared against and a `frames.csv` index. Notebook B (`droplet_size_measurement_v2.ipynb`) reads
this output. Nothing here measures size.

Build it as a **Jupyter notebook with clearly separated cells**. The operator debugs
cell-by-cell in VS Code. Use plain, readable functions. Every parameter goes in the config cell,
with no magic numbers buried in functions.

---

## 0. Environment
Python 3. Libraries: `numpy`, `opencv-python` (`cv2`), `pandas`, `matplotlib`, `scipy`. Read
video with `cv2.VideoCapture`. Stream frames; **never load a whole video into memory**. A
1440×1080 video of ~1000 frames is ~1.5 GB as float32.

## 1. Folder convention and config cell

```
DATE_DIR/                      e.g.  ...\Drop size measurements\2026.10.12
  drops/                       input videos (.avi preferred; .mp4 ignored by default)
  calibration/                 calibration-grid images (used by notebook B, not here)
  frames/                      <- created here
    <video_stem>/              one folder per video, e.g. frames/evap_30/
      evap_30_f000055.png      saved drop frames (8-bit grayscale, lossless)
      bg/evap_30_bg000050.png  background images (see section 3)
      qc_detection.png         summary figure (section 5)
    frames.csv                 index of every saved frame, all videos
```

Config (validated defaults):

```python
DATE_DIR        = r"...\Drop size measurements\2026.10.12"
VIDEO_SUBDIR    = "drops"
FRAMES_SUBDIR   = "frames"
VIDEO_EXTS      = (".avi",)          # the .mp4 copies are lower-quality re-encodes; ignore
VIDEOS          = None               # None = every video in drops/; or a list of stems, e.g. ["evap_30", "evap_31"]
FRAME_RANGES    = {"evap_29": (0, 450)}   # optional per-video (start, stop) frame window; camera was bumped after ~450
FPS_OVERRIDE    = None               # None = read from the file (evap_*: 3601/200 = 18.005 fps)

# Background (rolling temporal median, CONTEXT section 4b)
BG_STEP         = 5      # use every 5th frame ...
BG_HALF_SPAN    = 10     # ... 10 of them on each side of the current frame (+-50 frames, ~+-2.8 s)
BG_REFRESH      = 25     # recompute + save a new background every 25 frames; frames in between reuse it

# Lit-band detection (where drops can be)
BAND_FRAC       = 0.5    # band = rows whose median-background row mean > BAND_FRAC * max
BAND_MARGIN     = 15     # px trimmed inside each band edge

# Drop presence
K_DETECT        = 6.0    # pixel counts as "drop" if dev > K_DETECT * sigma   (dev = bg - frame)
MIN_DROP_AREA   = 15     # px
MAX_DROP_AREA   = 1500   # px; larger = artifact / band shift
EDGE_MARGIN     = 30     # px from left/right frame edge

# Focus (soft filter)
SHARPNESS_MIN   = None   # None = keep all detected frames (flag only). Set a number to skip blurry ones.
SAVE_PNG        = True
```

## 2. Cell: helpers
- `list_videos()` resolves `VIDEOS`/`VIDEO_EXTS`, sorted naturally (evap_9 before evap_10).
- `iter_frames(path, start, stop)` is a generator of `(index, gray_uint8)`. Convert with
  `cv2.COLOR_BGR2GRAY`.
- `video_info(path)` returns fps (`CAP_PROP_FPS`, or `FPS_OVERRIDE`), n_frames, width, height.
  Print them.

## 3. Cell: rolling background
For frame `i`, the background is the per-pixel **median of frames
`i ± k·BG_STEP` for k = 1..BG_HALF_SPAN** (frame `i` itself excluded), clipped to the video's
frame window. To keep this fast:

- Recompute the background only every `BG_REFRESH` frames (centred on that frame). Frames in
  between use the nearest one.
- Keep a ring buffer of the decoded frames needed. Only the sampled frames have to be retained.
  `collections.deque` of uint8 arrays is fine. A two-pass approach is also fine: first decode
  the sampled frames, then stream again.
- Save each background as 8-bit PNG under `frames/<stem>/bg/<stem>_bg<NNNNNN>.png`, where NNNNNN
  is the centre frame index. Round the median. Each saved frame records which background it uses.

Also compute, once per video from a global median of every 25th frame, the **lit band** rows
`(band_top, band_bottom)` per §1. With the evap_29 frame window this stays stable. Store it per
video.

## 4. Cell: detect drop frames, save PNGs, write `frames.csv`
For each frame in the window:

1. `dev = bg.astype(float32) − frame.astype(float32)` (positive = darker than background).
2. `sigma = 1.4826 × median(|dev − median(dev)|)` over the band rows.
3. `mask = GaussianBlur(dev, 3×3) > K_DETECT · sigma`, zero outside the band (± margin) and
   within `EDGE_MARGIN` of the left/right edges. Connected components, keeping
   `MIN_DROP_AREA ≤ area ≤ MAX_DROP_AREA`.
4. If ≥1 component survives, the frame is a **drop frame**. For each component record centroid,
   area and a **sharpness** score: the mean gradient magnitude of `dev` on the component's
   boundary, divided by its core depth (90th percentile of `dev` in the component). If
   `SHARPNESS_MIN` is set and every component is below it, skip saving and log the frame with
   `saved = False, reason = "blurry"`.
5. Save the **raw grayscale frame** (not `dev`, not cropped) as
   `frames/<stem>/<stem>_f<NNNNNN>.png` using `cv2.imwrite`. PNG is lossless.
6. Append a row to `frames.csv`:

| column | meaning |
|---|---|
| `video` | stem, e.g. `evap_30` |
| `frame` | 0-based frame index in the video |
| `time_s` | `frame / fps` |
| `fps` | value used |
| `png` | relative path of the saved frame |
| `bg_png` | relative path of the background to use for this frame |
| `band_top`, `band_bottom` | lit-band rows used |
| `sigma` | noise of `dev` in the band |
| `n_candidates` | components found |
| `max_area_px`, `max_depth` | of the largest component |
| `sharpness_min`, `sharpness_max` | over components |
| `saved`, `reason` | bool; reason if not saved |

Print a per-video summary: frames scanned, drop frames, saved, skipped, and the time taken.

## 5. Cell: QC per video (`frames/<stem>/qc_detection.png`)
One figure per video:

- (a) The global median background, with the band rows drawn.
- (b) `n_candidates` vs frame index. Drop lifetimes appear as runs.
- (c) Max component area vs frame.
- (d) Noise σ vs frame. evap_29 should show σ rising after ~630 if the frame window is removed,
  so this is the bump detector.
- (e) A 4×4 contact sheet of 60×60 px crops around the largest component for 16 evenly spaced
  drop frames, contrast-stretched, `interpolation="nearest"`, `aspect="equal"`.

## 6. Acceptance checks (put in a final cell)
On `evap_30.avi` with the defaults:

1. fps reads 18.005; 1057 frames scanned.
2. Frames 55–74 are all detected as drop frames, and frames 50–54 are not.
3. **No candidate is ever reported at the static dust location** (x≈535, y≈708) in more than
   2% of frames. This proves the temporal background removes it.
4. Roughly 600 drop frames are found. The validation prototype found 600 frames with a drop.
   Accept 520–680, since this depends on `K_DETECT`.
5. On `evap_29.avi` with `FRAME_RANGES` applied, only frames 0–449 are scanned, and σ stays
   ≈10–11 throughout.

## 7. Error handling
- Missing `drops/` folder or no matching videos: a clear, named error.
- Unreadable video or a decode failure mid-stream: warn, keep what was processed, continue to
  the next video.
- Re-running overwrites `frames/<stem>/` for the videos processed in this run only, and
  rewrites `frames.csv` by replacing those videos' rows and keeping the others.
