---
title: "Camera Configuration: External Mount vs Eye-in-Hand"
date: 2026-02-21 12:00:00 +0000
categories: [Project Updates, Design Decisions]
tags: [robotics, vision, ur10, failure-detection, perception]
author: myuwaishin
pin: false
---

Before building the failure detection pipeline, I had to decide where the camera would actually sit. This is a fundamental system design decision that directly affects how reliably I can detect whether a grasp has succeeded or failed.

## The Initial Plan: External Mount and Height Tracking

My first approach was to mount a camera externally, viewing the workspace from the side. The failure detection logic was based on tracking the target object's vertical position before and after a lift.

1.  **P1:** Before picking, the camera tracks the object's 3D position, specifically its height (Z coordinate) in the scene.
2.  **P2:** After the robot executes the lift, the camera re-checks the object's height.

The decision logic was simple: if the object's height increased significantly (P2 > P1), the pick succeeded. If the height remained unchanged, the robot missed or dropped the object. However, this approach introduced significant reliability issues when tested in practice.

<p style="font-size:0.82rem; color:#7c3aed; font-style:italic;">(P1 tracking picture to be added)</p>

## Why the External Mount Failed in Practice

While testing the P1/P2 height tracking method, I encountered several critical issues that made the external mount unreliable for this application:

*   **Occlusion:** Because the UR10 is a large robot arm, it frequently blocked the camera's view of the object during the pick. When the tracking is lost due to occlusion, the system cannot reliably compare P1 and P2.
*   **Non-Modularity and Safety:** The UR10 is powerful and has a large reach. Having an external camera hanging over the workspace makes the setup less modular. There is even a risk of the robot hitting the camera mount if the boundaries are not perfectly defined in the safety configuration.
*   **Verification Delays:** The most reliable way to confirm a successful grasp is a binary classifier that checks whether the object is visible between the gripper fingers directly after lifting. With an external mount, the robot would need to move the gripper near the camera's field of view for verification. This adds an unnecessary extra motion that increases cycle time and complexity.

## What About a Bird's Eye View?

Another common option is a top-down, bird's eye configuration, where the camera hangs from the ceiling looking straight down over the full workspace. However, for this project, this approach introduces the same core issues.

Considering the depth error from that distance, a small vertical shift might be within the sensor's noise margin, making it hard to distinguish a successful lift. Distinguishing a successful lift from a slip becomes unreliable at that distance. Also, with the camera directly above, the robot arm will frequently pass in front of the object during the approach and pick phases, which is a well-documented problem in overhead-mounted camera setups for robot manipulation.

<p style="font-size:0.82rem; color:#7c3aed; font-style:italic;">(Bird's eye view camera configuration picture to be added)</p>

## The Decision: Eye-in-Hand Configuration

After these tests, I decided to abandon the external mount and move to an **Eye-in-Hand** configuration, where the OAK-D Lite is mounted directly on the robot wrist. Rather than pointing the camera parallel to the tool axis and viewing straight ahead, I angled it slightly downward, giving a simultaneous view of both the workspace in front and the gripper below.

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/holding_20260309_212313_859171.png" alt="Eye-in-Hand camera view" style="display:block; float:none; width:70%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Eye-in-Hand camera view showing gripper and workspace simultaneously</em></figcaption>
</figure>

This setup solves primary problems mentioned above:
*   **Modularity:** The vision system moves with the robot, making the whole cell self contained.
*   **High Precision:** The camera is much closer to the small objects, drastically reducing depth error and increasing detection reliability.
*   **Instant Grasp Verification:** Because the gripper is always in frame, the system can immediately verify whether the object is held without any additional robot motion.

## Designing the Camera Mount

I initially used an adjustable off-the-shelf mount to find the optimal camera angle before committing to a permanent design. After iterative testing, the effective tilt angle was approximately **47 degrees downward** from the wrist axis.

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/cam_angle.jpeg" alt="Eye-in-Hand camera mount angle" style="display:block; float:none; width:55%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Adjustable mount used to determine the optimal camera tilt angle</em></figcaption>
</figure>

Once the angle was confirmed, I took precise measurements of the camera position relative to the TCP (Tool Centre Point) and the tilt angle. I then modelled a custom bracket in SolidWorks to hold the camera permanently at that position.

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/camera_mount_measurement.png" alt="Eye-in-Hand camera mount measurement" style="display:block; float:none; width:70%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem; text-align:center;"><em>Camera position and offset measurement relative to TCP</em></figcaption>
</figure>

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/CameraMount2.jpg" alt="Eye-in-Hand camera mount technical drawing" style="display:block; float:none; width:70%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem; text-align:center;"><em>SolidWorks technical drawing of the 3D printed bracket</em></figcaption>
</figure>


This `.stl` file is sent for 3D printing.

Find the `.stl` file at [Camera_Mount.stl](https://livemdxac-my.sharepoint.com/:f:/r/personal/ms3433_live_mdx_ac_uk/Documents/Camera_Mount_CAD?csf=1&web=1&e=TbLIlL).

---

