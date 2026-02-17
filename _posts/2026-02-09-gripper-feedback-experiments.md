---
title: "Gripper-Based Failure Detection Experiments I"
date: 2026-02-09 14:00:00 +0000
categories: [Project Updates, Hardware Testing]
tags: [gripper, failure-detection, rg2, ur10, calibration]
author: myuwaishin
pin: false
last_modified_at: false
---

Today I ran extensive experiments to detect pick-and-place failures using only gripper feedback — no vision involved yet. The goal: how does the robot know if it has grabbed something just from force sensing and gripper width?

## Experimental Setup

I tested preprogrammed pick-and-place poses with controlled variations:

**Variables I controlled:**
- Object presence (object vs. no object at pick location)
- Object orientation (cubes, cylinders, arcs at different angles)
- Position offset (slightly shifted from exact preprogrammed location)

**Objects:** Lightweight styrofoam cubes, cylinders, and arcs

---

## Understanding the RG2 Gripper

First, I studied the [RG2 datasheet](https://www.universal-robots.com/media/1226143/rg2-datasheet-v14.pdf) to understand what feedback I could get.

### Digital I/O Feedback

![RG2 Digital Feedback](/assets/img/RG2_DI8_feeback.jpg){: .shadow }
*Digital status pins from RG2 datasheet*

**DI8 (Force Reached):** Goes HIGH when the gripper senses force
- LOW = No resistance (gripper moving freely)
- HIGH = Force detected (gripping something or jaws touching)

**DI9 (Gripper Busy):** Indicates gripper is in motion

### Analog Feedback - Gripper Width

![RG2 Analog Feedback](/assets/img/RG2_AI2_feedback.jpg){: .shadow }
*Analog width measurement from AI2 pin*

**AI2 (Gripper Width):** Outputs voltage (0-3.7V) corresponding to gripper width (0-110mm)

Using the 0V:5V mode, the voltage maps to width via:
```
actual_width = (voltage / 3.7V) × 110mm
```

---

## Step 1: Remote Gripper Control

The first step I did was to connect to the UR10 Dashboard Server and load preprogrammed `open_gripper.urp` and `close_gripper.urp` that I created on the teach pendant and trigger them remotely via Python. I kept the `close_gripper.urp` running even after the object has been picked up, so the gripper would still close if the object dropped, causing the width (AI2) to change.

**Script:** `dashboard_gripper.py` in [UR10 folder](https://github.com/MyuWaiShin/Final_Year_Project_2026/tree/main/UR10)

**Purpose:** Basic connection and gripper control

---

## Step 2: Recording Pick Positions

Now that I have the gripper control working, I need to record the positions the robot will move to. I used the `read_robot_frame.py` script to read the TCP (Tool Center Point) coordinates of the robot.

**Script:** `read_robot_frame.py` in [UR10 folder](https://github.com/MyuWaiShin/Final_Year_Project_2026/tree/main/UR10)

**Purpose:** Read real-time TCP (Tool Center Point) coordinates

**Method:**
- Connected to port 30002 for binary packet parsing
- Manually jogged robot to each position
- Ran script to read and record coordinates

**Positions recorded:**
- **Pick 1:** (-0.2606, -1.1581, -0.4592),  # Cube 1
- **Pick 2:** (-0.0606, -1.1581, -0.4592),  # Cube 2 (X + 200mm)
- **Pick 3:** (0.1394, -1.1581, -0.4592),   # Cube 3 (X + 400mm)
- **Place location:** (0.7952, -0.8659, -0.3915)
- **Home position:** (-0.0645, -0.7940, 1.0128)

<div style="text-align: center;">
  <img src="/assets/img/pick_place_home.png" alt="Pick, Place, Home Positions" style="width: 60%; display: block; margin: 0 auto; box-shadow: 0 4px 6px rgba(0,0,0,0.1);">
  <p style="font-size: 0.9rem; color: #888; margin-top: 0.5rem;"><em>Pick, Place, Home Positions</em></p>
</div>

---

## Step 3: Gripper Logic and Calibration

In this step, I integrated object picked detection into the gripper control. When testing the width of gripper when it's fully closed and when object is between the jaws, I noticed that the raw width measurement from the voltage reading of the AI2 pin and the actual width of the gripper jaws are different. I needed to calibrate the raw gripper width readings to actual measurements.

**Script:** `gripper_with_detection.py` in [UR10 folder](https://github.com/MyuWaiShin/Final_Year_Project_2026/tree/main/UR10)

**Purpose:** Gripper control with object picked detection

### Calibration Process

**Measured values:**
- Gripper fully open: Raw = 65.8mm, Actual = 91mm
- Gripper fully closed: Raw = 8.5mm, Actual = 10.5mm

### Linear Interpolation

By assuming that the relationship between raw and actual is a straight line, I used the [linear interpolation formula](https://en.wikipedia.org/wiki/Linear_interpolation):

```
y = y₁ + (x - x₁) × (y₂ - y₁) / (x₂ - x₁)
```

(rearranged from y = mx + b)

**Step 1: Calculate the slope**

```
slope = (y₂ - y₁) / (x₂ - x₁)
slope = (91.0 - 10.5) / (65.8 - 8.5)
slope = 80.5 / 57.3
slope ≈ 1.405
```

The slope represents how much the actual width changes per unit of raw width change.

**Step 2: Calculate the offset**

```
offset = y₁ - (x₁ × slope)
```

Using the closed position calibration point (x₁ = 8.5, y₁ = 10.5):

```
offset = 10.5 - (8.5 × 1.405)
offset = 10.5 - 11.94
offset ≈ -1.44
```

Interpretation: The calibration line has a negative y-intercept, indicating the sensor has a constant negative bias.

**Step 3: Final calibration formula**

Combining slope and offset using the linear equation y = mx + b:

```python
actual_width = (raw_width * slope) + offset
actual_width = (raw_width * 1.405) - 1.44
```

### Detection Logic

The key insight: combine force sensing with width measurement to detect if object is actually grasped

**No object detected:**
- Force sensor (DI8) = LOW
- Width < 11mm
- *Interpretation:* Jaws touching each other, no object between them

**Object detected:**
- Force sensor (DI8) = HIGH
- Width > 12mm
- *Interpretation:* Gripper closed on something with thickness

**Still moving:**
- Force sensor (DI8) = LOW
- *Interpretation:* Gripper hasn't reached target yet

---

## Technical Implementation

**Communication:** TCP/IP sockets with URScript command injection

**Data parsing:** Binary packet parsing using Python's `struct` module

**Threading:** 
- Daemon thread for continuous feedback monitoring
- Temporary threads for parallel monitoring during operations

**Timing:** Hardcoded sleep delays calibrated for robot movement completion

---

## Next Steps

Now that I have reliable gripper-based detection working, the next phase is:

1. Implement full pick-and-place with gripper-based detection logic
2. Integrate vision system for pre-grasp pose estimation

The gripper feedback alone is surprisingly effective for detecting failures. The combination of force sensing and calibrated width measurement gives clear signals about grasp success.

---

**Code Repository:** All scripts mentioned in this post are available in the [UR10 folder](https://github.com/MyuWaiShin/Final_Year_Project_2026/tree/main/UR10) of my [project repository](https://github.com/MyuWaiShin/Final_Year_Project_2026).
