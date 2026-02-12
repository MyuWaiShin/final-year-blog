---
title: "Class Insights - Observing Failure Modes in Pick and Place Manipulation"
date: 2026-02-04 09:00:00 +0000
categories: [Project Updates, Research]
tags: [failure-detection, recovery, manipulation, ACT, diffusion, lerobot]
author: myuwaishin
pin: false
last_modified_at: false
---

This morning's class was incredibly insightful. It was a cross-disciplinary session bringing together students from Design Engineering, Product Design, Electronics Engineering, and Robotics Engineering to discuss our projects.

## The One-Page Overview Exercise

We were tasked with creating a rough sketch and one-page summary of our projects, what I'm doing, why it matters, and how it's feasible within the 12-week timeframe.

![Class Exercise](/assets/img/one_page_overview.jpg){: .shadow }
*One-page project overview from the class session*

## Owen's Dual-Arm Setup: A Case Study in Failure

The real insight of the session was observing Owen's dual robotic arm system he built for his final year project. He trained his setup using:
- **ACT (Action Chunking with Transformers)** and **Diffusion Policy** on pick and place task
- **VR-based teleoperation** for data collection
- **LeRobot framework** from Hugging Face for training the model

His task was a pick-and-place with transfer: one arm picks up an object, transfers it to the second arm, and the second arm places it down.

### Performance Observations

The system performed well in the trained environment with known objects. However, when tested on zero-shot backgrounds (untrained scenarios), I observed two distinct failure modes:

#### Failure Type 1: Pose Calculation Errors
The arm detects the object but fails to grasp it at the exact position. The gripper closes at the wrong location, missing the object.

**Root Cause:** Inaccurate camera sensor data or pose estimation errors.

**Recovery Question:** How can the system recover from incorrect pose calculations?

Potential approaches:
- **Multi-attempt grasping** with slight pose adjustments
- **Real-time pose refinement** using visual servoing during approach
- **Uncertainty planning** that triggers re-detection when confidence is low

#### Failure Type 2: Gripper Orientation Mismatch
The arm approaches the object but the gripper orientation doesn't align with the object's orientation, making it impossible to grasp.

**Root Cause:** Although the policy accounts for variations in object orientation through trained data, it does it poorly in zero-shot scenarios.

**Recovery Question:** How can the system adapt its gripper orientation?

Potential solutions:
- **6D pose estimation + geometric planning:** First estimate the object's pose, then compute grasp poses using analytical method
- **GraspNet-based grasp generation:** Train an end-to-end network (e.g., GraspNet-1Billion) that directly predicts viable 6-DoF gripper poses from point clouds, inherently accounting for object geometry and orientation
- **Adaptive re-grasping:** Use visual feedback during approach to adjust gripper orientation in real-time

### The Critical Component

The key problem was that the robot doesn't know it has failed. Even when the first arm fails to grasp the object, it continues the transfer motion to the second arm although it is transferring nothing.

This observation reinforces the core value of my research:

> **Robots need to detect when they've failed and autonomously recover.**

Without failure detection:
- Tasks become inefficient and time-consuming
- Unnecessary human intervention is required for debugging
- The system cannot operate autonomously in real-world scenarios

## Connection to My Project

This real-world observation validates the significance of failure detection and recovery in robotic manipulation. My project aims to address exactly this gap:

1. **Detect failures** from gripper and visual feedbacks
2. **Trigger recovery behaviors** when failures are detected
3. **Skip unnecessary action sequences** if a critical step fails

## Next Steps

Based on these insights, my immediate plan is:

1. **Experiment with the UR10** by introducing controlled variations (object position, orientation, speed, etc.)
2. **Detect failed grasps** using gripper force/current feedback
3. **Implement recovery logic** to skip subsequent actions when a grasp fails

Observing Owen's robot in action allowed me to see different failure modes that I need to address in robotic arm pick and place manipulation, which is very useful for my project.

---

*For my full project overview, see the [Project Timeline](/final-year-blog/project-timeline/) page.*
