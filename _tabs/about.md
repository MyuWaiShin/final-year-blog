---
layout: page
title: About
icon: fas fa-info-circle
order: 5
toc: true
---

## **About Me**

![Profile Picture](/assets/img/avatar.png){: width="200" height="200" .rounded-10 }

**Name:** Myu Wai Shin  
**Student ID:** M00964135  
**Programme:** BEng Mechatronics and Robotics Engineering  
**Institution:** Middlesex University London  
**Expected Graduation:** July 2026

I'm a final-year Mechatronics and Robotics Engineering student passionate about automation and robotics. Driven by a strong desire to develop my engineering skills, my final-year project brings together my interests in computer vision, embedded systems, and machine learning to tackle real-world challenges in robotic manipulation.

### Contact

- **Email:** [myuwaishin10@gmail.com](mailto:myuwaishin10@gmail.com)
- **GitHub:** [github.com/MyuWaiShin](https://github.com/MyuWaiShin)
- **LinkedIn:** [Myu Wai Shin](https://www.linkedin.com/in/myu-wai-shin-0bb3b2295/)

---

## **About the Project**

### Failure Detection and Recovery in Robotic Arm Grasp

### The Research Questions

This project investigates three fundamental questions:

1. **How does a robot know it has failed to grasp an object?**  
   Exploring multi-modal failure detection using gripper feedback, force sensors, and vision systems.

2. **What happens when a robot fails?**  
   Identifying and categorizing different failure modes (slips, shifts, collisions, workspace violations).

3. **How can it recover autonomously?**  
   Developing intelligent recovery strategies that adapt based on the type of failure detected.

### Background & Motivation

> For a deeper dive into existing research on robotic failure recovery, see the [Literature Review](/final-year-blog/literature-review/) page.  
> *(Note: Content is currently being developed)*

In real-world manufacturing environments, robotic failures are inevitable. Objects may be positioned inconsistently, environmental conditions vary, and mechanical tolerances introduce uncertainty. The ability to detect and recover from failures autonomously is crucial for truly reliable robotic systems.

---

## The Approach

![Approach Flowchart](/assets/img/approach_flowchart.png){: .shadow }
*Figure: Hybrid failure detection and recovery system workflow*

### 1. Failure Detection

**Multi-Modal Sensing:**
- **Gripper feedback**: Force/torque sensors and gripper width monitoring
- **Vision feedback**: Real-time object detection and pose estimation
- **Sensor fusion**: Combining multiple inputs for robust failure detection

**Failure Modes Under Investigation:**
- Object slips during grasp
- Object shifts or movements
- Collisions during approach
- Speed-related failures
- Pick angle variations

### 2. Recovery Strategies

#### Rule-Based Recovery (Fast & Reliable)
Predefined recovery procedures for common, predictable failure scenarios, enabling fast and reliable responses through decision-tree logic and known failure patterns.

#### Learning-Based Grasp Refinement (Adaptive)

- **GraspNet integration**: predicting alternative grasp poses when initial attempts fail
- **Success prediction**: training a lightweight classifier to evaluate "likely to succeed" vs "likely to fail" across different grasp configurations

#### Advanced Recovery Techniques

- **Object repositioning**: using the gripper to push or nudge objects into more favourable poses
- **Adaptive parameters**: slowing down retry attempts and adjusting approach trajectories rather than repeating identical motions

### 3. Data Collection & Analysis

A key contribution of this research is building a comprehensive failure mode dataset containing:
- Images of object poses before and after failures
- Sensor readings (force, gripper width, position)
- Success/failure labels and failure type classifications
- Recovery attempt sequences and outcomes

This dataset will allow us to identify which factors most affect success rates and provides a valuable resource for future research.

### 4. Benchmarking

To evaluate the effectiveness of intelligent recovery, I compare three different approaches:

1. **No Recovery (Baseline)**: the system stops after the first failure
2. **Simple Retry**: the same grasp is retried using the same pose
3. **Intelligent Recovery**: recovery strategies are selected based on the detected failure type

**Metrics:**
- Success rate (%)
- Recovery time (seconds)
- Number of attempts before success
- Strategy effectiveness across different failure modes

---

## Hardware & Software

### Hardware Setup
- **Vision**: Depth camera for object detection and pose estimation
- **Manipulator**: UR10 CB Series robotic arm
- **End Effector**: Gripper with force feedback

### Software Stack
- **Object Detection**: YOLO for real-time object recognition
- **Robot Control**: UR Python libraries
- **Simulation**: URSim (Polyscope virtual environment)
- **Grasp Planning**: GraspNet for grasp pose prediction
- **Data Analysis**: Python (NumPy, Pandas, Matplotlib)

---

## Scope & Limitations

**What This Project Covers:**
- Failure detection and recovery in robotic arm grasp operations
- Hybrid approach combining rule-based and learning-based methods
- Comprehensive data collection and benchmarking

**What's Outside the Scope:**
- **No mechanical fault detection**: The investigation focuses solely on manipulation failures, not hardware malfunctions
- **Controlled environment**: Experiments are conducted in a laboratory setting with known objects
- **Workspace constraints**: The system identifies irrecoverable scenarios (objects out of reach, insufficient clearance for arm movement)

---

*Last Updated: January 30, 2026*