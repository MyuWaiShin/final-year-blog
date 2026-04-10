---
title: "Stages 3 and 4: Grasp and Verify"
date: 2026-03-24 12:00:00 +0000
categories: [Project Updates, Pipeline]
tags: [grasp, verify, yolo, failure detection, width check]
author: myuwaishin
pin: false
last_modified_at: false
---

> **Note:** This post is written in April, backdated to the session date of 24 March.
{: .prompt-info }

> Find the scripts here: [grasp.py](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/full_pipeline/grasp.py) and [verify.py](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/full_pipeline/verify.py)

---

## Stage 3: Grasp

With the TCP hovering at `hover_z` directly above the object, `grasp.py` takes over.

### Step 1: Descend

The robot descends 70 mm from the hover position at 20 mm/s and closes the gripper.

```python
movel(p[x, y, hover_z - 0.07, rx, ry, rz], a=0.05, v=0.02)
close_gripper()
```

The 70 mm descent is calculated to ensure enough bottle contact surface:

- The gripper jaw cushion span is 33 mm
- The TCP height difference from fully open to fully closed is 28 mm
- PnP pose estimation places the TCP slightly above the tag, measuring about 9 mm additional offset

33 + 28 + 9 = 70 mm total descent to reach a gripping position with sufficient surface area.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/grasp_surface.jpeg" alt="Grasp Surface" style="width:40%; border-radius:6px; display:block; margin:0 auto;">
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Gripper contact surface at the gripping position</em></figcaption>
</figure>


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

**Why 400 mm?** During testing, a false positive scenario was observed: when the gripper is at a close distance to the object, the bottle can appear fully between the jaws in the camera frame even when it is not physically gripped. At close range, the classifier predicts `holding` although the gripper is actually empty. This can be seen in the 1st image.

Lifting to clearance height (>150 mm) eliminates this edge case as seen in the 2nd image, the object is no longer filling the frame and the classifier reaches near 100% confidence consistently. 400 mm was chosen as a safe margin above the critical threshold.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/false_positive_classify2.png" style="width:60%; border-radius:6px; display:block; margin:0 auto;" alt="False positive at low height">
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>False positive: gripper classifies as holding at close range when object is not gripped</em></figcaption>
</figure>

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

## Demo

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <div style="width:80%; position:relative; padding-bottom:45%; height:0; overflow:hidden; border-radius:8px;">
    <iframe src="https://www.youtube.com/embed/yCYymfOec5g"
      style="position:absolute; top:0; left:0; width:100%; height:100%; border:0;"
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
      allowfullscreen>
    </iframe>
  </div>
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Demo: grasp and verify</em></figcaption>
</figure>

## Next

If verify passes, `transit.py` carries the object horizontally at `clearance_z` to the drop zone.
