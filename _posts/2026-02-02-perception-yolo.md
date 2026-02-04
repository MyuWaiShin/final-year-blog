---
title: "Perception Training - YOLOv8 vs YOLOv26"
date: 2026-02-02 10:00:00 +0000
categories: [Project Updates, Perception]
tags: [yolo, computer-vision, deep-learning, object-detection]
author: myuwaishin
pin: false
last_modified_at: false
---

## The Perception Challenge

To pick up objects autonomously, the robot first needs to know exactly where they are. I am implementing a vision pipeline to detect and localize my three target objects: **Cubes, Cylinders, and Arcs**.

## Model Selection: Why YOLOv26?

With the recent release of **YOLOv26**, I wanted to benchmark it against the standard YOLOv8. 
YOLOv26 offers significant theoretical advantages for my embedded setup:
*   **No NMS (Non-Maximum Suppression):** Eliminating the NMS step reduces inference latency.
*   **Efficiency:** It is optimized for low-power compute, making it highly suitable for real-time applications running directly on edge devices (like my camera setup).

## Training Setup

I trained **4 different models** on a dataset of approximately **12,000 images per class**. 
*   **Classes:** Cube, Cylinder, Arc
*   **Background:** White table (Training data)
*   **Testing:** Zero-shot on real workspace background

**Models Trained:**
1.  YOLOv8 (50 epochs)
2.  YOLOv8 (100 epochs)
3.  YOLOv26 (50 epochs)
4.  YOLOv26 (100 epochs)

## Experimental Results

The results were surprisingly mixed, highlighting the importance of data quality over model architecture.

*You can find the training data for Model 1 [here on OneDrive](#).*

### 1. YOLOv8 (50 Epochs) - *The Current Best*
This model performed the best out of all four. 
![YOLOv8 Baseline](/assets/img/yolo_v8_baseline.png)

However, it still struggles with **Top-Down Views**, particularly for the cylinder. It often fails to detect the cylinder when viewed directly from above. 
![Top View Failure](/assets/img/yolo_top_view_fail.png)

### 2. YOLOv8 (100 Epochs) - *Overfitting*
I expected improvements with more training, but the performance actually degraded. The model appears to be overfitting to the white background of the training set.
![Multi Class Detection](/assets/img/yolo_multi_class.png)

### 3. YOLOv26 (50 Epochs) - *Fast but Inaccurate*
The inference is noticeably faster, validating the efficiency claims. However, the accuracy took a massive hit.
![YOLO Confusion](/assets/img/yolo_confusion.png)

**Major Issue:** Significant class confusion. It frequently mistakes cubes for arcs or cylinders. The lack of Non-Maximum Suppression seems to make it less robust to ambiguous features in my current dataset.

### 4. YOLOv26 (100 Epochs) - *Complete Failure*
This was the worst performer. The training actually **early-stopped at iteration 44**. The model overfitted so severely that it fails to detect *anything* in the real-world test.
![Overfitting Example](/assets/img/yolo_overfit.png)

## Technical Analysis

**The "Zero-Shot" Problem:**
All models were trained and tested on a clean white background workspace. The YOLOv26 architecture seems much more sensitive to this domain shift than YOLOv8.

**Data Deficiencies Identified:**
*   **Cylinder & Arc Data:** The dataset is unbalanced or lacks feature-rich angles for these shapes.
*   **Top-View Bias:** The models fail almost consistently from top-down angles. I need to capture significantly more data from this perspective.
*   **Variation:** The dataset lacks background diversity, causing the models (especially v26) to latch onto background artifacts rather than object features.

## Next Steps

1.  **Data Collection:** Capture a new dataset specifically targeting top-down views and adding background variation.
2.  **Retraining:** Retrain YOLOv26 with the augmented dataset.
3.  **Pose Estimation:** Once detection is reliable, implement 6D pose estimation.


