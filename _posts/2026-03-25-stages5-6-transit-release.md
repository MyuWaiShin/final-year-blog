---
title: "Stages 5 and 6: Transit and Release"
date: 2026-03-25 13:00:00 +0000
categories: [Project Updates, Pipeline]
tags: [transit, release, slip detection, yolo, urscript]
author: myuwaishin
pin: false
last_modified_at: false
---

> **Note:** This post is written in April, backdated to the session date of 25 March.
{: .prompt-info }

> Find the scripts here: [transit.py](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/full_pipeline/transit.py) and [release.py](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/full_pipeline/release.py)

---

## Stage 5: Transit

After the robot has successfully grasped and verified it is holding the object, `transit.py` carries it horizontally to the drop zone, staying at `clearance_z` the entire time.

### The Move

The drop zone X and Y coordinates are loaded from `data/drop_zone.json`, the TCP pose I saved by manually positioning the robot above the drop surface. Z stays fixed at `clearance_z`.

```python
target_x = drop_zone["x"]
target_y = drop_zone["y"]
target_z  = clearance_z
movel(p[target_x, target_y, target_z, rx, ry, rz], a=0.02, v=0.08)
```

Transit cannot just blindly drive to the drop zone, the object might slip mid-transit if the grip was not secure enough. So, I added a slip monitor during the carry phase. The way it works is it catches the object loss when the object is completely out of the gripper, and not the continuous decrease in grip force during the time of slipping.

### YOLO Classify Monitor

<div style="text-align:center; margin:1.5rem 0;">
  <video src="{{ '/assets/img/classifier_live_transit.mp4' | relative_url }}"
    style="width:70%; border-radius:8px; display:inline-block;"
    autoplay loop muted controls playsinline
    onended="this.currentTime=0;this.play();">
  </video>
  <p style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>YOLO slip monitor detecting object loss during transit</em></p>
</div>

A background thread runs `yolo26n_cls_V1` on live OAK-D camera frames throughout the move. After 3 consecutive `empty` predictions (confidence < 0.90), it sends `stopl()` and triggers recovery.

```python
if p_holding < 0.90:
    consec_empty += 1
else:
    consec_empty = 0
if consec_empty >= 3:
    stopl()
    return {"result": "empty", "layer": 2}
```

Three consecutive frames are required to avoid false triggers from a single blurry or occluded frame during arm motion.

### What Transit Returns

| Result | Meaning |
|---|---|
| `{"result": "arrived"}` | Drop zone reached, object still held |
| `{"result": "slip", "layer": 1}` | Gripper width dropped mid-transit |
| `{"result": "empty", "layer": 2}` | YOLO detected no object between fingers |

If `slip` or `empty`, `main.py` sends the pipeline to recovery.

---

## Stage 6: Release

Release is the final stage. The robot descends to drop height, opens the gripper, rises to clearance height, then returns to the scan pose ready for the next cycle.

### Step 1: Descend to Drop Height

```python
drop_z = clearance_z - DROP_HEIGHT_M   # = clearance_z - 0.470m = hover_z - 0.07m
movel([x, y, drop_z, rx, ry, rz], vel=0.02, acc=0.008)
```

`DROP_HEIGHT_M = 0.470 m` is set so that `drop_z = hover_z - 0.07 m` — the same depth the gripper was at when it picked the object up. This ensures the object is placed at exactly the right height without being pressed into the surface. The descent is slow (20 mm/s) to avoid jarring it loose just before release.

### Step 2: Open Gripper

The gripper opens via a URP program triggered through the dashboard port — the same mechanism used throughout the pipeline.

```python
_dashboard_cmd("load /programs/myu/open_gripper.urp")
_dashboard_cmd("play")
_wait_urp_done()
```

If the gripper is already open (width ≥ 80 mm), the URP trigger is skipped to avoid an unnecessary program load.

### Step 3: Rise to Clearance Height

After releasing, the arm rises back to `clearance_z` to clear the drop zone before swinging back, so there is no risk of knocking the just-placed object.

```python
movel([tcp_x, tcp_y, clearance_z, rx, ry, rz], vel=0.08)
```

### Step 4: Return to Scan Pose

A joint move returns the robot to the scan pose — the exact joint configuration saved at the start — resetting it for the next object.

```python
movej([j0, j1, j2, j3, j4, j5], a=0.3, v=0.5)
```

`release.py` always returns `True`. It is the only stage in the pipeline that cannot fail — once the object is released, the cycle is complete regardless of where it landed.

## Demo

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <div style="width:80%; position:relative; padding-bottom:45%; height:0; overflow:hidden; border-radius:8px;">
    <iframe src="https://www.youtube.com/embed/W2fNheZw6LU"
      style="position:absolute; top:0; left:0; width:100%; height:100%; border:0;"
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
      allowfullscreen>
    </iframe>
  </div>
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Demo: transit and release</em></figcaption>
</figure>

## Next

With all 6 stages of pick and place with failure detection complete, I will explain how my recovery routine works in the next post.