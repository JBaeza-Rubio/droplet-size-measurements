# INSTRUCTIONS C: QC display tweaks in `droplet_size_measurement_v2.ipynb`

**Display only.** Do not change anything that feeds `measurements.csv`. The detection, edge
finding, fits and linking all stay exactly as they are.
This applies to the qc_drops overlays too: labels, timestamps and drop_ids are drawn onto the
**figure** (matplotlib text and patches) after measuring. Never draw them into, or rescale, any
image array that `measure_drops` or the linking reads, and never reorder the measurement steps. After the changes, re-run the notebook and
confirm `measurements.csv` is byte-for-byte unchanged, apart from row order if that is not fixed.

## 1. Filmstrips (`results/qc_tracks/<video>/<drop_id>.png`): one shared brightness range
Right now each crop is contrast-stretched on its own min/max. A frame with a brighter glare spot
therefore looks darker overall, even though the raw background is constant. Checked on
evap_31_d0022, frames 775–795: the band mean is 36.0 in every frame.
- For each filmstrip, compute **one** `vmin`/`vmax` from all of that drop's crops together. Use
  the 1st and 99.5th percentiles of the pooled pixels, so a saturated glare spot doesn't set the
  scale. Apply it to every crop (`vmin=..., vmax=...`, `interpolation="nearest"`).
- Print the range used in the figure title, e.g. `display range 4–118 (shared)`.

## 2. Make frames easy to find in the video
Video time = `frame / fps`. fps is 18.005 for these videos (`3601/200`); read it from the file or
from `frames.csv`. Format it as `m:ss.ss`.
- **Filmstrips:** under each crop print `f<frame>`, `<m:ss.ss>`, `<D_eq px>`. In the title add the
  frame range and the video-time range, e.g. `evap_31_d0022  f775–795  0:43.04–0:44.15  21 frames`.
- **qc_drops overlays** (`results/qc_drops/<video>/<frame>.png`): put the video time in the
  title, and label each measured drop in the frame with its `drop_id` (short form, e.g. `d0022`),
  next to its box in the full view and in its zoom-panel title.
- Add a `video_time` column (`m:ss.ss`) to `measurements.csv`, placed right after `time_s`. This
  is the only CSV change: a new column with no change to any existing value.

## Acceptance
- evap_31_d0022's filmstrip: frames 783 and 793 no longer look darker than their neighbours. The
  glare spot is simply brighter.
- evap_31 frame 875 overlay shows `d0026` and `0:48.60`; frame 877 shows `d0029` and `0:48.71`.
- All existing columns of `measurements.csv` are unchanged.
