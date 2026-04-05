---
title: "Eye-in-Hand Calibration III: 30 Poses"
date: 2026-03-06 12:00:00 +0000
categories: [Project Updates, Calibration]
tags: [charuco, hand-eye calibration, tsai-lenz, aruco, pose estimation, tcp]
author: myuwaishin
pin: false
last_modified_at: false
---

The previous session identified the main problem where the wrist orientation was not varied enough across the collected poses, leaving the solver without enough rotational signal to constrain the transform. This time I ran a full 30-pose collection with deliberate wrist rotation variation and the result is evaluated below.

> **Note:** This post is written in April, backdated to the session date of 6 March.
{: .prompt-info }

---

## How the Calibration Script Works

The calibration script has two operating modes and handles both data collection and pose replay.

> Find the calibration script at [07_hand_eye_calibration.py](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/pipeline_dev/scripts/07_hand_eye_calibration.py).
{: .prompt-info }

A key change this session was using free-drive mode to move the robot instead of the teach pendant. Free-drive lets me physically push the arm to reach large rotational variations and completely arbitrary wrist angles. This was the main problem from the previous session where all poses had similar wrist orientations. It also means the dataset is more randomised. I'm not biased towards symmetric or incremental movements the way I would be when stepping through the pendant menus.

Each captured pose saves the joint angles to `calibration_poses.json`, which enables the second operating mode below for replay without having to re-run the full manual session.

**Manual Mode: `--pose false`**

The robot is moved into position using free-drive. The live camera feed is shown. When the ChArUco board is stable and visible, `SPACE` captures the pose. Each capture records the robot's flange pose from RTDE and the board pose from ChArUco PnP detection.

<div style="display:grid; grid-template-columns:repeat(3,1fr); gap:0.5rem; margin:1.5rem auto; max-width:90%;">
  <figure style="margin:0; text-align:center;">
    <img src="/assets/img/cali1.png" alt="Calibration pose 1" style="width:100%; border-radius:6px;" />
    <figcaption style="font-size:0.75rem; color:#666; margin-top:0.2rem;"><em>Pose 1</em></figcaption>
  </figure>
  <figure style="margin:0; text-align:center;">
    <img src="/assets/img/cali16.png" alt="Calibration pose 16" style="width:100%; border-radius:6px;" />
    <figcaption style="font-size:0.75rem; color:#666; margin-top:0.2rem;"><em>Pose 16</em></figcaption>
  </figure>
  <figure style="margin:0; text-align:center;">
    <img src="/assets/img/cali18.png" alt="Calibration pose 18" style="width:100%; border-radius:6px;" />
    <figcaption style="font-size:0.75rem; color:#666; margin-top:0.2rem;"><em>Pose 18</em></figcaption>
  </figure>
  <figure style="margin:0; text-align:center;">
    <img src="/assets/img/cali24.png" alt="Calibration pose 24" style="width:100%; border-radius:6px;" />
    <figcaption style="font-size:0.75rem; color:#666; margin-top:0.2rem;"><em>Pose 24</em></figcaption>
  </figure>
  <figure style="margin:0; text-align:center;">
    <img src="/assets/img/cali25.png" alt="Calibration pose 25" style="width:100%; border-radius:6px;" />
    <figcaption style="font-size:0.75rem; color:#666; margin-top:0.2rem;"><em>Pose 25</em></figcaption>
  </figure>
  <figure style="margin:0; text-align:center;">
    <img src="/assets/img/cali26.png" alt="Calibration pose 26" style="width:100%; border-radius:6px;" />
    <figcaption style="font-size:0.75rem; color:#666; margin-top:0.2rem;"><em>Pose 26</em></figcaption>
  </figure>
</div>

**Auto-Replay Mode: `--pose true`**

Loads the pre-saved joint angles from `calibration_poses.json` and automatically commands the robot via `rtde_c.moveJ()` to replay each pose. After a 1.5 s settle wait, it tries for up to 3 s to auto-detect the ChArUco board.

I added auto-replay mode so that once the final camera mount is fitted, I can replay the exact same poses and recalibrate without repeating the manual collection session from scratch.

---


## How the Solver Works

At the end of collection, `cv2.calibrateHandEye` is called using the Tsai-Lenz method:

```python
R_cam2gripper, T_cam2gripper = cv2.calibrateHandEye(
    R_gripper2base_list, T_gripper2base_list,   # robot flange → base
    R_target2cam_list,   T_target2cam_list,     # board → camera
    method=cv2.CALIB_HAND_EYE_TSAI
)
```

| Input per pose | Source |
|---|---|
| `R_gripper2base`, `T_gripper2base` | RTDE `getActualTCPPose()` — robot forward kinematics |
| `R_target2cam`, `T_target2cam` | ChArUco board pose via PnP |

The Tsai-Lenz algorithm solves `AX = XB` where:

- **A** = relative motion of the robot flange between poses
- **B** = corresponding relative motion of the board as seen by the camera
- **X** = the unknown camera-to-flange transform

**Output:** `T_cam2flange.npy` is a 4×4 rigid-body transform saved to `calibration/`. This describes the camera's position and orientation relative to the robot's tool flange, and is the transform used by the rest of the pipeline to convert pixel-space detections into robot base frame coordinates.

**Why cam2flange and not cam2tcp?** The calibration is solved relative to the flange because that is the fixed mechanical reference point the robot reports from RTDE. The TCP is a separate offset defined in the robot's installation settings on the teach pendant. For my project I am using Installation 8 and have set the TCP at Z = 186.6 mm and RZ = −3.142 rad relative to the flange, placing it at the midpoint between the gripper fingers for a secure centre-grip. The robot's kinematic chain applies this offset automatically when executing motion commands, so calibrating to the flange and letting the robot handle the TCP offset is the correct approach. If I had calibrated to the TCP directly, any change to the installation settings would invalidate the saved transform.

<div style="display:flex; justify-content:center; align-items:flex-start; gap:0.4rem; margin:1.5rem 0; flex-wrap:wrap;">
  <figure style="margin:0; text-align:center; width:48%;">
    <img src="/assets/img/flange2tcp.jpeg" alt="Flange to TCP offset setting" style="width:100%; height:400px; object-fit:cover; border-radius:8px;" />
    <figcaption style="font-size:0.8rem; color:#666; margin-top:0.3rem;"><em>TCP offset defined in installation settings</em></figcaption>
  </figure>
  <figure style="margin:0; text-align:center; width:48%;">
    <img src="/assets/img/tcp_point.jpeg" alt="TCP midpoint between gripper fingers" style="width:100%; height:400px; object-fit:cover; border-radius:8px;" />
    <figcaption style="font-size:0.8rem; color:#666; margin-top:0.3rem;"><em>TCP at midpoint of gripper fingers</em></figcaption>
  </figure>
</div>

**Summary flow:**

```
Collect N poses (≥15)
  ├─ Robot FK pose → T_flange2base
  └─ ChArUco PnP  → T_board2cam
           ↓
calibrateHandEye (Tsai-Lenz)
  └─ Solves AX = XB across all pose pairs
           ↓
Save T_cam2flange.npy
```

---

## Verification: ArUco Transform Check

After solving, the calibration is verified using a separate script that reads `T_cam2flange.npy` and transforms the detected ArUco tag pose into both the camera frame and the robot base frame simultaneously, displaying both on screen.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/aruco_transform.png" alt="ArUco pose shown in base frame and camera frame" style="display:block; margin:0 auto; width:80%; border-radius:8px;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>ArUco tag pose expressed in the robot base frame and camera frame simultaneously</em></figcaption>
</figure>

The script reports three values simultaneously. From the screenshot above:

**TCP frame: `(0.162, -0.043, 0.258) m`** (tag pose relative to the TCP). The ArUco tag is 16.2 cm to the right of the TCP centre, 4.3 cm behind it, and 25.8 cm below. These values match its physical position on the cube relative to the gripper.

**Base frame: `(0.059, -1.075, -0.625) m`** (tag pose in the robot base coordinate frame). X is near zero meaning the tag is close to the robot centreline, Y puts it about 1.07 m forward of the base, and Z at −62.5 cm places it below the robot mounting height, consistent with the robot being elevated above the workspace.

Since neither the robot base nor the ArUco board is moving, these base frame values should remain constant regardless of where the TCP moves. When I jogged the robot to different positions, I observed the base frame values staying stable. This is an indicator that the board consistency is good and the calibration is correct.

---

## Physical Test: Moving to the Tag

Using the base frame coordinates reported by the verification script, I typed the XYZ values directly into the `movel` command on the teach pendant to send the TCP to the tag's reported position.

The TCP consistently went to a position directly above the object and was able to pick it up. I repeated this several times to stress test the result, and it landed accurately most of the time.

The remaining inaccuracy is in the approach orientation, the gripper is currently approaching from the side at an angle rather than straight down from above. Once the approach direction is corrected to be perpendicular to the workspace, the pick accuracy should improve further.

---

## What's Next

With calibration working reliably, the camera-to-robot transform is no longer the bottleneck. The next component to implement is the binary visual classification layer, the Stage 2 failure detection to verify gripper state (holding / not holding) after the grasp attempt. 
