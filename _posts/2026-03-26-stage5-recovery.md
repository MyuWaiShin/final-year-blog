---
title: "Stage 5: Recovery Loop"
date: 2026-03-26 12:00:00 +0000
categories: [Project Updates, Pipeline]
tags: [recovery, yolo, failure detection, circular sweep, recentering]
author: myuwaishin
pin: false
last_modified_at: false
---

> **Note:** This post is written in April, backdated to the session date of 26 March.
{: .prompt-info }

> Script: `full_pipeline/recover.py`
{: .prompt-info }

---

## When Recovery Triggers

Recovery is called whenever a grasp or verify failure is detected:

- **Layer 1 fail**: jaw width < 11 mm — object was missed entirely
- **Layer 2 fail**: YOLO26n classifies as `empty` at clearance height — object not in gripper
- **Object lost during transit**: 3 consecutive empty frames during slip monitoring

Each failure increments a counter. After **5 total failures**, the pipeline aborts.

---

## Recovery Sequence

### 1. Open Gripper

The gripper opens immediately to release any partial contact.

### 2. Rise to Scan Height

The arm rises directly above the area where the object was last seen, returning to the same height used during the initial sweep.

### 3. 60 mm Circular Sweep

The arm traces a 60 mm radius circle across 5 waypoints while `yolo26n_detect_V1` runs on every camera frame. As soon as the object is detected, the sweep stops immediately.

```python
for waypoint in circle_waypoints:
    if object_detected:
        stopj(accel=0.8)
        break
    movej(waypoint)
```

### 4. Active J0 Recentering

Once the object is found, J0 corrections centre it within ±20 px of the frame centre. Centering matters here for the same reason as during the initial sweep — PnP pose estimation is most accurate when the object is near the optical centre where lens distortion is lowest.

### 5. Back to Navigate

Control returns to `navigate.py`. The pipeline retries from pose estimation — the robot re-runs the full ArUco PnP / stereo depth pipeline and moves to a fresh hover position.

---

## Key Parameters

| Parameter | Value |
|---|---|
| Sweep radius | 60 mm |
| Sweep waypoints | 5 |
| YOLO confidence threshold | ≥ 80% |
| Recentre tolerance | ±20 px |
| Max total failures | 5 → abort |

---

## Next

After recovery re-centres the object, `navigate.py` takes over again and the pipeline retries from Stage 2.
