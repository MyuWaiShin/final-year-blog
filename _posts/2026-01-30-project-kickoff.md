---
title: "Project Kickoff: Failure Detection and Recovery in Robotic Pick-and-Place Systems"
date: 2026-01-26 04:17:00 +0000
categories: [Project Updates, Planning]
tags: [ur10, robotics, research, setup, methodology]
author: Myu Wai Shin
pin: true
---

## Welcome to My Final Year Project!

I'm excited to share my journey through this final year project, which focuses on a challenge that's been bugging me since I started working with robotic arms: **what happens when a robot fails to grasp something?**

Most of the time, the robot either keeps going like nothing happened (which causes problems later) or just stops and waits for someone to fix it. Neither option is great, especially in industrial settings where time is money.

## The Big Question

Here's what I'm trying to figure out: **Can we make robots smart enough to realize they've messed up and fix the problem themselves?**

This breaks down into three main questions:
1. How does a robot know it failed to grab something?
2. What exactly went wrong?
3. How can it recover without human help?

## Why This Matters

I've been reading about manufacturing facilities where a single robot failure can halt an entire production line. Workers have to stop what they're doing, walk over, figure out what went wrong, and manually fix it. This happens way more often than you'd think, and it's expensive.

If robots could handle these failures on their own, companies could save time and money while making their automation more reliable. That's the goal here.

## The Approach: Hybrid Recovery

After doing some initial research and testing with our UR10 arm, I've settled on what I'm calling a **hybrid approach**. Here's the idea:

### Rule-Based Recovery (The Fast Response)
For common failures that happen all the time—like the gripper slipping or the object shifting slightly—I'm building a set of predefined recovery procedures. Think of it like muscle memory: the robot recognizes the situation and immediately knows what to do.

### Learning-Based Refinement (The Smart Adjustment)
For more complex situations, I'm integrating **GraspNet**, which can predict alternative ways to grab an object. If the first attempt fails, the system can suggest different grasp poses to try. I'm also training a simple classifier to predict whether a particular grasp is likely to succeed or fail before even attempting it.

The beauty of this approach is that I don't need to dive into full reinforcement learning (which would take forever and require complex simulation setups). Instead, I'm using supervised learning on real data I collect during experiments.

## What I'm Testing

I'll be running experiments to induce different types of failures:
- **Slips**: Object slides out of the gripper
- **Shifts**: Object moves during the approach
- **Collisions**: Gripper bumps into something
- **Speed issues**: Moving too fast or too slow
- **Angle problems**: Approaching from the wrong direction

For each failure type, I'll test different recovery strategies and see what works best.

## The Hardware & Software Stack

**Hardware:**
- UR10 CB Series robotic arm (the workhorse)
- Gripper with force feedback sensors
- Depth camera for vision (object detection and pose estimation)

**Software:**
- YOLO for detecting objects in real-time
- UR Python libraries for robot control
- GraspNet for grasp planning
- URSim for testing things safely in simulation first

## Building a Dataset

One thing I'm particularly excited about is building a comprehensive **failure mode dataset**. Every time the robot fails, I'll capture:
- Images of the object before and after
- Sensor readings (force, gripper width, position)
- What recovery strategy was used
- Whether it worked

This dataset could be useful for other researchers working on similar problems. Plus, it'll let me do statistical analysis to figure out which factors actually matter for success.

## Benchmarking: Proof It Works

To prove this approach actually helps, I'm comparing three scenarios:

1. **No Recovery**: Robot stops on first failure (baseline)
2. **Simple Retry**: Robot tries the exact same grasp again
3. **Intelligent Recovery**: Robot uses my hybrid system to adapt

I'll measure success rates, recovery time, and how many attempts it takes before succeeding. The goal is to show that intelligent recovery significantly outperforms the alternatives.

## What's Next?

Over the next few weeks, I'll be:
- Finishing up the literature review (due mid-February)
- Getting the hardware fully calibrated and tested
- Setting up the vision system
- Running initial failure experiments

I'll be posting weekly updates here documenting progress, challenges, and interesting findings. Expect lots of videos, graphs, and probably some funny failure cases.

## The Timeline

Here's the rough plan (check out the [full Gantt chart](/final-year-blog/gantt-chart/) for details):
- **February**: Literature review, system setup
- **March**: Failure mode identification
- **April**: Recovery strategy development, data collection
- **May**: Analysis and benchmarking
- **June**: Thesis writing
- **July**: Final revisions and submission

## Challenges I'm Expecting

Let's be real—there are some things I know will be tough:

**Time Constraints**: I've got about 5 months to do everything, so I'm being strategic about what to prioritize. That's why I'm skipping reinforcement learning—it's cool, but sim-to-real transfer alone could eat up months.

**Scope Management**: I'm focusing on pick-and-place failures only, not mechanical faults or hardware issues. The robot also can't magically reach objects outside its workspace, so I need to identify when a failure is truly unrecoverable.

**Data Quality**: Getting clean, consistent data from real-world experiments is always tricky. I'll need to be careful about calibration and validation.

## Want to Learn More?

If you're interested in the technical details, check out:
- [About the Project](/final-year-blog/about/) - Full methodology and background
- [Resources](/final-year-blog/resources/) - Tutorials and tools I'm using
- [Literature Review](/final-year-blog/literature-review/) - Academic research (coming soon)

---

Thanks for following along! I'll be posting weekly updates as the project progresses. Feel free to reach out if you have questions or suggestions—I'm always happy to chat about robotics.

**Next week:** Setting up the vision system and running initial calibration tests.
