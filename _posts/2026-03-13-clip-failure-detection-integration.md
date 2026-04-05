---
title: "Implementing Full Grasp Failure Detection Pipeline"
date: 2026-03-13 12:00:00 +0000
categories: [Project Updates, Failure Detection]
tags: [clip, failure-detection, pipeline, rtde, urscript, slip-detection]
author: myuwaishin
pin: false
last_modified_at: false
---

> **Note:** This post is written in April, backdated to the session date of 13 March.
{: .prompt-info }


Today I started implementing the full grasp failure detection pipeline. This is a proof of concept integrating three check points to verify whether the robot has grasped an object at each stage of the pick sequence. This post describes the logic of each check point and the steps taken to build and integrate them.

The three check points are:

1. Gripper width check after initial close (Check 1)
2. CLIP visual verification after lift (Check 2)
3. Slip detection during transit to place (Check 3)

The pipeline does not include navigation, retry or recovery motion at this stage. When a check fails, a recovery flag is raised. The robot places the object only if all three checks pass. The pipeline is solely for the proof of concept for the failure detection.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/failure_detection_pipeline.png" alt="Failure detection pipeline flowchart" style="display:block; margin:0 auto; width:90%; border-radius:4px;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Failure detection pipeline with three check points</em></figcaption>
</figure>

---

## Step 1: Integrating Check 1 and Check 2

I started by integrating the first two checks before adding slip detection.

### Check 1: Gripper Width

After the gripper closes, the script reads the RG2 jaw width from the analogue voltage on `AI2` and makes the first decision:

- **Width ≤ 11 mm** → gripper closed fully, nothing was in the way → flag recovery
- **Width > 11 mm** → fingers are spread by something → proceed

The width is converted from the raw voltage using a two-point linear calibration:

```python
raw_mm = (voltage / 3.7) * 110
width  = raw_mm * slope + offset
# calibrated: raw≈8.5mm → actual 10.5mm, raw≈65.8mm → actual 91.0mm
```

Width alone is not enough to confirm a good grasp, but it catches definite misses without running the camera.

### Check 2: CLIP Visual Verification

After passing the width check, the robot lifts 200 mm. The script then:

1. Takes the latest frame from the OAK-D camera queue (streaming at 30 fps)
2. Crops a 1400×600 px ROI from the bottom of the frame
3. Passes the crop through CLIP ViT-B/32 → 512-d feature vector
4. Runs the trained logistic regression probe → class probabilities

```python
max(probs) < 0.75               # Uncertain - flag recovery
pred == 1 and prob >= 0.75      # Holding - proceed
pred == 0 and prob >= 0.75      # Empty - flag recovery
```

Only a Holding result at ≥75% confidence allows the pipeline to continue.

### RTDE Issues

The initial version used `RTDEControlInterface` for motion commands. Every step was taking around 7 seconds, moving to target, descending, closing, lifting because `RTDEControlInterface` reconnects and re-uploads on every call, and each call also invoked `stopScript()` before running any gripper URP, adding another round-trip. The overhead made the pipeline too slow to use.

So I removed `RTDEControlInterface` entirely. Motion is sent as raw URScript strings to port 30002 via a persistent socket. `RTDEReceiveInterface` (port 30004, read-only) polls `getActualTCPPose()` every 10 ms to detect when a move completes.

---

## Test Results: Check 1 and Check 2

After switching to raw URScript, each motion step completed in under a second. With the pipeline running at normal speed, I tested a pick attempt where the gripper intentionally missed the object first.

On the first attempt, the gripper closed on empty space and missed. CLIP classified the result as Empty and flagged retry.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/empty_clip.png" style="display:block; margin:0 auto; width:80%; border-radius:6px;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>CLIP output — Empty, below confidence threshold</em></figcaption>
</figure>

On the retry, the gripper picked up the object. CLIP classified it as Holding above 75% confidence and the pipeline continued to place.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/holding_clip.png" style="display:block; margin:0 auto; width:80%; border-radius:6px;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>CLIP output — Holding, above confidence threshold</em></figcaption>
</figure>

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <div style="width:90%; position:relative; padding-bottom:56.25%; height:0; overflow:hidden; border-radius:8px;">
    <iframe src="https://www.youtube.com/embed/53IFbvx_Keg"
      style="position:absolute; top:0; left:0; width:100%; height:100%; border:0;"
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
      allowfullscreen>
    </iframe>
  </div>
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Running Post Grasp Verification using CLIP</em></figcaption>
</figure>

---

## Step 2: Attempting Check 3 (Slip Detection During Transit)


Once Check 1 and Check 2 were working, I started adding Check 3.

The idea was to run a background thread that polls the gripper width every 100 ms during transit. A URP (`close_gripper_timed.urp`) would run on the robot's dashboard in parallel with arm motion, re-issuing the close command every ~200 ms. If the width dropped back below the threshold, the thread would set a slip flag, stop the arm, and re-run CLIP.

### Port Layout

Three channels were running simultaneously:

| Port | Purpose |
|---|---|
| 29999 (Dashboard) | Loading and playing gripper URPs |
| 30002 (Secondary Interface) | Sensor stream (AI2 + DI8) and URScript motion commands |
| 30004 (RTDE Receive) | Read-only TCP pose polling every 10 ms |

### Issues

The first issue was that `dashboard_cmd("stop")` was being called immediately after the transit move completed, which killed the re-close loop too early and so the gripper was not actively re-closing during transit. 

The second was an attempt to embed the URscript gripper close command `rg_grip()` inline and send it via port 30002. The script compiled fine but the gripper did not respond. `rg_grip()` is a URCap function and only works when called through the dashboard URP interpreter, not the secondary client 30002. So then I reverted to the URP approach and sent the URP via the dashboard port 29999 and stumbled upon the first issue again. The URP close command cannot complete once the arm motion started, so the gripper wasn't actively re-closing during transit.

After working through these bugs for quite some time, I couldn't manage to get the close loop URP approach to work for slip detection. Slip detection was listed as an optional objective and was not a core requirement, so I decided to stop here and move on. Check 1 and Check 2 together are sufficient to verify grasp state before and immediately after lift. I will try to implement the slip detection again in the future if I have time.

---

## Next

With the proof-of-concept failure detection pipeline working for Check 1 and Check 2, the next step is to mount the camera on the final bracket, do a fresh calibration with the fixed mount, and start implementing the full end-to-end pick-and-place pipeline.

