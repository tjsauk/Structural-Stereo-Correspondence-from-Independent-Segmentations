The sourcecode has a GUI that allows the user to run a correctly formatted dataset.

# Dataset format

This document describes the dataset layout supported by **DPG Structural Stereo v1.0.0**. The program processes pairs of rectified stereo RGB images, optionally using camera calibration and depth ground truth (GT) for post-hoc evaluation.

**The dataset is not part of this GitHub repository.** Store it locally (outside the source repository, if possible) and select its folder or ZIP file in the application. The GUI and command-line tools accept either format.

## 1. Required stereo images

The simplest supported directory layout is:

```text
rig_name/
└── mav0/
    ├── cam0/
    │   └── data/
    │       ├── 1791365593052814610.png
    │       ├── 1791365593119442610.png
    │       └── ...
    └── cam1/
        └── data/
            ├── 1791365593052814610.png
            ├── 1791365593119442610.png
            └── ...
```

- `cam0` is the **left** camera; `cam1` is the **right** camera.
- RGB input files may be `.png`, `.jpg`, or `.jpeg`. They are read as three-channel RGB images.
- Both images in a pair must have the **same pixel dimensions**. The stereo geometry assumes the images have already been **rectified** (corresponding image points lie on the same image row).
- Without a `clip_manifest.json`, the program sorts left-image filenames and looks for an **identically named** file under `cam1/data/`. Only matched pairs are indexed.
- The `mav0/` level is optional for an unpacked directory; `rig_name/cam0/data/` and `rig_name/cam1/data/` are also accepted.
- Stereo pairs are numbered **0, 1, 2, ... in sorted filename order** when no manifest is present. These indices are used in the GUI; they are **not** image timestamps.

A ZIP containing the same relative layout is also accepted, including a ZIP with a single enclosing `rig_name/` directory. For ZIPs without a manifest, the reader similarly pairs files by identical filenames under `cam0/data/` and `cam1/data/`.

## 2. Recommended complete dataset layout

To enable calibrated stereo alignment and GT-based measurements, use:

```text
rig_name/
├── clip_manifest.json              # Recommended for a subset (see Section 4)
├── export_report.json              # Optional source documentation
├── mav0/
│   ├── cam0/
│   │   ├── sensor.yaml              # Camera calibration
│   │   ├── data.csv                 # Optional timestamp-to-filename index
│   │   └── data/<timestamp>.png     # Left RGB images
│   ├── cam1/
│   │   ├── sensor.yaml
│   │   ├── data.csv
│   │   └── data/<timestamp>.png     # Right RGB images
│   ├── depth0/
│   │   ├── sensor.yaml              # Optional GT sensor metadata
│   │   ├── data.csv                 # Optional GT file index
│   │   └── data/<timestamp>.png     # Left-camera GT depth
│   └── imu0/                        # Optional; not needed for single-pair stereo
│       ├── sensor.yaml
│       └── data.csv
└── lio_odom/                        # Optional auxiliary data
```

The stereo solver itself requires only the **two RGB images per pair**; valid calibration should be provided for meaningful geometric results. Depth GT is required **only for GT evaluation**. IMU and odometry files are not required for the single-frame stereo alignment workflows.

## 3. Calibration and depth ground truth

### Camera calibration

The program reads common metadata formats, including EuRoC-style `mav0/cam0/sensor.yaml` and `mav0/cam1/sensor.yaml`, and recognized JSON/YAML calibration files. For the `sensor.yaml` route, both camera entries must contain:

- `intrinsics: [fx, fy, cx, cy]` in pixels;
- `T_BS` as a 4-by-4 rigid transform, typically represented with 16 numbers in its `data` field.

The baseline is calculated from the two camera transforms. The program can also read rectified stereo projection matrices (`P1` and `P2`) or a recognized simple calibration object with `focal_px`, `baseline_m`, `cx` and `cy`. Preserve the dataset's original calibration files wherever possible.

**Important:** the solver assumes rectified input images. The mere presence of `sensor.yaml` does not prove that the RGB images have been rectified. Camera parameters must correspond to the images actually supplied.

An example of *simple calibration metadata* (use your dataset's actual values) is:

```json
{
  "focal_px": 536.2546,
  "baseline_m": 0.1197414,
  "cx": 622.7123,
  "cy": 341.6386,
  "rectified": true,
  "gt_kind": "depth_mm",
  "gt_side": "left"
}
```

Save such metadata as `calibration.json` in the dataset root if this is the calibration source being used. The numbers above illustrate the previously used ZED-style rig; **do not reuse them for a different camera**. If both cameras have fully specified `sensor.yaml` files, that calibration takes precedence over simple flat JSON fields in the current reader; for confirmed rectified calibration, recognized `P1`/`P2` matrices take precedence.

### Depth GT

For the ZED-style dataset used in this research:

- Store the **left-camera** depth images under `mav0/depth0/data/`.
- Use **single-channel 16-bit PNG** images containing depth in **millimetres** (`uint16`, `depth_mm`).
- `0` represents invalid or missing depth.
- Prefer the **same timestamp filename** as the corresponding `cam0/data/` RGB image.
- The depth image must have the **same width and height** as the stereo RGB images; the evaluator does not rescale it.
- The GT reader matches depth by the camera-image timestamp and can fall back to a nearby timestamp within **50 ms** when an exact match is absent. Exact filenames are preferable.

For example, an integer depth value of `2500` means **2.5 metres**. The code converts `depth_mm` to metres by multiplying by `0.001`.

Other supported GT array/image formats include `.tif`, `.tiff`, `.exr`, `.npy`, `.npz` and `.pfm`, provided that the field is a readable single-channel depth or disparity map. Declare nonstandard depth/disparity encodings in metadata; the code should not be expected to infer an unknown disparity scale.

GT is **never used to calculate segment correspondences or the stereo alignment**. It is used only afterward to measure the accuracy of right-image structures placed in left-camera coordinates.

## 4. Preserving original frame numbers in a subset

A dataset extracted from a larger recording should include `clip_manifest.json`. Without it, a clip containing original frames 200–600 would normally be re-indexed as **0–400** when opened.

The manifest links each original integer frame index to the actual timestamp-named RGB files. Example (only two rows shown):

```json
{
  "format": "structural_stereo_subset_v1",
  "frame_start": 200,
  "frame_end": 201,
  "frame_count": 2,
  "pair_source_ids": [
    {
      "pair_index": 200,
      "left": "mav0/cam0/data/1791365593052814610.png",
      "right": "mav0/cam1/data/1791365593052814610.png"
    },
    {
      "pair_index": 201,
      "left": "mav0/cam0/data/1791365593119442610.png",
      "right": "mav0/cam1/data/1791365593119442610.png"
    }
  ]
}
```

For the real **200–600** clip, provide **401** `pair_source_ids` entries with indexes from 200 through 600 and the actual original timestamp filenames. Do not rename the image files to `200.png`, `201.png`, etc. The manifest's `left` and `right` values are paths relative to the dataset root. The exporter generates this manifest automatically, so it normally should not be edited manually.

The program's subset command is:

```bash
python subset_dataset.py --dataset "/path/to/original/rig" --start 200 --end 600 --output "rig_frames_200_600.zip"
```

It preserves original RGB and depth file bytes, sensor metadata, calibration and documentation. It selects the relevant camera and depth images, filters compatible timestamped auxiliary sensor logs to the interval, and keeps unfilterable or unrecognized files unchanged. Therefore **some large auxiliary recordings may still be present** in the resulting ZIP.

The same functionality is available in the GUI as **Export dataset subset (frame range)...**. Both endpoints are inclusive.

## 5. Optional `data.csv` indexes

EuRoC-style camera/depth CSV files can retain the mapping from original timestamps to filenames:

```csv
#timestamp [ns],filename
1791365593052814610,1791365593052814610.png
1791365593119442610,1791365593119442610.png
```

The normal stereo reader indexes image files directly and does not require these CSV files to find pairs. The subset exporter, however, preserves and filters the corresponding indexes when they exist.

## 6. How to load the data

1. Keep the dataset directory or ZIP **outside the Git repository**, or otherwise exclude it from version control.
2. Open `START_WINDOWS.bat` and choose **Open folder** or **Open ZIP** in the dataset selector.
3. Set a single frame or an inclusive frame range (e.g. **200–600** if using a manifest-preserving clip).
4. Run a stereo workflow. If depth GT is present and readable, post-hoc accuracy measurements will be added automatically.

Command-line example for one frame:

```bash
python run_element_stereo.py --dataset "/path/to/rig_name" --frame 200 --output "results/frame_000200" --solver-mode experimental --segmentation-method dpg
```

The frame specified with `--frame` is the **index returned by the dataset reader**, not the timestamp in the image filename.

## 7. Common problems

| Symptom | Likely cause and action |
| --- | --- |
| `No stereo image pairs found` | Check that both `cam0/data/` and `cam1/data/` contain matching RGB filenames, or supply an accurate `clip_manifest.json`. |
| Selected frame 200 is missing in a clip | The clip was opened without an original-index manifest and its pairs were re-indexed from 0. |
| `GT frame ... not found` | Check `depth0/data/`, source timestamps, and the presence of a depth image matching each `cam0` RGB image. |
| GT image size mismatch | Supply native-resolution GT depth aligned with the stereo RGB images. |
| Large or implausible geometric error | Verify that the stereo images are rectified and that focal length, principal point and baseline refer to those images. |
| Dataset too large for GitHub | Do not commit the full dataset. Host it separately and provide a link, format description and any distribution restrictions in the repository README. |

**Code version documented:** DPG Structural Stereo GitHub release v1.0.0 (`src/dataset_io.py`, `src/dataset_metadata.py`, `src/gt_element_eval.py`, and `src/dataset_subset.py`).
