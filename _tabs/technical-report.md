---
layout: page
title: Technical Report
icon: fas fa-file-pdf
order: 1
toc: true
---

## Full Technical Documentation

This page provides an overview of the technical report, which documents the design, implementation, and evaluation of a closed-loop grasp failure detection and recovery system for a robotic arm pick-and-place pipeline.

Find the technical report below:

> [Technical Report on Failure Detection and Recovery in Robotic Arm Grasp PDF](https://1drv.ms/b/c/4543b91bdec2fa57/IQDyrMlXL9WvTostCgzS7sBeAT9ytM7OswPYMdmN4Izg5B0?email=myuwaishin10%40gmail.com&e=eJ7ecy)  
{: .prompt-info }

---

### Project Overview

**Title:** Failure Detection and Recovery in Robotic Arm Grasp  
**Programme:** BEng Mechatronics and Robotics, Major Project (PDE3823)  
**Institution:** Middlesex University London  
**Academic Year:** 2025 - 2026

The report investigates how a UR10 robot with an OnRobot RG2 rigid parallel-jaw gripper can detect missed, misaligned, and slipped grasps, then recover autonomously. The system combines binary gripper contact and jaw-width feedback with YOLO26n visual classification from a wrist-mounted eye-in-hand camera.

---

## Key Findings

**Failure Detection Accuracy:** 92% using a two-layer detection approach  
**Recovery Success Rate:** 93% autonomous recovery completion  
**Navigation Accuracy:** 92% using ArUco marker-based positioning  
**Visual Classifier:** 92% accuracy and 100% recall on the holding class  
**Visual Classification Confidence:** Near 100% at the 400 mm clearance height

---

## System Architecture

The complete pipeline integrates:

- **Hardware:** UR10 CB-Series robotic arm, OnRobot RG2 parallel-jaw gripper, and OAK-D Lite stereo RGB-D camera
- **Control:** Python sockets, URScript, dashboard commands, and robot state feedback
- **Vision:** YOLO26n detection and classification, ArUco PnP, OpenCV, and DepthAI
- **Feedback:** RG2 DI8 contact signal, AI2 jaw-width feedback, and continuous visual monitoring

---

## Research Questions Addressed

1. **How does a robot know it has failed to grasp an object?**  
   → Two-layer detection using gripper feedback and visual classification

2. **What happens when a robot fails?**  
   → Systematic categorization across slip, shift, collision, and workspace violation modes

3. **How can it recover autonomously?**  
   → Adaptive recovery strategies triggered by failure mode detection

---

## Resources

- **Project Repository:** [github.com/MyuWaiShin/Final_Year_Project_2026](https://github.com/MyuWaiShin/Final_Year_Project_2026)
- **Literature Review:** [Literature Review Page](/final-year-blog/literature-review/)
- **Weekly Progress:** [Weekly Progress](/final-year-blog/weekly-progress/)
- **All Posts by Category:** [Categories](/final-year-blog/categories/)
- **All Posts by Tag:** [Tags](/final-year-blog/tags/)

---

*For inquiries or collaboration, contact: [myuwaishin10@gmail.com](mailto:myuwaishin10@gmail.com)*
