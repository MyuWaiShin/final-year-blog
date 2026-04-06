---
title: "Stage 2: Navigate"
date: 2026-03-23 12:00:00 +0000
categories: [Project Updates, Pipeline]
tags: [navigation, aruco, yolo, pnp, stereo depth, urscript, transform chain, obb]
author: myuwaishin
pin: false
last_modified_at: false
---

> **Note:** This post is written in April, backdated to the session date of 23 March.
{: .prompt-info }


After `explore.py` sweeps the base joint to find the object and roughly centre it in frame, the pipeline enters Stage 2: `navigate.py`. In this stage, the TCP moves to hover directly above the object, aligned and ready to descend.

Now that I have a working YOLO detection model, the next task is using it to navigate to the object. By this point it is already week 10, so implementing full OBB-based pose estimation from detection alone is not feasible in the time left. However, I can still demonstrate a proof of concept for gripper orientation for grasp maximisation, which is part of my original proposal, using the ArUco tag on the object.

The approach uses a fallback hierarchy: once the object is found during the sweep, the pipeline first looks for the ArUco tag. If the tag is visible, it uses PnP pose estimation, getting full 6-DOF pose including orientation, which lets the gripper align its approach angle to the object. If the tag is not visible, it falls back to YOLO detection combined with stereo depth to get a 3D position without orientation. The full transform chain, centring correction, and clearance height reference are the same in both modes (see below).

Find my navigation script here: [navigate.py](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/full_pipeline/navigate.py)
---

## What the Camera Sees

`navigate.py` opens two simultaneous streams from the OAK-D camera:

- RGB video at 1280×720, with manual focus locked to value 46 (calibrated for the working distance above the table)
- Stereo depth from the OAK-D's left and right mono cameras, aligned to the RGB frame

Both streams run continuously from the moment navigation starts. For every frame, two detectors run in priority order.

---

## Detection Hierarchy: ArUco First, YOLO Second

### Priority 1: ArUco Tag

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/aruco_tag_13.jpeg" style="width:40%; border-radius:6px; display:block; margin:0 auto;" alt="ArUco Tag">
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>ArUco Tag 13</em></figcaption>
</figure>

The pipeline expects an ArUco tag (dictionary `DICT_6X6_250`, ID 13, physical size 36 mm) placed on the bottom of the object. On every frame:

```python
corners, ids, _ = detector.detectMarkers(grey_frame)
```

If tag ID 13 is found, `estimatePoseSingleMarkers()` runs PnP (Perspective-n-Point) using the known 36 mm physical size and the detected 2D pixel corners. This gives an `rvec` (rotation) and `tvec` (translation) which represents the tag's full 6-DOF pose relative to the camera.

The 3D position is then transformed into the robot base frame through the full transform chain:

```
T_tag2cam     ← solvePnP  (tag relative to camera)
    ×
T_cam2flange  ← hand-eye calibration (camera relative to flange)
    ×
T_tcp2base    ← robot live FK (flange relative to robot base + TCP offset set in the settings)
     =
T_tag2base    ← tag position in robot world coordinates
```

`T_cam2flange` was computed once using the Tsai-Lenz hand-eye calibration algorithm and is loaded from [`calibration/T_cam2flange.npy`](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/full_pipeline/calibration/T_cam2flange.npy).

`T_tag2base[:3, 3]` gives tag XYZ in metres in the robot base frame, which becomes the hover target, just above the tag.

### Priority 2: YOLO + Stereo Depth

If the ArUco tag is not visible (obscured, poor lighting, or tag not present), the system falls back to YOLO object detection. `yolo26n_detect_V1` runs on every frame at 640 px resolution with a confidence threshold of 80%.

The highest-confidence bounding box gives a pixel centre `(px_cx, px_cy)`. To get a 3D position, the OAK-D stereo depth map is sampled at that pixel. The median depth over an 8-pixel radius patch is taken to reduce noise:

```python
depth_m = median(depth_frame[patch]) / 1000.0   # mm → m
```

This depth deprojets the pixel into 3D camera space using the camera intrinsics matrix `K`, and the same transform chain (camera → flange → base) gives the world-frame position.

YOLO has no knowledge of physical object size, so it cannot compute distance on its own, so stereo depth is used to get the distance. And ArUco does not need stereo depth at all, the known 36 mm marker size is enough to recover full 3D pose from geometry alone.

---

## Orientation: Aligning the Gripper to the Tag

ArUco tag gives both position `tvec` and orientation `rvec`. The code uses `rvec` to align the gripper's yaw (rotation around the vertical Z-axis) to the ArUco tag's X-axis.

**Step 1: gripper pointing straight down**

```python
R_hover_baseline, _ = cv2.Rodrigues(np.array([2.225, 2.170, 0.022]))
```

This Rodrigues vector encodes the rotation that points the gripper's Z-axis straight down toward the table. I got this rotation matrix from the teach pendant when the gripper was pointing straight down.

**Step 2: Extract the tag's yaw**

```python
x_tag   = R_tag_base[:, 0]                   # tag X-axis in base frame
yaw_tag = np.arctan2(x_tag[1], x_tag[0])    # angle in horizontal XY plane
```

**Step 3: Pick the smallest equivalent rotation**

```python
delta_a = wrap(yaw_tag - yaw_base)    # raw angle difference
delta_b = wrap(delta_a + π)           # same grip, rotated 180°
chosen  = whichever has smaller absolute value
```

Because the RG2 gripper is symmetric, rotating it 0° and rotating it 180° produce an identical grip. This means for any target angle, there are always two equivalent solutions separated by 180°. The code computes both and picks whichever requires the smaller wrist rotation.

**Step 4: Apply yaw on top of baseline**

```python
R_z   = rotation_matrix_around_Z(chosen)
R_tgt = R_z @ R_hover_baseline
rvec, _ = cv2.Rodrigues(R_tgt)
```

The gripper still points straight down, but its opening direction is now aligned to the ArUco tag's X-axis. If the tag is rotated 45° on the table, the gripper approaches at 45° too, maximising grip surface contact.

For YOLO-only mode, there is no orientation information, so the fixed baseline straight-down pose is used.

---

## The Hover Move

Once triggered, a single `movel` (linear Cartesian move) is sent as raw URScript to the robot over port 30002:

```
movel(p[tag_X, tag_Y, tag_Z, rx, ry, rz], a=0.05, v=0.09)
```

The robot moves at 90 mm/s. Python polls the live TCP pose every 10 ms until within 3 mm of target (no RTDE).

---

## Post-Hover Centring Correction

After arriving at hover, the TCP may still be a few millimetres off due to residual calibration error. A correction loop runs up to 3 times.

**ArUco path (no stereo depth needed):**

PnP is re-run on a fresh frame after the hover move. The tag's pixel X centre is compared to the image centre. The pixel offset is converted to metres using the PnP-derived camera-to-tag distance (`tvec[2]`) and focal length:

```python
delta_metres = offset_px * tvec[2] / focal_length_fx
```

This is converted from camera frame into base frame and added to the current TCP position:

```python
delta_base = R_cam2base @ [delta_metres, 0, 0]
new_x = tcp_x + delta_base[0]
new_y = tcp_y + delta_base[1]
```

A `movel` shifts the TCP horizontally. This repeats until the tag is within ±40 px of frame centre, or 3 iterations are exhausted.

**YOLO path:**

The same formula applies here, but `tvec[2]` is replaced by the OAK-D median depth, since YOLO has no inherent geometry.

---

## Navigation Demo

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <div style="width:80%; position:relative; padding-bottom:45%; height:0; overflow:hidden; border-radius:8px;">
    <iframe src="https://www.youtube.com/embed/AYuxPyeHzB0"
      style="position:absolute; top:0; left:0; width:100%; height:100%; border:0;"
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
      allowfullscreen>
    </iframe>
  </div>
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Navigation: hovering TCP above the tag and center correction</em></figcaption>
</figure>

---

## What Navigate Returns

```python
{
    "hover_pose":     [x, y, hover_z, rx, ry, rz],   # where the robot is now
    "clearance_z":    hover_z + 0.40,                # safe transit height
    "detection_mode": "aruco" | "yolo_only"          # for logging
}
```

---

## Next

When the robot is already at `hover_z`, the next stage is to descend, close the gripper, and verify contact through width and force checks.



