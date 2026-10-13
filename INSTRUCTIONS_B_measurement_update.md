# INSTRUCTIONS B: `droplet_size_measurement.ipynb` → `droplet_size_measurement_v2.ipynb`

Read `CONTEXT.md` first (especially §3–§4). This document changes the existing notebook to
measure **backlit drops in frames produced by notebook A** (`INSTRUCTIONS_A_frame_extraction.md`).

**Save as a new file, `droplet_size_measurement_v2.ipynb`. Leave the original untouched.** It is
still needed for the v1 ring-light data. The overall structure (config, helpers, calibration,
drop functions, run, table and CSV, QC, tuning guide, acceptance checks) **stays the same**. Keep
the operator's cell-by-cell workflow and the existing style.

---

## 1. What stays as is
- **Cell 2 helpers** (`list_images`, `load_gray`, `largest_true_run`, `rotate_image`,
  `periodicity_score`, `fft_period`, `stretch_for_display`) stay unchanged.
- **Cells 3–4 calibration:** the algorithm is unchanged. It is validated on the new slides in
  CONTEXT §4a. Only the additions in §3 below.
- **QC philosophy:** square insets, `aspect="equal"`, `interpolation="nearest"`, and no
  resampling of drop images. Keep the existing comments that enforce this.
- **Per-axis scales:** X and Y stay separate everywhere.

## 2. Config cell changes

Folders:
```python
DATE_DIR            = r"...\Drop size measurements\2026.10.12"
FRAMES_SUBDIR       = "frames"          # NEW: output of notebook A (contains frames.csv)
CALIB_SUBDIR        = "calibration"
CALIB_OVERRIDE_DIR  = None
CALIB_EXCLUDE       = ("evap_45.jpg",)  # NEW: hand-rotated export, never use for scale
RESULTS_DIR_OVERRIDE = None
VIDEOS              = None              # NEW: None = all videos in frames.csv; or list of stems
```
`DROPS_SUBDIR` and `DROPS_DIR_OVERRIDE` are removed. Drop frames come from `frames.csv`.

Remove the v1 drop parameters: `TOPHAT_KERNEL`, `K_SIGMA`, `MIN_PEAK_SIGMA`, `CLOSE_ITER`,
`CLOSE_KERNEL`, `SATURATION`, `BLUR_SHARPNESS`.

Add the v2 drop parameters (validated defaults):
```python
# Detection (same meaning as notebook A; keep values in sync)
K_DETECT        = 6.0     # dev > K_DETECT*sigma marks drop pixels   (dev = background - frame)
MIN_DROP_AREA   = 15      # px
MAX_DROP_AREA   = 1500    # px
EDGE_MARGIN     = 30      # px from left/right frame edge; band edges handled via band_top/bottom

# Edge + fit (CONTEXT 4c)
EDGE_FRACTION   = 0.5     # edge where dev falls to this fraction of the core depth (half-depth)
CORE_PCTL       = 90      # core depth = this percentile of dev inside the blob
N_RAYS          = 72      # rays from the centroid (every 5 deg)
RAY_MAX_PX      = 25      # ray length; > max expected radius (~10 px)
RAY_STEP_PX     = 0.1     # sampling step along a ray (bilinear interpolation)
GLARE_K         = 4.0     # ray excluded if frame is brighter than bkg by > GLARE_K*sigma on it
GLARE_PAST_EDGE = 2.5     # px beyond the edge still checked for glare on that ray
GLARE_SECTOR_DEG = 0      # optional: also exclude rays within +-this many deg of horizontal (0 = off)
MIN_EDGE_POINTS = 12      # fewer usable rays => fit unreliable, drop flagged `fit_failed`
OUTLIER_MAD_K   = 3.0     # one round of residual-outlier rejection in the circle fit
ASPECT_FLAG     = 0.10    # flag when |H - V| / mean_HV exceeds this
SHARPNESS_FLAG  = None    # soft blur flag threshold (tune on QC; None = report value only)
DARK_CLIP_LEVEL = 0       # raw pixel <= this inside the blob => `dark_clipped` flag

# drop_id linking (CONTEXT 4d)
LINK_MAX_GAP    = 3       # frames
LINK_MAX_DIST   = 80      # px centroid displacement
LINK_MAX_GROWTH = 1.30    # D_eq may not grow by more than 30% within one drop_id
```
`QC_VIEW_CROP`: default to `None`, meaning use each frame's `band_top/band_bottom` from
`frames.csv` as the view rows. Keep the manual override.

## 3. Calibration additions (cell 3/4)
1. Apply `CALIB_EXCLUDE` (filename match) before measuring. Print what was excluded.
2. **Duplicate guard:** hash each calibration file (`hashlib.md5`). Measure byte-identical files
   once, and print a warning naming them. `evap_42.jpg` and `evap_43.jpg` are identical.
3. **Investigate (don't hide) the X-axis rejections** on `evap_41/42/43.jpg` (CONTEXT §4a).
   Look at the QC overlay and the column profile, and check whether the fibre crossing the grid,
   the printed labels, or the ROI column bounds are responsible. If a small, principled ROI or
   prominence fix makes them pass with RMS < 0.5 px and values consistent with evap_44/46, apply
   it. If not, leave them rejected. Either way, report in a markdown note what the cause was.
   **Don't loosen `MAX_FIT_RMS_PX`** to let them through.

## 4. Drop functions (replace cell 5)

### 4.1 Inputs
`load_frame_and_bg(row)` reads the frame PNG and its `bg_png` from `frames.csv` (paths relative
to `frames/`) as float32. `band_top/band_bottom` come from the same row.

### 4.2 `measure_drops(row) -> (drops, diag)`
1. `dev = bg − frame`. `sigma = 1.4826·MAD(dev)` over the band rows.
2. Detection mask exactly as notebook A §4 step 3. Connected components with the area and
   edge filters.
3. For each component:
   - centroid `(cx, cy)` (mean of component pixels); `depth = percentile(dev[comp], CORE_PCTL)`;
     `level = EDGE_FRACTION · depth`.
   - **Rays:** for each angle θ in `N_RAYS`, sample `dev` along the ray with
     `scipy.ndimage.map_coordinates(order=1)` at `r = 0 … RAY_MAX_PX` step `RAY_STEP_PX`. Find the
     **first** sample with `r > 1.5` and `dev < level`, and interpolate linearly between it and
     the previous sample for a sub-pixel radius `R(θ)`. No crossing gives NaN.
   - **Glare test per ray:** mark the ray as glare if any sample from `r = 0` to
     `R(θ) + GLARE_PAST_EDGE` has `dev < −GLARE_K·sigma` (frame brighter than background). If
     `GLARE_SECTOR_DEG > 0`, also mark rays within that angle of horizontal.
   - **Usable rays:** not glare, finite, and `R ≤ 2·median(R)`. If fewer than `MIN_EDGE_POINTS`,
     set `fit_failed = True` and still emit the row with the measurements set to NaN.
   - **Circle fit** (algebraic least squares on the edge points). Then drop points whose radial
     residual exceeds `OUTLIER_MAD_K·1.4826·MAD(residuals)`, and refit once.
     `D_eq_px = 2·radius`.
   - **Ellipse fit** (`cv2.fitEllipse`) on the same final points. Report its axis-aligned
     extents: `H_px = 2·sqrt((a·cosφ)² + (b·sinφ)²)`, `V_px = 2·sqrt((a·sinφ)² + (b·cosφ)²)`,
     with a, b the semi-axes and φ the angle. Read the convention via `cv2.boxPoints` as v1's
     Diagnostic 4 did, since OpenCV's width/height order is not guaranteed. Also keep
     `ellipse_major_px`, `ellipse_minor_px` and `ellipse_angle_deg`.
   - **Fit quality:** `fit_rms_px` (RMS radial residual of the circle fit), `n_edge_points`,
     `n_glare_rays`.
   - **Flags:** `aspect_flag` (|H−V|/mean_HV > ASPECT_FLAG), `dark_clipped` (raw ≤
     DARK_CLIP_LEVEL inside the blob), `sharpness` as in notebook A, and `blurry` (if
     `SHARPNESS_FLAG` is set).
4. Return the per-drop dicts sorted by x. `diag` carries `dev`, `sigma`, the band, the per-drop
   ray points (used/excluded) and the fits, for QC.

### 4.3 QC overlay (`save_drop_qc`, adapted)
Keep the layout: band view on top, square zoom insets below. In each inset show:

- the raw frame, contrast-stretched;
- **used edge points** (small dots, one colour) and **excluded/glare points** (another colour,
  e.g. x);
- the fitted circle (solid) and ellipse (dashed);
- a title with `drop_id` (once linked), `D_eq`, H, V in px and µm, `n_edge_points`, and flags.

## 5. Run cell (replace cell 6)
Loop over `frames.csv` rows (saved frames only, filtered by `VIDEOS`), call `measure_drops`,
save the QC overlay to `results/qc_drops/<video>/<png stem>.png`, and collect rows. A frame with
0 drops after measurement emits an `n_drops = 0` row, as in v1. Print one line per frame, as v1
does.

## 6. NEW cell: link detections into `drop_id`
Per video, process frames in order. Keep a list of **open tracks** (last frame, last centroid,
last D_eq). For each detection in the current frame, greedily (nearest first) attach it to an
open track if:

- `frame − last_frame ≤ LINK_MAX_GAP`,
- distance ≤ `LINK_MAX_DIST`,
- and `D_eq ≤ LINK_MAX_GROWTH · last_D_eq`.

Otherwise start a new track. Close tracks that have exceeded the gap. IDs look like
`evap_30_d0008`. Add:

- `drop_id`,
- `n_frames_in_drop`,
- `t_rel_s` = `time_s` − the first `time_s` of that drop,
- `frame_in_drop` (0, 1, 2, …).

Save `results/qc_tracks/<video>/<drop_id>.png`: a filmstrip of the drop's frames (square crops,
same size, nearest interpolation) with D_eq printed under each, plus a tiny D_eq-vs-frame line.
This is a QC figure only. The real size-vs-time analysis is a later script.

## 7. Conversion and results table (adapt cell 7)
- `D_eq_um = D_eq_px · sqrt(scale_x · scale_y)` (geometric mean of the two axis scales).
- `H_um = H_px · scale_x`, `V_um = V_px · scale_y`, `mean_HV_um = (H_um + V_um)/2`,
  `aspect = H_um / V_um`.

`results/measurements.csv` columns (exact order):

| column | meaning |
|---|---|
| `video`, `frame`, `time_s`, `png` | from frames.csv |
| `drop_id`, `frame_in_drop`, `t_rel_s`, `n_frames_in_drop` | from linking |
| `drop_index`, `n_drops` | index within the frame / count in the frame |
| `centroid_x`, `centroid_y`, `area_px` | blob |
| `depth`, `sigma`, `depth_sigma` | core darkness, noise, and their ratio (contrast/noise) |
| `D_eq_px`, `H_px`, `V_px` | fitted sizes (sub-pixel) |
| `ellipse_major_px`, `ellipse_minor_px`, `ellipse_angle_deg` | ellipse fit |
| `D_eq_um`, `H_um`, `V_um`, `mean_HV_um`, `aspect` | converted |
| `n_edge_points`, `n_glare_rays`, `fit_rms_px` | fit quality |
| `aspect_flag`, `dark_clipped`, `sharpness`, `blurry`, `fit_failed` | flags |
| `scale_x_um_per_px`, `scale_y_um_per_px`, `calib_source` | calibration provenance |
| `edge_fraction`, `k_detect`, `glare_k`, `glare_sector_deg` | parameter provenance |

Keep the **aspect table** the operator likes, the v1 `diagnostics_aspect.csv`-style display, as
an inline display after the CSV is written. Show `video, frame, drop_id, D_eq_px, H_px, V_px,
aspect, n_glare_rays, flags` for the first ~20 rows, plus a summary: median and IQR of `aspect`
per video.

## 8. Tuning guide (rewrite the markdown cell)
- **`EDGE_FRACTION`** is the edge definition. Hold it at 0.5 for everything that will be
  compared. Don't tune it per video, since it is already gain-independent.
- Noise picked up as drops: raise `K_DETECT` or `MIN_DROP_AREA`.
- Small late-life drops missed: lower `K_DETECT` slightly (≥4). Check QC for false hits.
- `fit_failed` often, with many glare rays: lower `GLARE_K` a little or use `GLARE_SECTOR_DEG`.
  Expect this only while the laser is on.
- Wrong `drop_id` splits or merges: adjust `LINK_MAX_DIST` / `LINK_MAX_GAP`, and check the
  filmstrips.
- Always confirm on the QC overlays and filmstrips before trusting a batch.

## 9. Retire the v1 diagnostic cells
Cells 12–15 of v1 (raw-vs-flattened aspect, display audit, ellipse orientation) are about
top-hat artifacts that v2 doesn't have. **Remove them from v2.** They stay in the v1 notebook.
Keep, as a single small cell, the **no-resampling audit** (v1 check 3): nothing in the drop
path may rotate, warp or resize.

## 10. Acceptance checks (replace cell 11)
1. **Calibration** (`evap_44.jpg`, `evap_46.jpg`): X ≈ 62.2 px/100 µm (≈1.608 µm/px),
   Y ≈ 61.2–61.5 px/100 µm (≈1.63 µm/px), RMS < 0.5 px. evap_45 excluded; evap_42/43 counted
   once. Median X within 0.5% of the operator's hand value 1.603 µm/px.
2. **One drop, evap_30 frames 55–74:**
   - all 20 frames carry the **same `drop_id`**;
   - `D_eq_px` is within ±1.5 px of the reference table in CONTEXT §7 at frames 55, 60, 65, 70, 74;
   - first-to-last D_eq falls from ≈15.7 to ≈7.4 px;
   - no `fit_failed`.
3. **Whole video:**
   - evap_30 yields 30–40 drop_ids with ≥8 frames, and evap_31 yields 24–34;
   - median `aspect` is 0.78–0.90 on both (laser-on glare, expected; do **not** tune to force 1.0);
   - no `D_eq_px` above 40 px survives;
   - the static dust at (535, 708) never appears.
4. Every saved frame has a QC overlay, and every drop_id has a filmstrip.

## 11. Error handling
- No `frames/frames.csv`: a clear error telling the operator to run notebook A first.
- Missing PNG or background PNG: skip that row with a warning, continue.
- No usable calibration image: error (never a default scale).
