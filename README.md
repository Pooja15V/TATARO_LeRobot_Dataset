# TATARO LeRobot Dataset

Pick-and-place demonstrations for a **UFactory Lite6** arm with a **Gripper Lite**: pick a coloured cube from the table and drop it in a bowl. Format: [LeRobot](https://github.com/huggingface/lerobot) v3.0.

| dataset | episodes | frames | content |
|---|---|---|---|
| `tataro/record_cube001_20261106` | 77 | 53,297 | **Original recording** on the real arm, 2026-11-06: 58 blue, 10 red, 9 green cube |
| `tataro/record_cube001_20261106_all` | 193 | 132,727 | v1 training set: the 77 real episodes + 116 synthetic ones |
| `tataro/record_zone001_20261110` | 50 | 33,925 | **Zone recording** on the real arm, 2026-11-10: cube only inside a 250×250 mm square, centre (280, −60) mm in the robot frame (x 155–405, y −185…65). 24 green, 21 red, 5 blue |
| **`tataro/record_tataro_v2_table`** | 243 | 166,652 | **Current training set (v2)**: zone + `_all`, with the overhead camera replaced by a 336×336 top-down **table view** |

## How the data was recorded
Each episode starts with the arm at HOME. The operator places the cube, then hand-guides the open gripper around it in manual mode. The arm then runs the recorded motion with no hands in view:
pregrasp 150 mm above the cube → descend → close → lift → swing around the base → over the bowl → release → up.
The cube's colour is recorded in the task text, e.g. `Pick up the blue cube and place it in the bin`.

## Why the zone recording and the table view (v2)
An ACT policy trained on `_all` missed every real pick by 6–16 cm. There were three causes:
- **Too few demos for the area:** 77 real demos spread over the whole table.
- **Background clutter:** objects on the desk next to the table were visible in the overhead image.
- **Moved camera:** the overhead camera shifted about 10 px after the recording.

v2 addresses these as follows:
- **Denser data:** 50 more demos, all inside a smaller zone.
- **Table view:** every overhead frame is warped to a 336×336 top-down view of the table only (about 2.5 mm/px). The warp is registered on ArUco markers (DICT_7X7_250) at the table corners: id 52 top-left, 51 top-right, 53 bottom-right; the fourth corner is extrapolated. Markers are detected once per episode, from its first frame. The view no longer shows the desk, and a small camera shift no longer changes the image.

At run time the live overhead image must be warped the same way (`table_warp.py` / `run_policy.py` in the TATARO firmware repo).

`record_tataro_v2_table` layout:
- **Episodes 0–49:** the zone recording.
- **Episodes 50–242:** `_all`, also in table view. Episodes 50–126 are its real recordings and 127–242 its synthetic copies.

The wrist camera is unchanged (480×640).

## Synthetic episodes (`_all` and v2)
The 58 blue-cube episodes were recoloured to red and to green by masking the cube in every video frame. The colour targets were measured on the real red and green cubes. The actions and state are identical to the blue source, so the colour counts in `_all` are 58 blue, 68 red, 67 green (v2: 63 blue, 89 red, 91 green).

- In `_all`, episodes 0–76 are the real recordings in their original order, and episodes 77–192 are synthetic.
- **Train/val split:** keep each blue episode together with its red and green copies, otherwise the trajectory leaks into validation. `split_train_val.py` in the TATARO firmware repo does this.
- Known artefacts: a thin cyan rim on some cube edges, and occasionally a small tinted patch on the bowl next to the cube.

## Features
| key | shape | content |
|---|---|---|
| `action` | 7 | absolute TCP target `x, y, z` [m], `wx, wy, wz` [rotation vector, rad, relative to gripper pointing down], `gripper` (0 open / 1 closed) |
| `observation.state` | 13 | `joint1..6` [rad], measured TCP pose (same 6 values as the action), `gripper` |
| `observation.images.wrist` | 480×640×3 | Intel RealSense D435i on the wrist, AV1 video |
| `observation.images.overhead` | 480×640×3 (v2: 336×336×3 table view) | Intel RealSense D455 above the table, AV1 video |

All data is recorded at 30 fps, and episodes are 20–28 s long. The TCP pose includes an 83.6 mm tool offset, so it is the fingertip point.

## Use
The videos and parquet files are stored with Git LFS, about 4 GB in total.
```bash
git lfs install
git clone https://github.com/Pooja15V/TATARO_LeRobot_Dataset.git
```
```python
from lerobot.datasets import LeRobotDataset
ds = LeRobotDataset("tataro/record_tataro_v2_table",
                    root="TATARO_LeRobot_Dataset/tataro/record_tataro_v2_table")
```
Train with `lerobot-train` using `--dataset.repo_id=tataro/record_tataro_v2_table` and `--dataset.root=<that path>`.
The v2 run used colour-only image augmentation: brightness, contrast, saturation and sharpness. LeRobot's default random shift was not used, because it moves the image by about 4 cm, which conflicts with the absolute-position actions. Hue changes were also left out, to keep the cube colours.

Older datasets (`lite6_cube_traj`, `lite6_cube_traj_failures`, and the separate synthetic parts) were removed from the tree but are still in the git history. The intermediate table-view sets (`record_cube001_20261106_all_table`, `record_zone001_20261110_table`) and the train/val splits are not stored. Rebuild them with `convert_table_view.py` and `split_train_val.py`.
