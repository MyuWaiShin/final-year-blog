---
title: "Camera Mount Redesign and Current Project Status"
date: 2026-03-04 12:00:00 +0000
categories: [Project Updates, Hardware]
tags: [camera mount, 3d printing, eye-in-hand, calibration, hardware design]
author: myuwaishin
pin: false
last_modified_at: false
---

The first camera mount I designed and printed did not work. The angle was too aggressive, the height was too low, and the camera was not looking at the gripper the way I needed it to. Before I could run another calibration session, I needed to fix the mount. Today I mostly spent time taking accurate measurements, redesigning the camera mount and assessing where I stand with the project.

---

## Rejected Previous Mount

The previous mount positioned the camera too low and tilted it too steeply. The camera was not really capturing the gripper properly in frame. Part of the reason was that the measurements I took were a replica of the current adjustable camera mount position, which has two pivot points for tilt adjustment. This made it difficult to reason about the actual camera angle, as I had to consider the combined effect of two offsets at once.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/cam_tilt_design.jpeg" alt="Two-pivot adjustable camera mount" style="display:block; margin:0 auto; width:70%; border-radius:8px;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Two-pivot adjustable camera mount</em></figcaption>
</figure>

That is not a good way to design something that needs precise angular control. This time I detached the gripper from the bracket entirely and measured the geometry precisely referenced to the camera window view instead of taking rough estimates mid-air.

---

## Design v3: Angle Reduced from 47° to 30°

The first fix was to reuse the existing model but reduce the tilt angle from 47° to 30°. The 47° tilt had been far too steep. After looking at what field of view was needed to see both the gripper fingers and enough of the workspace below, 30° was a better estimate.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/cam_mount_3.png" alt="Camera mount v3: two-pivot, 30 degree tilt" style="display:block; margin:0 auto; width:70%; border-radius:8px;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Mount v3: two-pivot, 30 degree tilt</em></figcaption>
</figure>

This design still has a structural problem. There is two separate points of tilt in the model, which made it more error-prone to estimate the actual tilt angle.

---

## Design v4.1: Vertical Stand, Single Tilt at the Camera

For the next design I changed the approach entirely. Instead of having the entire arm angled, I made the stand body vertical and shifted the tilt to only the camera seating area at the top. The camera moves back slightly relative to the previous design, viewing from a slightly lower and more rearward position rather than directly above.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/cam_mount_4_1.png" alt="Camera mount v4.1: vertical stand, single tilt at the camera seat" style="display:block; margin:0 auto; width:70%; border-radius:8px;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Mount v4.1: vertical stand, single tilt at the camera seat</em></figcaption>
</figure>

I cut out the top cover of the mount and the back cover to reduce weight and material usage. Also increased the width of the bracket from 60 to 61 mm to fit the the gripper's own bracket better since the previous one was a bit too tight.

**Measurements for v4.1:**
- Distance from camera lens to the end of the gripper bed (including the ~36 mm gripper width): approximately **125 mm**
- Tilt angle: **30°**

This is a better design. There is one tilt angle, at one location. If an adjustment is needed, I can easily change the height or the angle parameter.

---

## Design v4.2: Lower Height, Smaller Angle

I was still concerned that v4.1's 30° tilt might still be too much. So I designed a second variant with both the height and the angle brought down, as a fallback in case v4.1 overestimates the needed tilt again.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/cam_mount_4_2(final).png" alt="Camera mount v4.2: shorter height, shallower tilt angle" style="display:block; margin:0 auto; width:70%; border-radius:8px;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Mount v4.2: shorter height, shallower tilt angle</em></figcaption>
</figure>

**Measurements for v4.2:**
- Distance from camera lens to end of gripper bed: approximately **110 mm**
- Tilt angle: **21°**

Having both variants will give me fallback options and allow me to redesign the mount with better precision if needed.

---

## Getting Them Printed

I asked a friend to 3D print all three designs as it would be quicker than waiting on a print queue. Calibration is a hard dependency for my navigation and recovery loop. The faster I can get a working mount, the faster I can get a working calibration.

---

## What Comes Next

Since the calibration work so far has been constrained by the camera position being temporary, once a mount is physically fixed, I plan to run a full calibration session with more varied wrist orientations for 30 different poses. This was the primary weakness identified from the replay analysis in the previous session

---

## Where the Project Stands

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/system_flow_simplified.png" alt="System overview" style="display:block; margin:0 auto; width:100%; border-radius:8px;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>System overview</em></figcaption>
</figure>

I am taking a step back to summarise the current state of each component:

| Component | Status |
|-----------|--------|
| YOLO object detection model | Done ✓ |
| Detection adjustments (OBB, orientation estimation) | Done ✓ |
| Stereo depth estimation | Done ✓ |
| Stage 1 failure detection (gripper feedback only) | Done ✓ |
| Slip failure detection logic | Done ✓ |
| Hand-eye calibration | In progress |
| Camera mount | Redesigned, awaiting print |
| Navigation / robot motion planning | Blocked on calibration |
| Stage 2 failure detection (binary classifier) | Pending |
| Stage 2 integration into pipeline | Pending |
| Full recovery loop integration | Pending |

The project is halfway through the remaining implementation work. Some schedule slip has accumulated, mainly because calibration has taken longer to stabilise than expected. The important thing is that all the individual components that are blocked on calibration are otherwise well-understood. Once a reliable transform is in place, the navigation work should move quickly.

The final pipeline integration will be the most time-consuming phase not because it is technically complex, but because getting all the pieces to behave correctly together under real conditions always takes debugging time. That is expected. The plan is to have all the individual pieces tested and verified before integration begins, so the debugging surface is as narrow as possible.
