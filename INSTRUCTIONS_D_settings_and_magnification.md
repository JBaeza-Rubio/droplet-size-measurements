# INSTRUCTIONS D: per-video settings table and magnification correction

Do this **after** the current run (INSTRUCTIONS C) has finished and its acceptance checks pass.
All of it is in `droplet_size_measurement_v2.ipynb`. One part (§3) also touches
`extract_drop_frames.ipynb`.

## Why
evap_30 was recorded with the zoom at about 1.8x. The calibration slide was imaged at 2x. Every
evap_30 size is therefore about 11% too small in µm, and its K about 24% too small. The camera
zoom changes the image scale, so a single calibration is only valid for videos taken at the same
zoom.

The lit band in every frame is the gap between the trap rods, which has a fixed physical width.
Its height in pixels is therefore a built-in magnification gauge. Measured during design (sub-pixel
half-level crossings, median over column strips):

| image | band height (px) | ratio to calibration |
|---|---|---|
| calibration evap_41 / 44 / 46 | 621.5 / 620.4 / 621.0 | 1 |
| evap_29 (frames 0–449) | 623.0 | 1.003 |
| evap_30 | 557.6 | **0.898** |
| evap_31 | 626.5 | 1.009 |

## 1. Settings table
Copy `videos.csv` (provided next to this file) to `<DATE_DIR>/videos.csv`, e.g.
`F:\Imaging\2026.10.12\videos.csv`. It has one row per video: drop rate, zoom note, gain, black
level, laser power, inkjet height, a `settings_certain` flag, an optional usable frame window
(`frame_start`, `frame_stop`), and notes. Don't edit the values. If a video in `frames.csv` is
missing from it, just print a warning.

## 2. Per-video magnification correction (measurement notebook)
- **Band height of a video:** take the median of that video's background PNGs from notebook A.
  For each of ~12 column strips 100 px wide (x = 100…1340), average the strip into a row profile.
  Let `lo` = median of the top and bottom 150 rows (dark), `hi` = median of rows 400–700 (lit),
  and `h = (lo + hi)/2`. The band height is (last row above h) − (first row above h), with both
  edges linearly interpolated to sub-pixel. Use the median over the strips.
- **Band height of the calibration:** the same measurement on each usable calibration image, but
  only on column strips **outside** the grid ROI, because the grid lines disturb the profile.
  Use the median over the calibration images.
- `mag_ratio_measured = band_video / band_calibration`.
- Config `MAG_CORRECT_MIN = 0.02`. If |mag_ratio_measured − 1| > MAG_CORRECT_MIN, apply
  `mag_ratio = mag_ratio_measured`; otherwise `mag_ratio = 1.0`. This avoids adding ±0.5%
  measurement noise to videos that were not re-zoomed.
- Per video, `scale_x = calib_scale_x / mag_ratio` and `scale_y = calib_scale_y / mag_ratio`.
- Apply this **in the conversion cell only**. The pixel measurements (`*_px` columns) don't
  change, so there is **no need to re-run detection or linking**: re-run from the conversion cell
  on.
- New `measurements.csv` columns, after `calib_source`: `band_height_px`, `mag_ratio_measured`,
  `mag_ratio`. The existing `scale_x_um_per_px` / `scale_y_um_per_px` columns now hold the
  per-video scale that was used.
- Print a per-video table of band height, measured ratio, applied ratio and the `magnification`
  note from videos.csv. Print a warning when the note and the measurement disagree, i.e. a note
  of "2x" with a ratio off by more than 2%, or a note other than "2x" with a ratio within 2%.
- QC overlays: use the per-video scale for µm labels from now on. Existing overlays don't need
  re-rendering.

## 3. Frame window from videos.csv (frame-extraction notebook)
Use `frame_start` / `frame_stop` from videos.csv when present, falling back to `FRAME_RANGES`.
evap_29 keeps 0–450 (the camera was bumped after that). No re-extraction is needed now. It only
matters for future runs.

## Acceptance
- Calibration band ≈ 621 ± 2 px.
- evap_29 ≈ 623 px (ratio ≈ 1.003, **not applied**).
- evap_30 ≈ 558 px (ratio ≈ 0.898, **applied**; scale_x ≈ 1.79 µm/px).
- evap_31 ≈ 627 px (ratio ≈ 1.009, **not applied**).
- evap_30's `D_eq_um` / `H_um` / `V_um` are ≈ 1.114× their previous values. All `*_px` columns are
  unchanged for every video. All µm columns are unchanged for every video whose ratio was not
  applied.
- Report the band height and ratio for every other video too (evap_21–38). Any ratio beyond 2% is
  worth flagging to the operator.
