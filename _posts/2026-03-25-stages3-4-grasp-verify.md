---
title: "Stages 3 and 4: Grasp and Verify"
date: 2026-03-25 12:00:00 +0000
categories: [Project Updates, Pipeline]
tags: [grasp, verify, yolo, failure detection, width check]
author: myuwaishin
pin: false
last_modified_at: false
---

> **Note:** This post is written in April, backdated to the session date of 25 March.
{: .prompt-info }

> Script: `full_pipeline/grasp.py` and `full_pipeline/verify.py`
{: .prompt-info }

---

## Stage 3: Grasp

With the TCP hovering at `hover_z` directly above the object, `grasp.py` takes over.

### Step 1: Descend

The robot descends 70 mm from the hover position at 20 mm/s and closes the gripper.

```python
movel(p[x, y, hover_z - 0.07, rx, ry, rz], a=0.05, v=0.02)
close_gripper()
```

This hoever 

### Step 2: Layer 1 — Width Check

Once the gripper closes, the jaw width is read back from the RG2 via the AI2 analogue input on the robot controller. The width indicates whether anything is between the jaws.

| Width reading | Interpretation |
|---|---|
| < 11 mm | Object missed — jaws closed on air |
| ≥ 11 mm | Object may be between the jaws |

### Outcome

If width is below threshold, recovery is triggered immediately. If width passes, control moves to Stage 4.

---

## Stage 4: Verify

`verify.py` provides a second layer of confirmation before the robot commits to carrying the object.

### Step 1: Rise to Clearance Height

The robot lifts straight up to `clearance_z` (400 mm above `hover_z`). This gives the camera a clear overhead view of the closed gripper.

```python
movel(p[x, y, clearance_z, rx, ry, rz], a=0.1, v=0.1)
```

### Step 2: Layer 2 — YOLO26n Classify

`yolo26n_cls_V1` classifies the current camera frame as either `holding` or `empty`. The threshold is 90% confidence.

| Result | Action |
|---|---|
| `holding` ≥ 90% | Proceed to Stage 5 (transit) |
| `empty` or confidence below 90% | Recovery triggered |

The classifier was trained on overhead views at clearance height, so this is exactly the viewpoint it was designed for.

---

## Next

If verify passes, `transit.py` carries the object horizontally at `clearance_z` to the drop zone.
