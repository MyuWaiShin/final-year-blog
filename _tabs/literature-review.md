---
layout: post
title: Literature Review
icon: fas fa-graduation-cap
order: 3
toc: true
date: 2026-01-28
categories: [Literature Review]
tags: [research, papers]
pin: false
---

> **Last Updated:** 19 February 2026.

> **Status:** This page is continuously updated as I review relevant research for the project.

This section contains academic papers and research publications reviewed for my project on multi-modal failure detection and recovery in robotic arm grasp.

*This page follows the same citation format as the [technical report](https://github.com/MyuWaiShin/final-year-blog/blob/main/assets/files/Technical%20Report%20-%20Chpt%201%20%262.pdf).*

---

## **Grasp Planning & Execution**

> Understanding what conditions lead to success or failure.
{: .prompt-info }

**[3]** Bicchi, A., & Kumar, V. (2000). Robotic grasping and contact: A review. In *Proceedings of IEEE ICRA*, pp. 348–353.

<details>
<summary>Show abstract</summary>
<p>A classic review paper on how robots make contact with objects and what makes a grasp stable. Good starting point for understanding the physics behind why grasps succeed or fail.</p>
</details>

---

**[4]** Miller, A. T., & Allen, P. K. (2004). GraspIt!: A versatile simulator for robotic grasping. *IEEE Robotics & Automation Magazine*, 11(4), 110–122.

<details>
<summary>Show abstract</summary>
<p>GraspIt! is a grasp simulation tool that lets you test and score different grasp poses before running anything on the real robot. Useful background for the simulation testing approaches.</p>
</details>

---

**[5]** Mahler, J., et al. (2017). Dex-Net 2.0: Deep learning to plan robust grasps with synthetic point clouds and analytic grasp metrics. In *Proceedings of RSS 2017*.

<details>
<summary>Show abstract</summary>
<p>Trains a neural network entirely on synthetic data to pick good grasp poses, and it works on real objects. I referenced this mainly because it introduced the idea of using the previous failed grasp to plan a better next one, which is what my re-grasp recovery routine does.</p>
</details>

---

**[7]** Zeng, A., et al. (2022). Robotic pick-and-place of novel objects in clutter with multi-affordance grasping and cross-domain image matching. *International Journal of Robotics Research*, 41(7), 690–705.

<details>
<summary>Show abstract</summary>
<p>Looks at how robots can pick objects they've never seen before in a messy pile. Uses visual affordance matching across different image domains. Relevant to object detection challenges, especially when the camera view changes as the robot moves.</p>
</details>

---

**[13]** Morrison, D., et al. (2018). Closing the loop for robotic grasping: A real-time, generative grasp synthesis approach. In *Proceedings of RSS 2018*.

<details>
<summary>Show abstract</summary>
<p>Instead of planning a grasp once and executing it blindly, this paper continuously updates the grasp candidate as the robot approaches the object. The idea of checking and adjusting in real-time (rather than one-shot) directly influences how I designed the post-grasp verification step.</p>
</details>

---

**[22]** Qiao, J., et al. (2025). Multi-object robotic grasping setup based on RGB-D sensor. *Heliyon*, 11(3), e43445. [https://doi.org/10.1016/j.heliyon.2025.e43445](https://doi.org/10.1016/j.heliyon.2025.e43445)

<details>
<summary>Show abstract</summary>
<p>Uses depth cameras and neural networks to predict grasp poses for multiple objects in clutter. This paper is useful background on how deep learning models handle Oriented Bounding Boxes (OBB) for grasping, which I compared against my own lightweight OpenCV approach.</p>
</details>

---

## **Failure Detection & Multi-Modal Sensing**

> How robots detect grasp failures using different sensor combinations.
{: .prompt-info }

**[1]** Drigalski, R., et al. (2020). Robust picking via grasping under object pose uncertainty. *IEEE Robotics & Automation Letters*, 5(2), 3304–3311.

<details>
<summary>Show abstract</summary>
<p>Tackles the problem of grasping when you don't know the object's exact position and the robot has to deal with pose uncertainty. Directly relevant to position shift failures in my project, where the object isn't quite where the camera estimated it to be.</p>
</details>

---

**[2]** Zhu, F., et al. (2021). Failure handling of robotic pick and place tasks with multimodal cues under partial object occlusion. *Frontiers in Neurorobotics*, 15, Art. 570507.

<details>
<summary>Show abstract</summary>
<p>This is the closest paper to the architecture of my project. They combine visual and proprioceptive signals to detect pick-and-place failures even when the object is partially hidden. Shows that using multiple sensor types together really does improve detection reliability compared to any single signal.</p>
</details>

---

**[6]** Corrigan, T., et al. (2019). Evaluation of binary gripper feedback reliability in bin-picking applications. *IEEE Transactions on Automation Science and Engineering*, 16(3), 1404–1413.

<details>
<summary>Show abstract</summary>
<p>Tests whether just using the gripper's open/close signal is reliable enough to detect grasp success in industrial bin-picking and no, it isn't. This is pretty much the motivation for why my project doesn't rely only on the gripper signal and adds camera verification on top.</p>
</details>

---

**[8]** Tian, S., et al. (2019). Manipulation by feel: Touch-based control with deep predictive models. In *Proceedings of IEEE ICRA 2019*, pp. 818–824.

<details>
<summary>Show abstract</summary>
<p>Uses deep learning on tactile/haptic signals to control robot manipulation. The idea is to predict what the next touch signal should be and flag anything unexpected. Background reading on tactile-based detection is relevant to the force-sensing side of my multi-modal system.</p>
</details>

---

**[10]** Lukezic, A., et al. (2018). Discriminative correlation filter with channel and spatial reliability. *International Journal of Computer Vision*, 126, 671–688.

<details>
<summary>Show abstract</summary>
<p>Introduces the CSRT visual tracker, which is more reliable than basic correlation filters because it accounts for which parts of the image are actually useful for tracking. Used in related work [9] for detecting if an object shifts before the robot touches it.</p>
</details>

---

**[12]** Hosoda, K., et al. (2006). Anthropic robotic soft fingertip with randomly distributed receptors. *Robotics and Autonomous Systems*, 54(2), 104–109.

<details>
<summary>Show abstract</summary>
<p>Develops a soft tactile fingertip sensor modelled after human touch receptors. Background paper on how robots can sense contact forces and slip without rigid sensors. Relevant context for why I use jaw-width monitoring as a rough proxy for force feedback with the RG2 gripper.</p>
</details>

---

**[18]** Roberge, J.-P., et al. (2016). A theoretical and experimental comparison of slip detection and re-grasping capabilities of optical and capacitive 3-axis tactile sensors. *Sensors*, 16(4), 497.

<details>
<summary>Show abstract</summary>
<p>Compares two types of tactile sensors for detecting slip and triggering a re-grasp. Useful for understanding the slip failure mode I'm dealing with, and for thinking about what sensor signals are actually useful for triggering a recovery action.</p>
</details>

---

## **Post-Grasp Verification (Vision-Based)**

> Using the camera to confirm success or failure after the robot has lifted an object.
{: .prompt-info }

**[9]** Zeng, A., et al. (2022). Robotic pick-and-place of novel objects in clutter. *International Journal of Robotics Research*, 41(7), 690–705.

<details>
<summary>Show abstract</summary>
<p>Covers perception for picking unfamiliar objects in clutter, including how to visually confirm whether the grasp has succeeded. Relevant to the camera-based post-grasp check I run after each pick attempt.</p>
</details>

---

**[11]** Deng, R., et al. (2022). Deep learning-based pose estimation of cylindrical objects for robotic grasping. *Robotics*, 11(2), 42.

<details>
<summary>Show abstract</summary>
<p>Uses a deep learning model to figure out the orientation of cylindrical objects for grasping. Directly relevant to orientation failure, as cylinders are hard to grasp because they look the same from many angles, making it easy to get the pose wrong.</p>
</details>

---

**[14]** Simeonov, A., et al. (2025). Sim2Real transfer for vision-based grasp verification. *arXiv preprint arXiv:2505.03046*.

<details>
<summary>Show abstract</summary>
<p>Trains YOLO and ResNet classifiers in simulation to verify whether a grasp succeeded, then transfers that to a real robot. Relevant to the YOLOv8n-cls classifier I'm using to distinguish "holding something" vs "empty gripper" after each pick attempt.</p>
</details>

---

## **Recovery Strategies**

> How robots plan and execute recovery actions after a failed grasp.
{: .prompt-info }

**[15]** Mahler, J., et al. (2017). Dex-Net 2.0: Deep learning to plan robust grasps with synthetic point clouds and analytic grasp metrics. In *Proceedings of RSS 2017*.

<details>
<summary>Show abstract</summary>
<p>Also listed under grasp planning. From a recovery angle, Dex-Net 2.0 uses information from a failed grasp to update the next grasp plan rather than just trying the same pose again. That iterative re-grasping idea is what my position-shift recovery routine is based on.</p>
</details>

---

**[16]** Eppner, C., et al. (2016). Lessons from the Amazon Picking Challenge. In *Proceedings of RSS 2016*.

<details>
<summary>Show abstract</summary>
<p>A writeup of everything that went wrong (and right) when teams tried to build real warehouse picking robots. Very grounded. Lots of practical lessons about failure recovery that you don't find in purely academic papers. Useful for checking my recovery routines against real-world constraints.</p>
</details>

---

**[Li et al., 2024]** Li, T., Yan, Y., Yu, C., An, J., Wang, Y., & Chen, G. (2024). A comprehensive review of robot intelligent grasping based on tactile perception. *Robotics and Computer-Integrated Manufacturing*, 90, 102792. [https://doi.org/10.1016/j.rcim.2024.102792](https://doi.org/10.1016/j.rcim.2024.102792)

<details>
<summary>Show abstract</summary>
<p>A broad review of how tactile sensors are used to make robot grasping smarter. Covers sensor types and how combining touch with vision improves reliability. Useful for getting an overview of the field before narrowing down to the specific detection methods used in this project.</p>
</details>

---

**[Vats et al.]** Vats, S., Jha, D. K., Likhachev, M., Kroemer, O., & Romeres, D. RecoveryChaining: Learning local recovery policies for robust manipulation. *arXiv preprint arXiv:2410.13979*. [https://arxiv.org/html/2410.13979v1](https://arxiv.org/html/2410.13979v1)

<details>
<summary>Show abstract</summary>
<p>Teaches a robot to automatically chain together recovery actions when manipulation goes wrong (includes collision, slip, position error). Trained in PyBullet sim. Very relevant to my project structur in identifying the failure type and validating recovery routines. It also gives me a clear insight on the constraints of the recovery routines.</p>
</details>

---

## **Hardware & Software Tools**

> References for the hardware and libraries directly used in this project.
{: .prompt-info }

**[17]** Universal Robots. (2019). *UR10/CB3 Technical Specifications*. Universal Robots A/S, Odense, Denmark. [https://www.universal-robots.com](https://www.universal-robots.com)

<details>
<summary>Show abstract</summary>
<p>The official spec sheet for the UR10 robot arm I'm using. Has all the information about the robot arm including reach, payload, joint limits, I/O specs that I used to justify a hardware constraint or design decision.</p>
</details>

---

**[19]** Jocher, G., et al. (2023). YOLO by Ultralytics. [https://github.com/ultralytics/ultralytics](https://github.com/ultralytics/ultralytics)

<details>
<summary>Show abstract</summary>
<p>The official citation for the Ultralytics YOLO library. I use YOLOv8n for object detection during the pick task, and YOLOv8n-cls for binary post-grasp classification (holding vs empty).</p>
</details>

---

**[20]** Luxonis. (2022). *OAK-D Lite Product Brief*. Luxonis Holdings. [https://docs.luxonis.com](https://docs.luxonis.com)

<details>
<summary>Show abstract</summary>
<p>Product documentation for the OAK-D Lite camera I'm using in an eye-in-hand setup on the robot. Covers the RGB and stereo depth specs. Required for the camera's field of view, depth range, and how it integrates with the DepthAI SDK.</p>
</details>

---

**[21]** Andersen, J. S. (2021). ur_rtde: A C++ and Python interface for the UR RTDE interface. [https://gitlab.com/sdurobotics/ur_rtde](https://gitlab.com/sdurobotics/ur_rtde)

<details>
<summary>Show abstract</summary>
<p>The library I use to control the UR10 robot from Python. It gives direct access to the robot's real-time data exchange (RTDE) interface so I can send joint targets, read the TCP pose, and check I/O signals with low latency. This is the main robot control interface throughout the project.</p>
</details>

---


For practical tutorials and implementation resources, see the [Resources](/final-year-blog/resources/) page.