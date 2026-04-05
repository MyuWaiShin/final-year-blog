---
title: "Final Camera Mount and Calibration Session"
date: 2026-03-16 12:00:00 +0000
categories: [Project Updates, Calibration]
tags: [calibration, camera-mount, hand-eye, charuco, pose-estimation]
author: myuwaishin
pin: false
last_modified_at: false
---

> **Note:** This post is written in April, backdated to the session date of 16 March.
{: .prompt-info }

I got the camera mounts. I tested all of them and luckily one of them worked. 

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/cam_mount_4_2_final.png" alt="Final camera mount render" style="display:block; margin:0 auto; width:60%; border-radius:4px;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Final camera mount design</em></figcaption>
</figure>


The one with the 21 degree tilt angle worked well.The camera had a clear view of the gripper and the workspace. The tilt angle and distance from TCP to camera were also easier to measure physically with this design compared to earlier versions.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/cam_mount_final.JPG" alt="Failure detection pipeline flowchart" style="display:block; margin:0 auto; width:90%; border-radius:4px;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Final camera mount technical drawing</em></figcaption>
</figure>


---

## Calibration with a Wider ChArUco Board

I ran calibration again using the same method from the last session. And partway through I decided to switch to a wider board by using an A3 ChArUco board to get more coverage across the frame and improve accuracy.

After calibration, I tested by moving the robot to an ArUco tag on the workspace. The robot hovered above the tag and was close, and sometimes it would land right on the tag. But getting the TCP to land precisely at the centre was difficult. There was a consistent lateral offset at different ranges.

---

## Measuring the Physical Offset

The calibration estimated a 23° tilt and the camera sitting approximately 8 mm to the right of the TCP centre line. The camera is mounted along the centre of the TCP, so it should not be 8 mm off unless I accidentally shifted off-centre when I designed the mount.

Because of this I wanted to check whether the calibrated transform was less accurate than the one based on physical measurements. I measured the camera offset manually using a ruler and the known tilt angle of 21°, then used those values as the transform directly, replacing the calibrated one, and tested moving to the ArUco tag.

Both gave fairly the same result. There was lateral inaccuracy with both transforms, and it appeared at different ranges at different points in the workspace.

I cannot tell if the lateral shift is coming from the calibration or not because I genuinely am not sure. Getting the exact pose is just difficult, the system can get close but cannot reliably land precisely at the centre every time. For general pick and place this is acceptable, but for precision picking where the TCP needs to land at an exact point, it is not reliable enough.


---

## Workaround

The workaround I am thinking about is running two pose estimates instead of one which involves taking a second estimate closer to the object before descending to correct for any residual lateral offset at the last moment. This still needs to be tested.

---

## Current Project Status

Although not perfect, I think the project is in a good state. I now have all the necessary components to build the final pipeline.

| Component | Status |
|---|---|
| YOLO object detection model | Done ✓ |
| Detection adjustments (OBB, orientation estimation) | Done ✓ |
| Stereo depth estimation | Done ✓ |
| Stage 1 failure detection (gripper feedback) | Done ✓ |
| Stage 2 failure detection (CLIP visual classifier) | Done ✓ |
| Failure detection integration | Done ✓ |
| Slip detection | TBC |
| Hand-eye calibration | Done ✓ |
| Camera mount | Done ✓ |
| Navigation and robot motion planning | Done ✓ |
| Full recovery loop integration | In progress |

---

## Next

The next step is implementing two-point pose estimation that involves re-estimating the object position at a closer height before descending to pick. I also need to make the TCP approach vertically above the object regardless of camera tilt, and rotate the gripper to match the object's orientation from the YOLO OBB output.
