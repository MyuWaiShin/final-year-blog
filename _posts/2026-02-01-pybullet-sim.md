---
title: "Day 4: Moving Complex Failures to Simulation"
date: 2026-02-01 14:00:00 +0000
categories: [Project Updates, Simulation]
tags: [pybullet, simulation, ned2, rl, failure-recovery]
author: myuwaishin
pin: false
last_modified_at: false
---

## Research: Hybrid Action Spaces

I've been looking into how to handle failures where simple retries won't work. I found some interesting research on **RecoveryChaining** which uses a "Hybrid Action Space."

The core idea is to combine:
1.  **Primitive Actions:** Basic robot movements.
2.  **Nominal Options:** Temporally extended actions (like "pick object") that hand over control to a standard model-based controller.

During exploration, the agent checks in simulation if a task can be solved reliably. If a failure is detected (like a collision), the system learns a **recovery policy** to not just fix the immediate error, but to guide the robot back to a state where the standard controller can take over again.

<div style="text-align: center;">
  <img src="/assets/img/recovery_chaining.png" alt="Recovery Chaining Concept" style="width: 90%;" />
  <p style="font-size: 0.85em; color: #6c757d; margin-top: 0.5em;">
    <em>Visualizing Nominal Execution vs. Recovery Actions (Source: Research Reference)</em>
  </p>
</div>

## Setting up PyBullet

Since this requires training an RL (Reinforcement Learning) policy, I can't do it comfortably on the physical hardware yet. I decided to set up a simulation environment using **PyBullet**.

**Setup Steps:**
1.  Installed PyBullet via the [Quick Start Guide](https://github.com/bulletphysics/bullet3/tree/master).
2.  Imported the **Ned2 URDF model** from the official [Niryo ROS repository](https://github.com/NiryoRobotics/ned_ros/tree/master/niryo_robot_description/urdf/ned2).
3.  Loaded a simple cube URDF into the environment.

## The Wiggling Robot Issue

I wrote a python script based on online examples to attempt a simple pick-and-place operation in the sim.

**Current Status:** The robot is unstable in simulation - there's excessive wiggling and oscillation during movement.

I suspect it's an issue with the joint damping or the control loop frequency in my script. I need to debug the physics parameters in the URDF or check my motor control logic.

*Work in progress...*
