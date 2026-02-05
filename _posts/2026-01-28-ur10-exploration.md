---
title: "Transitioning to the UR10"
date: 2026-01-28 10:00:00 +0000
categories: [Project Updates, Hardware Testing]
tags: [ur10, urx, rtde, failure-detection]
author: myuwaishin
pin: false
last_modified_at: false
---


Came in early to discuss my progress and the Ned2 gripper feedback issue with my supervisors. They recommended switching to the **UR10 CB-Series** available in the lab and taking a more experimental approach.

**Their advice:**
- Run multiple pick-and-place trials to observe and document failure modes
- Collect data on how failures occur and what triggers them
- Use this empirical data to develop targeted recovery strategies

Failure recovery research is surprisingly scarce despite being critical for any company deploying robotic arms in logistics or assembly. Even just collecting failure data would be a valuable contribution to the field.

## Getting Started with the UR10

I started familiarizing myself with the hardware by going through the [UR10 CB Series User Manual](https://www.manualslib.com/manual/2028124/Universal-Robots-Ur10-Cb3.html) and the [Polyscope teach pendant tutorials](https://academy.universal-robots.com/video-tutorials/#tab-329421).

**Problem discovered:** The CB3.1 Polyscope interface for UR10 CB-Series (2019) doesn't expose force or torque data from the gripper joints unlike Polyscope X for E-Series (2020 onwards). Without this feedback, I can't monitor whether an object is actually held throughout the manipulation task.

## Gripper Control Workaround

Since direct gripper state feedback wasn't available, I implemented a toggle-based control system:

**Implementation:**
1. Created two `.urp` program files on the teach pendant:
   - `close_gripper.urp` - commands full gripper closure
   - `open_gripper.urp` - commands full gripper opening
2. Used the **URX Python library** to execute these programs remotely
3. Established communication via **RTDE (Real-Time Data Exchange)** protocol over TCP for synchronized robot control

This gave me programmatic gripper control even without direct joint feedback.

## Testing Failure Detection

The fundamental question: **How does the robot even know it failed?** Before developing recovery strategies, I need reliable failure detection.

**Approach - Intentional Failure Method:**
1. Robot executes a grasp
2. I manually pull the object away (simulating a drop/slip)
3. Monitor gripper width - if it decreases after object removal, the change indicates loss of grasp

**Results:** Only worked 2 out of 5 times. Either the detection logic needs refinement or this method just isn't reliable enough with the current hardware constraints.

**Possible solution:** Migrate to ROS 2 for better sensor access and faster control loop response times.

## Current Roadblocks

**1. Camera occlusion:** The UR10's size creates a major problem - mounting a camera high enough to see the workspace means the robot's own arm blocks the view when reaching for objects. This would require an **eye-in-hand camera** mounted directly on the end effector.

**2. Time constraints:** Operating the UR10 is significantly slower than the Ned2. The handling effort and operational overhead make me question whether I can realistically collect enough experimental failure data in the remaining project timeline.

## Next Steps

1. Explore integrating a **tactile sensor** on the Ned2 as an alternative feedback source
2. Investigate **ROS 2** control for both robots to access more comprehensive sensor data and improve control responsiveness

---

*Posted Jan 28, 2026*