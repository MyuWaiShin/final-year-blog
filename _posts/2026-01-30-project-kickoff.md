---
title: "Getting Started with the Niryo NED2"
date: 2026-01-26 04:17:00 +0000
categories: [Project Updates, Hardware Testing]
tags: [niryo, ned2, gripper, calibration]
author: myuwaishin
pin: true
last_modified_at: false
---

## Follow Along

In this project, I am investigating failure detection and recovery methods for when a robotic arm fails to grasp an object. This blog documents my research, experiments, and findings along the way.

## Project Structure

I've split my project into three layers to keep things organized:


![Project Pipeline](/assets/img/project_pipeline.png)
_Flowchart created using [Miro](https://miro.com/app/dashboard/)_

**1. Perception and Configuration**
- Object detection and pose estimation
- Camera-to-robot frame transformation (hand-eye calibration)

**2. Failure Detection**

These are the failure types I'm looking into:
- **Position error**: Did the gripper miss the object during the approach?
- **Slips**: Did the object slide out during transport?
- **Collisions**: Did the gripper bump into something?

How I'm planning to detect them:
- **Gripper width feedback**: Can we tell if it actually picked something up?
- **Tactile sensor**: Is the object still in the gripper throughout the movement?
- **Visual sensor (depth camera)**: Was the object lifted and then dropped? (checking for changes in z-axis position)

**3. Recovery Methods**
- How do we recover autonomously depending on the failure type?
- How do we teach the robot to recover?
- Which recovery method works best for which failure?
- Compare the methods and do analysis

## What I Did on January 26th

### Reading Documentation

I went through the Niryo NED2 documentation to understand what I'm working with:
- [NED2 Robot Documentation](https://docs.niryo.com/robots/ned2/)
- [PyNiryo Python Library](https://niryorobotics.github.io/pyniryo/v1.2.3/index.html)
- [NED2 Manual](https://www.manualslib.com/manual/2855549/Niryo-Ned2.html)
- [Vision-Based Pick and Place Tutorial](https://academy.niryo.com/mod/page/view.php?id=235)

### Hands-On with NED2

**Camera Calibration**

I calibrated the camera to the robot frame using the eye-in-hand setup with the [2D camera kit](https://docs.niryo.com/accessories/vision-set/) that came with NED2. Here's what I did:
1. Selected a few points on the table
2. Used freehand movement to navigate the robot to each point and saved the position
3. Repeated this at least 4 times to get enough point correspondences

After calibrating, I ran a simple pick and place from point A to point B to test it out.

**Gripper Testing**

This is the critical part for failure detection. I need real-time feedback from the gripper to detect when a grasp fails. I tested what sensor data the NED2 gripper can provide:

- **Width feedback**: Can I read the current gripper opening distance to detect object presence?
- **Torque/force feedback**: Can I measure grip force to detect slips or verify grasp stability?

I also wrote test scripts using the PyNiryo NED2 API to see if I could programmatically control the gripper width—setting it to specific opening distances rather than just binary open/close commands.

**What I Found:**

The NED2 gripper operates using a resistance-based closing mechanism. When commanded to close, it continues closing until it encounters resistance (contact with an object), then stops. The limitations I discovered:

- **No torque or force feedback**: The API doesn't expose any force sensor data, so I can't measure grip strength or detect when an object is slipping
- **No width control**: I can't command specific opening distances (e.g., "open to 40mm"). Only binary commands: fully open or close until resistance
- **No width reading**: I can't query the current gripper position, so I can't monitor changes during transport

The gripper does provide basic binary feedback, i.e., if it closes fully, nothing was grasped. If it stops partway, something was encountered.

**The Problem:**

While I can detect initial grasp success (gripper stopped at some width vs. closed fully), **I cannot detect if the object drops during transport.** The gripper maintains its position after stopping even if the object falls out, the fingers stay at that width. From the robot's perspective, the gripper state looks identical whether it's holding the object or not.

## Key Takeaway

This is a valuable finding because now I know what I'm dealing with. Getting tool feedback is going to be harder than expected with NED2.

Next steps:
- Dig deeper into NED2's capabilities
- Explore other robots in the lab that might give better tool feedback

## Time Concerns

I'm a bit anxious about the timeframe. My project needs a lot of data collection and experimental runs to identify potential failures, so I'm worried about getting valuable results within the deadline.

To tackle this, I'm making a Gantt chart ASAP to plan out the entire project so I can follow it without panicking.

See my [Gantt chart](/final-year-blog/gantt-chart/) for the full breakdown.

---

**Next:** Exploring gripper alternatives and diving deeper into sensor options.
