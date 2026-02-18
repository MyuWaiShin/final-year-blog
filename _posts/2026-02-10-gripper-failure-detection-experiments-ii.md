---
title: "Gripper-Based Failure Detection Experiments II"
date: 2026-02-10 14:00:00 +0000
categories: [Project Updates, Hardware Testing]
tags: [gripper, failure-detection, rg2, ur10, pick-and-place, experiments]
author: myuwaishin
pin: false
last_modified_at: false
---

Following on from the calibration and detection logic established in [Experiments I]({% post_url 2026-02-09-gripper-feedback-experiments %}), today I ran a full set of pick-and-place experiments to stress-test the gripper-only detection approach across different objects, orientations, and positions.

**The core question:** Can force sensing + gripper width alone reliably tell the robot whether it has grabbed something?

---

## Experimental Setup

I tested preprogrammed pick-and-place poses with controlled variations across three key variables:

| Variable | Conditions Tested |
|---|---|
| Object presence | Object at pick location vs. empty |
| Object orientation | 0° (aligned) vs. 45° rotated |
| Position offset | Exact preprogrammed position vs. slightly shifted |

**Objects used:** Lightweight styrofoam cubes and cylinders.

---

## Robot Movement Strategy

Each pick-and-place cycle followed a fixed motion sequence:

| Movement | Motion Type | Reason |
|---|---|---|
| Move to home | `movej` | Safe starting configuration |
| Pick approach (above object) | `movel` | Precise straight-line descent |
| Down to pick | `movel` | Precise straight-line descent |
| Lift up (above object) | `movel` | Precise straight-line lift |
| Transfer to place approach | `movej` | Long distance — avoids singularities |
| Down to place | `movel` | Precise straight-line descent |
| Retract up (above place) | `movel` | Precise straight-line lift |

**Detection logic during pick:**
- Force HIGH + Width **> 11mm** → Object grabbed → proceed to place
- Force HIGH + Width **< 11mm** → Gripper fully closed, nothing grabbed → skip place, try next pick position
- After all picks, return home and close gripper for safety

The three pick positions are offset by 200mm steps along the positive X axis of the robot frame.

---

## Experiment 1 — Large Cubes, Object Present vs Empty Pick

<div style="position: relative; width: 80%; margin: 0 auto; padding-bottom: 45%; height: 0; overflow: hidden; border-radius: 8px; box-shadow: 0 4px 12px rgba(0,0,0,0.2);">
  <video controls style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;">
    <source src="{{ '/assets/img/1.mp4' | relative_url }}" type="video/mp4">
  </video>
</div>
<p style="text-align: center; font-size: 0.9rem; color: #888; margin-top: 0.5rem;"><em>Experiment 1 — Large cubes at exact preprogrammed positions and empty pick locations (10x speed)</em></p>


**First half:** All three cubes placed exactly at the preprogrammed positions.

**Second half:** No cubes placed — empty pick locations.

**Result:** Full success.
- All cubes were picked and placed correctly in the first half.
- In the second half, the gripper correctly detected nothing was grabbed (width < 11mm) and skipped the place step for all three positions.

This validated the core detection concept under ideal conditions.

---

## Experiment 2 — Large Cubes, 45° Rotation

<div style="position: relative; width: 80%; margin: 0 auto; padding-bottom: 45%; height: 0; overflow: hidden; border-radius: 8px; box-shadow: 0 4px 12px rgba(0,0,0,0.2);">
  <video controls style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;">
    <source src="{{ '/assets/img/2.mp4' | relative_url }}" type="video/mp4">
  </video>
</div>
<p style="text-align: center; font-size: 0.9rem; color: #888; margin-top: 0.5rem;"><em>Experiment 2 — Large cubes rotated 45° (10x speed)</em></p>


**Setup:** Same positions, but cubes rotated 45°. Repeated twice.

**Result:** Partial failure — third cube not picked.

The gripper touched the third cube but did not close fully enough to secure it. However, the system still reported a successful pick because:
- Force sensor (DI8) = HIGH (gripper touched the cube)
- Width > 11mm (gripper didn't fully close)

Both conditions matched the "object grabbed" criteria, so the robot proceeded to place, even though nothing was actually in the gripper.

Force + width alone is not sufficient. The gripper can satisfy both detection conditions by merely touching an object from the side, without actually securing it. A third sensor or confirmation method is needed.

---

## Experiment 3 — Small Cubes, Aligned then 45° Rotated

<div style="position: relative; width: 80%; margin: 0 auto; padding-bottom: 45%; height: 0; overflow: hidden; border-radius: 8px; box-shadow: 0 4px 12px rgba(0,0,0,0.2);">
  <video controls style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;">
    <source src="{{ '/assets/img/3.mp4' | relative_url }}" type="video/mp4">
  </video>
</div>
<p style="text-align: center; font-size: 0.9rem; color: #888; margin-top: 0.5rem;"><em>Experiment 3 — Small cubes, aligned then 45° rotated (10x speed)</em></p>


**First half (0° rotation, exact positions):** All successful.

**Second half (45° rotation):** Same failure pattern as Experiment 2.

The third cube was again not picked. My interpretation is that since the gripper orientation does not change when approaching the third position, the gripper descends slightly off-centre relative to the rotated cube. The gripper jaws then nudge the cube as they close, physically shifting it from its original position.

This introduces an additional failure type worth noting:

> **Contact-induced displacement** — the gripper physically moves the object during a failed pick attempt, making the position unreliable for any retry.

Simple retries will not work in this case because the object is no longer where it was.

---

## Experiment 4 — Cylinders, Upright then Random Orientation

<div style="position: relative; width: 80%; margin: 0 auto; padding-bottom: 45%; height: 0; overflow: hidden; border-radius: 8px; box-shadow: 0 4px 12px rgba(0,0,0,0.2);">
  <video controls style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;">
    <source src="{{ '/assets/img/4.mp4' | relative_url }}" type="video/mp4">
  </video>
</div>
<p style="text-align: center; font-size: 0.9rem; color: #888; margin-top: 0.5rem;"><em>Experiment 4 — Cylinders upright (success) then random orientation (failures) (10x speed)</em></p>

**First half (upright, exact positions):** All successful.

**Second half (random orientations):** Failures including a new failure type.

When one cylinder was oriented horizontally — parallel to the gripper fingers — the robot failed to grab it. The cylinder's length is greater than the maximum opening width of the RG2 gripper (~100mm), so the gripper could not close around it.

Instead, the gripper descended and pressed down on top of the cylinder, triggering the force sensor. The system interpreted this as a successful pick.

**New failure type identified: Orientation mismatch**

The gripper needs to adapt its approach angle to match the object's orientation. A fixed approach direction will fail for elongated objects in certain orientations.

---

## Experiment 5 — Cylinders, Shifted Positions + Random Orientation

<div style="position: relative; width: 80%; margin: 0 auto; padding-bottom: 45%; height: 0; overflow: hidden; border-radius: 8px; box-shadow: 0 4px 12px rgba(0,0,0,0.2);">
  <video controls style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;">
    <source src="{{ '/assets/img/5.mp4' | relative_url }}" type="video/mp4">
  </video>
</div>
<p style="text-align: center; font-size: 0.9rem; color: #888; margin-top: 0.5rem;"><em>Experiment 5 — Cylinders with position offsets and random orientations (10x speed)</em></p>

**Result:** All picks failed.

This experiment combined two fundamental failure sources: position offset and orientation mismatch. All attempts failed. This confirmed that:

- Force can be triggered by top contact, not just side contact between the jaws
- Width measurement alone cannot distinguish "object between jaws" from "object under jaws"

The gripper feedback system has no way to know *where* the object is relative to the jaws.

---

## Experiment 6 — Slip Failure During Transfer

<div style="position: relative; width: 80%; margin: 0 auto; padding-bottom: 45%; height: 0; overflow: hidden; border-radius: 8px; box-shadow: 0 4px 12px rgba(0,0,0,0.2);">
  <video controls style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;">
    <source src="{{ '/assets/img/6.mp4' | relative_url }}" type="video/mp4">
  </video>
</div>
<p style="text-align: center; font-size: 0.9rem; color: #888; margin-top: 0.5rem;"><em>Experiment 6 — Slip failure during transfer with arc-shaped objects (10x speed)</em></p>

Testing with different shaped objects (cubic arc, small cube, cylinder) in random orientations revealed another failure mode: **slip during transfer**.

The arc was picked up with a marginal grip. But during the fast `movej` transfer to the place position, the object slipped and fell mid-trajectory.

**New failure type identified: Slip during transfer**

An insecure grip that passes the initial detection check can still result in a dropped object during movement. Continuous grip monitoring during transfer (not just at the moment of pick) is needed.

---

## Summary of Failure Types Identified

| # | Failure Type | Cause | Detection Gap |
|---|---|---|---|
| 1 | **False positive — side contact** | Gripper touches object but doesn't close around it | Force HIGH + Width > 11mm even without secure grip |
| 2 | **Contact-induced displacement** | Gripper nudges object during failed pick | Object moves, retry position is now wrong |
| 3 | **Orientation mismatch** | Object too long / wrong angle for gripper approach | Force triggered by top contact, not jaw contact |
| 4 | **Slip during transfer** | Insecure grip passes detection, object drops mid-move | No continuous monitoring after initial pick check |

---

## Conclusions

After six experiments, the verdict is clear: **gripper feedback alone is not sufficient for reliable failure detection.**

| Signal | Limitation |
|---|---|
| Force (DI8) | Triggers on any contact — top, side, or between jaws |
| Width (AI2) | Doesn't confirm object is *between* the jaws |
| Combined | Can produce false positives when gripper barely touches object |

### What's Needed

The system needs a combination of:
1. **Camera (vision)** — confirm object is detected and estimate its pose before picking
2. **Gripper feedback** — confirm jaws closed around something with appropriate width
3. **Depth change verification** — check if object height changed after pick (confirming it was lifted)

### Proposed Recovery Logic

| Condition | Action |
|---|---|
| Camera: object detected + Gripper: fully closed (width < 11mm) | Simple retry — object was missed cleanly |
| Camera: object detected + Gripper: partially closed (force HIGH, width > 11mm) | Verify with additional sensor or depth check; if unconfirmed → recovery (adjust TCP rotation, height, or lateral shift) |
| Camera: no object detected | Skip — object not present |

Simple retries are only safe when the gripper fully closed without touching anything. In all other cases, a smarter recovery strategy is needed — especially given that failed picks can physically displace the object.

---

## Next Steps

These experiments have been essential for mapping out the failure landscape before building the recovery system. The next phase is:

1. **Start the perception pipeline** — integrate the OAK-D camera for real-time object detection and pose estimation
2. **Implement pick-and-place with pose estimation** — replace hardcoded positions with vision-guided picks
3. **Combine gripper feedback + vision** for robust failure detection and recovery

---

**Code Repository:** All scripts are available in the [UR10 folder](https://github.com/MyuWaiShin/Final_Year_Project_2026/tree/main/UR10) of my [project repository](https://github.com/MyuWaiShin/Final_Year_Project_2026).
