# TATARO LeRobot Dataset

Pick-and-place demonstrations for a **UFactory Lite6** arm with a **Gripper Lite**: pick a coloured cube anywhere on the table and drop it in a bowl. Format: [LeRobot](https://github.com/huggingface/lerobot) v3.0.

| dataset | episodes | frames | content |
|---|---|---|---|
| `tataro/record_cube001_20261106` | 77 | 53,297 | **Original recording** on the real arm, 2026-11-06: 58 blue, 10 red, 9 green cube |
| `tataro/record_cube001_20261106_all` | 193 | 132,727 | **Training set**: the 77 real episodes + 116 synthetic ones |

## How the data was recorded
Each episode starts with the arm at HOME. The operator places the cube, then hand-guides the open gripper around it in manual mode. The arm then runs the recorded motion with no hands in view:
pregrasp 150 mm above the cube → descend → close → lift → swing around the base → over the bowl → release → up.
The cube's colour is recorded in the task text, e.g. `Pick up the blue cube and place it in the bin`.

## Synthetic episodes (`_all` only)
The 58 blue-cube episodes were recoloured to red and to green by masking the cube in every video frame. The colour targets were measured on the real red and green cubes. The actions and state are identical to the blue source, so the colour counts in `_all` are 58 blue, 68 red, 67 green.

- Episodes 0–76 are the real recordings in their original order. Episodes 77–192 are synthetic.
- **Train/val split:** keep each blue episode together with its red and green copies, otherwise the trajectory leaks into validation. `split_train_val.py` in the TATARO firmware repo does this.
- Known artefacts: a thin cyan rim on some cube edges, and occasionally a small tinted patch on the bowl next to the cube.

## Features
| key | shape | content |
|---|---|---|
| `action` | 7 | absolute TCP target `x, y, z` [m], `wx, wy, wz` [rotation vector, rad, relative to gripper pointing down], `gripper` (0 open / 1 closed) |
| `observation.state` | 13 | `joint1..6` [rad], measured TCP pose (same 6 values as the action), `gripper` |
| `observation.images.wrist` | 480×640×3 | Intel RealSense D435i on the wrist, AV1 video |
| `observation.images.overhead` | 480×640×3 | Intel RealSense D455 above the table, AV1 video |

All data is recorded at 30 fps, and episodes are 20–28 s long. The TCP pose includes an 83.6 mm tool offset, so it is the fingertip point.

## Use
The videos and parquet files are stored with Git LFS, about 3 GB in total.
```bash
git lfs install
git clone https://github.com/Pooja15V/TATARO_LeRobot_Dataset.git
```
```python
from lerobot.datasets import LeRobotDataset
ds = LeRobotDataset("tataro/record_cube001_20261106_all",
                    root="TATARO_LeRobot_Dataset/tataro/record_cube001_20261106_all")
```
Train with `lerobot-train` using `--dataset.repo_id=tataro/record_cube001_20261106_all` and `--dataset.root=<that path>`.

Older datasets (`lite6_cube_traj`, `lite6_cube_traj_failures`, and the separate synthetic parts) were removed from the tree but are still in the git history.
