---
title: "Pipeline Planning and Stage 1: Object Search Sweep"
date: 2026-03-18 12:00:00 +0000
categories: [Project Updates, Pipeline]
tags: [yolo, sweep, explore, pipeline, detection]
author: myuwaishin
pin: false
last_modified_at: false
---

> **Note:** This post is written in April, backdated to the session date of 18 March.
{: .prompt-info }


## Pipeline Plan

The full pipeline is broken into six stages:

| Stage | Script | Description |
|---|---|---|
| 1 | `explore.py` | Sweep and find object in workspace |
| 2 | `navigate.py` | Hover TCP precisely above object using ArUco PnP |
| 3 | `grasp.py` | Descend, close gripper, width + force check |
| 4 | `verify.py` | Lift, YOLO + CLIP visual verification |
| 5 | `transit.py` | Carry to drop zone, slip monitor |
| 6 | `release.py` | Open gripper and release object |

A top-level `main.py` runs the stages in sequence and handles the retry loop. If grasp or verify fails, the robot recovers and navigates again from Stage 2.

Find my full pipeline code on [GitHub](https://github.com/MyuWaiShin/Final_Year_Project_2026/tree/main/full_pipeline).

---

## Skipped Using YOLO Object Detection

The YOLO object detection model I trained has low confidence on the objects I have been using. The objects are mostly uniformly coloured blocks, which makes them hard to distinguish from the background. The labelling used during training was not accurate enough either. When I tested the model, the detection either misses the object, picks up the wrong region, or returns a low confidence score.

For the final pipeline, I need the detection to be reliable for the robot to navigate to the object. This is a proof of concept for navigation, detection is not a priority for demonstrating failure detection and recovery. If I get time later, although I am running out of time, I most likely will retrain with better labels and swap out the objects.

---

## Stage 1: explore.py

> Script: `full_pipeline/explore.py`
{: .prompt-info }

`explore.py` finds the object on the table and positions the camera so the object is roughly centred in frame. It does not compute the 3-D pose, that is handled by the next stage `navigate.py`. It only does aruco/object search and hands off to `navigate.py` once the object is found.

### Infrastructure

Two persistent sockets run throughout:

**`RobotStateReader`** — daemon thread connected to port 30002 that continuously parses the robot's secondary-client binary stream. It extracts:

| Sub-packet type | What it extracts | Used for |
|---|---|---|
| Type 1 (Joint Data) | 6 joint angles (radians) | Arrival detection for joint moves |
| Type 2 (Tool Data) | AI2 voltage → gripper width mm | Checking gripper is open |
| Type 4 (Cartesian Info) | TCP pose [x,y,z,rx,ry,rz] | Arrival detection for Cartesian moves |

Arrival detection works by polling the live TCP pose from the type-4 sub-packet the robot sends in a loop and checking the distance to the target. There is no `wait()` or blocking call, which means there's no extra delays between steps. The motion command is sent and immediately the script starts polling:

```python
sender.send(f"movel(p[{x},{y},{z},...],a=...,v=...)")
while time.time() < deadline:
    cur  = state.get_tcp_pose()
    dist = sqrt((cur[0]-x)**2 + (cur[1]-y)**2 + (cur[2]-z)**2)
    if dist < 0.005:   # within 5mm
        return
    time.sleep(0.01)
```

**`URScriptSender`** sends motion commands to port 30002, write-only. Port 30002 pushes ~125 Hz state data back to every connected socket whether it is used for receiving or not. If that data is never read, the OS receive buffer fills in under a second and `sendall()` blocks, but a background drain thread used to prevent this:

```python
def _drain(self):
    while True:
        try:
            self._sock.recv(4096)   # read and discard
        except Exception:
            time.sleep(0.01)
```

### Configuration

These are the parameters that control the behaviour of the sweep:

```python
SWEEP_START_RAD   = -0.5    # J0 sweeps from -0.5 rad...
SWEEP_END_RAD     =  0.5    # ...to +0.5 rad
SWEEP_SPEED       =  0.2    # rad/s (slow enough for camera to catch it)
CONF_THRESHOLD    = 0.80    # YOLO must be ≥80% confident
CENTER_TOL_PX     = 80      # ±80px from frame centre = "found"
RECENTER_TOL_PX   = 20      # tighter ±20px for active correction
RECENTER_GAIN     = 0.0008  # J0 radians per pixel of error
RECENTER_MAX_ITER = 6       # max correction moves
SCAN_MAX_SWEEPS   = 2       # try twice before giving up
RETRY_LOWER_M     = 0.10    # descend 10cm on second sweep
```

### Sequence

**Step 1: Move to scan pose**

The robot loads `data/scan_pose.json` (a pre-recorded overhead joint configuration) and moves there. The gripper opens if it is not already open.

```python
movej_joints(sender, state, SCAN_JOINT_POS, APPROACH_SPEED, APPROACH_ACCEL)
open_gripper(ROBOT_IP, state)   # skips if already open
```

**Step 2: Sweep**

J0 (base joint) sweeps ±0.5 rad across the workspace. Only J0 is moved because rotating the base joint gives the most XY horizontal coverage per radian of movement compared to any other joint.

The sweep runs in a background thread while the main thread reads camera frames and runs detection. The initial version used ArUco detection only.

*After training YOLO detection, the sweep could find the object directly without a tag, which works better to identify the object in a workspace because the tag is sometimes not visible due to light glare.*

```python
def _sweep():
    for waypoint in (sweep_end, sweep_start):   # forward then back
        if result["centred"]:
            break
        movej_joints(..., stop_check=lambda: result["centred"])
```

As soon as a detection fires, a hard stop is sent immediately:

```python
result["centred"] = True
stopj(sender, accel=0.8)
```

The sweep thread sees `stop_check()` return True and exits. The arm brakes in place.

**Step 3: Active recentering**

After stopping, the object may not be centred. Recentering matters because PnP pose estimation accuracy increases when the object is observed near the optical centre of the image, where lens distortion is lowest.

The correction loop adjusts only J0 (base joint) up to 6 times, leaving all other joints unchanged:

```python
offset_px = pixel_x - frame_cx         # pixels from frame centre
delta_j0  = -offset_px * RECENTER_GAIN # 0.0008 rad per pixel
joints[0] += delta_j0
```

Only J0 is adjusted here for the same reason: rotating the base joint shifts the camera horizontally without changing height or orientation.

**Step 4: Retry with descent**

If nothing is found after the first sweep, the robot descends 10cm closer to the table before sweeping again. The descent is camera-compensated. It projects the camera's optical axis down to table level, then adjusts the TCP X/Y so the camera is still looking at the same region after descending:

```python
def _compute_lower_tcp(scan_tcp, T_cam2flange, lower_m, table_z):
    # Project camera ray to table → get look_x, look_y
    # Compute new TCP X/Y so camera still points at the same spot after descent
```

This uses `T_cam2flange` (from hand-eye calibration) to know where the camera is relative to the flange.

**Step 5: Return**

```python
return True    # object found and centred → navigate.py takes over
return None    # not found after 2 sweeps → pipeline aborts
```

`explore.py` does not return any coordinates. `navigate.py` re-detects the object from scratch when it starts and computes the 3-D pose.

---

## Demo

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <div style="width:90%; position:relative; padding-bottom:56.25%; height:0; overflow:hidden; border-radius:8px;">
    <iframe src="https://www.youtube.com/embed/TKqfiEFnv6Y"
      style="position:absolute; top:0; left:0; width:100%; height:100%; border:0;"
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
      allowfullscreen>
    </iframe>
  </div>
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Demo: J0 sweep detecting the object and recentering</em></figcaption>
</figure>

## Next

Once `explore.py` returns `True`, `main.py` calls `navigate.py` to compute the precise hover position above the object using ArUco PnP and move the TCP there.
