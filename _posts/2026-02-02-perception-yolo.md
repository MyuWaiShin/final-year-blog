---
title: "Object Detection Training - YOLOv8 vs YOLOv26"
date: 2026-02-02 10:00:00 +0000
categories: [Project Updates, Perception]
tags: [yolo, computer-vision, deep-learning, object-detection]
author: myuwaishin
pin: false
last_modified_at: false
---


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

Here are the results from the live testing of all models:

<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(45%, 1fr)); gap: 10px;">
  <img src="/assets/img/yolo_v8_baseline.png" alt="YOLOv8 Baseline" style="width: 100%;" />
  <img src="/assets/img/yolo_top_view_fail.png" alt="Top View Failure" style="width: 100%;" />
  <img src="/assets/img/yolo_multi_class.png" alt="Multi Class Detection" style="width: 100%;" />
  <img src="/assets/img/yolo_confusion.png" alt="YOLO Confusion" style="width: 100%;" />
  <img src="/assets/img/yolo_overfit.png" alt="Overfitting Example" style="width: 100%;" />
  <img src="/assets/img/yolo_new_1.png" alt="Test Result 1" style="width: 100%;" />
  <img src="/assets/img/yolo_new_2.png" alt="Test Result 2" style="width: 100%;" />
  <img src="/assets/img/yolo_new_3.png" alt="Test Result 3" style="width: 100%;" />
</div>
<br>

**Key Observations & Issues:**



Based on these tests, it's clear that the models (especially the newer YOLOv26) are struggling. Here are the four main issues I need to fix:

1.  **Top-Down Blindness:** None of the models detect objects well when viewed from directly above. I need significantly more data from this angle.
2.  **Background Sensitivity:** The models were trained on a brighter lighting but tested on lower-lighting. They are over-sensitive to this domain shift and need more background variation and lighting variation in the dataset.
3.  **Cylinder & Arc Confusion:** These shapes are frequently either missed completely or mistaken for each other.
4.  **Small Cube Detection:** The smaller cubes are harder to detect and need more dedicated training examples.

## Next Steps

I definitely need the speed of YOLOv26 for the live detection pipeline, so I'm not giving up on it yet.

1.  **Data Collection:** Capture a new dataset specifically targeting top-down views and adding background variation.
2.  **Retraining:** Retrain YOLOv26 with the augmented dataset.
3.  **Pose Estimation:** Once detection is reliable, implement 6D pose estimation.

*You can find the training data for Model 1 [here](https://livemdxac-my.sharepoint.com/:u:/g/personal/ms3433_live_mdx_ac_uk/IQCzK2LMzAIgT6h4ukf1n96lASdN4jogo36mJXC-pVdGLYQ?e=xKAwV0).*

<div style="text-align: center; margin: 20px 0;">
  <video style="width: 50%; border-radius: 8px;" autoplay loop muted playsinline>
    <source src="/final-year-blog/assets/img/cubes_video.mp4" type="video/mp4">
    Your browser does not support the video tag.
  </video>
  <p style="font-size: 0.85em; color: #6c757d;"><em>Running inference on the test data.</em></p>

