# Ridgeback + Franka Panda + D455 GUI Demo
| Complete simulated workspace and measurement targets. | D455 stereo camera mounted above the Panda hand. |
|---|---|
| ![demo-enviroment](demo1.png) | ![demo-D455 stereo camera](demo2.png) |


<p align="center">
  <strong>Drag an IK target, aim a wrist-mounted stereo camera, and capture auditable RGB + depth measurements in Isaac Sim.</strong>
</p>


## Demo result

| RGB: target ID and representative pixel | Depth: axial depth at the same pixel |
|---|---|
| ![Annotated left RGB](outputs/captures/2026-0730-1/rgb_left_annotated.png) | ![Annotated axial depth](outputs/captures/2026-0730-1/depth_preview_annotated.png) |

The included `2026-0730-1` reference capture reports eight visible targets from
near to far and excludes one detected floor region. The closest representative
surface is `0.594 m` from the camera; the wall surface is `3.370 m` away. The
capture completed with zero controller errors and left the USD unchanged.

[Open the compact result](outputs/captures/2026-0730-1/summary.json) ·
[Open the complete measurements](outputs/captures/2026-0730-1/measurements.json) ·
[Open capture provenance](outputs/captures/2026-0730-1/capture.json)

## What it demonstrates

- GUI manipulation of `/World/IKTarget` while simulation is running.
- Franka Panda inverse kinematics with a wrist-mounted Intel RealSense D455.
- Automatic capture IDs such as `2026-0730-1`, `2026-0730-2`, and so on.
- Left, right, and color RGB plus axial and radial depth arrays.
- Color-independent visible-surface grouping from depth discontinuities.
- Camera-space and world-space surface coordinates with near/median/far depth.
- Operator overlays that make every reported measurement visually auditable.

## What the D455 simulation represents

The wrist accessory is an Intel RealSense D455 geometry and camera model in
Isaac Sim. It contains separate left, right, and color camera prims and preserves
a measured left-to-right baseline of approximately `0.095 m`. The camera moves
with the Panda hand, so every capture records its runtime world position and its
right, up, and forward basis vectors.

This demo reads two RTX-rendered depth products from the left camera:

- `distance_to_image_plane` supplies axial depth along camera Z-forward and is
  saved as `depth_axial_left.npy`.
- `distance_to_camera` supplies radial range and is saved as
  `depth_radial_left.npy`.

The current pipeline therefore tests camera geometry, visibility, coordinate
conversion, and depth-surface reporting. It does **not** simulate the physical
D455 stereo-disparity algorithm, firmware processing, quantization, optical
noise, or material-dependent hardware failure modes. This distinction defines
which accuracy claims can be made from the reference capture.

## Requirements and installation

This release was tested in the Isaac Sim 6.0 GUI on Linux with an NVIDIA RTX
GPU. Install Isaac Sim using NVIDIA's
[Workstation Installation](https://docs.isaacsim.omniverse.nvidia.com/6.0.0/installation/install_workstation.html)
guide and run the supplied compatibility checker first. The scene references
Isaac Sim 6.0 assets, so allow asset access when opening it for the first time.

Clone the repository; no separate `pip install` is required because the scripts
run inside Isaac Sim and use its bundled Python packages and extensions.

```bash
git clone https://github.com/hayashimatsu/ridgeback-franka-d455-demo.git
cd ridgeback-franka-d455-demo
```

## Live GUI workflow

1. Open `scenes/ridgeback_franka_d455_demo.usd` in Isaac Sim.
2. Press **Play**.
3. Open `scripts/demo_start.py` in **Window > Script Editor** and press
   **Ctrl+Enter** once.
4. Select `/World/IKTarget` in the Stage tree. Move or rotate it in small,
   reachable increments and wait for the wrist camera to settle.
5. In the Script Editor console, capture the current view:

```python
demo_capture()
```

No filename is required and an earlier capture is never overwritten. An
optional note is supported when useful:

```python
demo_capture("front-inspection")
```

See [the demo runbook](docs/DEMO_RUNBOOK.md) for the complete operator checklist
and [the GUI object guide](docs/ADDING_OBJECTS_GUI.md) before adding scene or
wrist-mounted objects.

## System architecture

```mermaid
flowchart LR
    A[IKTarget in GUI] --> B[IK controller]
    B --> C[Franka arm joints]
    C --> D[Wrist-mounted D455]
    D --> E[RGB + axial/radial depth]
    E --> F[Depth-connected surfaces]
    F --> G[measurements.json]
    F --> H[summary.json]
    F --> I[RGB / depth / region overlays]
```

The RGB image labels each target as `T# (u,v)`. The depth image uses the same
representative pixel but displays `T# <depth> m`, where depth is axial camera
Z-forward distance. `summary.json` is a compact projection of the complete
`measurements.json`; it is not a second detector.

## Capture contents

```text
outputs/captures/<YYYY-MMDD-sequence>/
├── rgb_color.png, rgb_left.png, rgb_right.png
├── depth_axial_left.npy, depth_axial_right.npy
├── depth_radial_left.npy, depth_preview.png
├── depth_region_labels.npy
├── rgb_left_annotated.png
├── depth_preview_annotated.png
├── region_overlay.png
├── capture.json
├── measurements.json
└── summary.json
```

`measurements.json` records the left camera world pose, every visible surface's
camera/world coordinates, its image coverage, 3D visible bounds, and surface
depth distribution. Raw axial depth plus `depth_region_labels.npy` preserves
non-planar surface samples without embedding large point arrays in JSON. Known
floor regions remain available as raw evidence but are excluded from the target
list and operator overlays.

## Repository layout

```text
scenes/          Demo USD scene
scripts/         GUI bootstrap, IK controller, and D455 capture
docs/            Operator and object-maintenance guides
outputs/         One reviewed reference capture; new runtime captures are ignored
demo_manifest.yaml
```

## Measurement limits

### Coordinate and distance result

For capture `2026-0730-1`, the left camera world position is
`C_world = (0.657637, 0.360578, 1.171612) m`. The report separates four
positions that answer different questions:

- `C_world`: runtime world position of the left camera optical center.
- `S_camera`: measured representative surface point in camera right/up/forward
  coordinates.
- `S_world`: the same measured surface point transformed into world
  coordinates using the runtime camera pose.
- `U_world`: USD-authored world position of the object's center, attached only
  after depth detection as a GUI/oracle reference.

The camera directly measures a visible surface point. Its camera-to-surface
range is:

<br>

```math
D_{\text{surface}} = \Vert S_{\text{camera}} \Vert = \Vert S_{\text{world}} - C_{\text{world}} \Vert
```

<br>

The camera-to-center range is an additional USD/GUI calculation, not a direct
depth measurement:

<br>

```math
D_{\text{center}} = \Vert U_{\text{world}} - C_{\text{world}} \Vert
```

<br>

| Target · post-detection GUI label | `S_camera` measured surface XYZ (m) | `S_world` measured surface XYZ (m) | `D_surface` (m) | `U_world` USD center XYZ (m) | `D_center` (m) |
|---|---|---|---:|---|---:|
| T1 · red sphere | `(0.351, -0.222, 0.425)` | `(1.080, 0.012, 0.942)` | 0.594338 | `(1.091, 0.013, 0.942)` | 0.601265 |
| T2 · spring-green cube | `(0.076, -0.096, 0.617)` | `(1.273, 0.288, 1.063)` | 0.628880 | `(1.303, 0.288, 1.063)` | 0.658382 |
| T3 · rose sphere | `(0.437, 0.145, 0.890)` | `(1.553, -0.071, 1.300)` | 1.002391 | `(1.596, -0.078, 1.299)` | 1.043978 |
| T4 · yellow cube | `(-0.081, -0.480, 1.252)` | `(1.899, 0.448, 0.667)` | 1.343311 | `(1.975, 0.445, 0.658)` | 1.416179 |
| T5 · azure sphere | `(0.538, -0.788, 1.926)` | `(2.570, -0.168, 0.346)` | 2.149148 | `(2.596, -0.160, 0.354)` | 2.167250 |
| T6 · violet cube | `(-1.573, 0.819, 2.017)` | `(2.681, 1.947, 1.947)` | 2.685410 | `(2.731, 1.931, 1.934)` | 2.709834 |
| T7 · chartreuse sphere | `(0.108, 1.265, 2.781)` | `(3.463, 0.272, 2.381)` | 3.056607 | `(3.530, 0.265, 2.366)` | 3.112675 |
| T8 · wall | `(-0.377, 0.463, 3.317)` | `(3.981, 0.758, 1.568)` | 3.370186 | `(4.030, 0.725, 1.500)` | 3.408238 |

`D_surface` and `D_center` deliberately differ: one ends at the visible surface
sample and the other ends at the authored object center. Their difference is
not camera error; it also contains object radius or half-size, viewing
direction, and representative-point selection.

The coordinate transform itself is checked by comparing the two expressions
of the same surface range:

<br>

```math
\Vert S_{\text{camera}} \Vert = \Vert S_{\text{world}} - C_{\text{world}} \Vert
```

<br>

The coordinate-frame closure error is:

<br>

```math
e_{\text{closure}} = \left| \Vert S_{\text{camera}} \Vert - \Vert S_{\text{world}} - C_{\text{world}} \Vert \right|
```

<br>

The maximum closure error is approximately `0.000040 mm`. This demonstrates
numerical consistency between camera-space back-projection and runtime
camera-to-world transformation. It is **not** a physical D455 depth-accuracy
measurement. Target names in the table are attached only after depth detection
for GUI checking; they do not participate in detection.

### What is measured—and what is not

- Axial depth is the representative pixel's Z-forward value. It is not the
  oblique three-dimensional range unless the pixel lies on the optical axis.
- Camera-to-surface range is the Euclidean distance to one visible surface
  sample, never a distance to the hidden object center.
- `surface_point_to_authored_center_m` is not sensor error. It includes sphere
  radius or cube half-size, viewing direction, and representative-point
  selection. A wall representative point is especially unsuitable for a
  comparison with the center of the entire wall.
- A true simulated surface-accuracy test would cast the representative camera
  ray against the authored USD geometry and compare RTX depth with that visible
  surface intersection. That independent surface oracle is not part of the
  current release report.

Depth discontinuities identify visible surface regions, not semantic object
identity. Touching objects at nearly identical depth can merge, while one
occluded object can split into multiple regions. Reflective, transparent, or
invalid-depth surfaces require a different sensor or material-error strategy.
The reported representative point is a repeatable visible surface sample, never
an inferred hidden object center.
