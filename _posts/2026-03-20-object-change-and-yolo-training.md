---
title: "Changing Object and Retraining Detection and Classification"
date: 2026-03-20 12:00:00 +0000
categories: [Project Updates, Training]
tags: [yolo, yolo26, detection, classification, clip, vitamin bottle, obb, dataset]
author: myuwaishin
pin: false
last_modified_at: false
---

> **Note:** This post is written in April, backdated to the session date of 20 March.
{: .prompt-info }


## Changing Object

Navigation is the next stage of the pipeline, but before I could start writing it I hit a wall because my previously trained detection model doesn't perform well enough for the navigation stage. The small cylindrical blocks I was using are uniformly coloured with no strong distinguishing features, so the model struggled to consistently detect them at the confidence level needed for the robot to act on those detections, They were also often confused with the background.

Practically, I decided to change the object entirely and retrain everything from scratch.

I switched to vitamin bottles. The reasons are:

- They have distinct visual features, which means auto-labelling will be more reliable
- The base of the bottle is large and flat, giving enough surface area to attach a bigger ArUco tag, which improves pose estimation accuracy
- They are cylindrical in shape, which fits the Oriented Bounding Box (OBB) approach I proposed for the proof of concept
- I also placed an ArUco tag on the bottom of the bottle to use ArUco based pose-estimation as a fallback in case detection confidence drops

---

## Data Collection

I spent the full day collecting video data. Everything was shot from the OAK-D camera mounted on the robot's wrist, mirroring the actual perception setup.

### Detection data

Videos of the vitamin bottle from multiple angles, specifically overhead and near-overhead views since that is how the camera sees the workspace during the sweep and navigation stages.

I varied:

- Rotation and orientation of the bottle across the field of view
- Lighting conditions and positions within the workspace
- Both bottle-only and bottle-with-ArUco-tag-visible frames

### Classification data

For the gripper state classifier (empty vs holding), I collected at typical clearance heights of 200–600 mm above the floor:

- **Empty**: Gripper closing on air with different jaw widths. Potential false positive scenarios where bottles were visible between the gripper jaws, including close up floor and object views
- **Holding**: Gripper closed around the bottle, with varied rotation of the object between the fingers

---

## The Architecture: YOLO26 Nano

Both models, detection and classification, use YOLO26 nano (`yolo26n`), a new object detection model released by Ultralytics on 14 January 2026.

YOLO26 is specifically engineered for edge and low-power deployment. The key architectural changes relevant to this project:

| Feature | What it means |
|---|---|
| NMS-free inference | No NMS post-processing step, no manual threshold tuning needed |
| DFL removed | Simpler bounding box regression, better compatibility with edge hardware |
| STAL | Improves accuracy for small objects in frame |

Nano was chosen deliberately. The pipeline runs detection and classification on live camera frames in real-time, so inference speed matters. No NMS post-processing is also a plus, as my previous YOLOv8n model required NMS tuning to avoid duplicate detections. 

Both models were initialised with pretrained ImageNet weights and fine-tuned on custom robot-collected data (transfer learning).

---

## Model 1: YOLO26n Object Detection

### Dataset

| Property | Value |
|---|---|
| Total images | 6,140 |
| Classes | 1 |
| Annotation format | YOLO `.txt` (normalised cx, cy, w, h) |
| Source | Overhead OAK-D frames at various robot positions |

### Training Parameters

| Parameter | Value |
|---|---|
| Epochs | 100 (early stop patience: 15) |
| Image size | 640 × 640 |
| Batch size | 32 |

### Augmentation

| Augmentation | Setting |
|---|---|
| HSV hue shift | ±1.5% |
| HSV saturation | ±70% |
| HSV value (brightness) | ±40% |
| Horizontal flip | 50% probability |
| Scale | ±50% |
| Translation | ±10% |
| Mosaic | 100% (4-image mosaic every batch) |
| RandAugment | Auto |


The heavy saturation and brightness augmentation matters here because the robot workspace has fixed overhead lighting that produces reflections and shadows depending on bottle orientation and position.

### Results

> Find my results here: [results](https://github.com/MyuWaiShin/Final_Year_Project_2026/tree/main/full_pipeline/models/detection/yolo26n_detect_V1)
{: .prompt-info }

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/results copy.png" style="width:70%; border-radius:6px; display:block; margin:0 auto;" alt="Results">
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Training results</em></figcaption>
</figure>

All three loss curves (box loss, cls loss, DFL loss) descend cleanly over 100 epochs with no overfitting. Validation losses track training closely throughout.

Precision and recall both climb to near 1.0 by around epoch 30. The remaining training epochs polish the bounding box regression further.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/BoxF1_curve.png" style="width:70%; border-radius:6px; display:block; margin:0 auto;" alt="Results">
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>F1-Confidence Curve</em></figcaption>
</figure>

**F1-Confidence Curve:** The confidence threshold at which F1 is maximised. It's low (28.8%) because the model is extremely certain. Every detection above 28.8% confidence in the val set was correct with no misses. I will set the live threshold to 0.80 which is way above this, so only very strong detections fire.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/BoxPR_curve.png" style="width:70%; border-radius:6px; display:block; margin:0 auto;" alt="Results">
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Precision-Recall Curve</em></figcaption>
</figure>

**Precision-Recall Curve:** mAP@0.5 = 0.995. Precision stays at 1.0 all the way to recall ≈ 1.0. Essentially perfect.

**Confusion Matrix (validation set):**

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/confusion_matrix.png" style="width:90%; border-radius:6px; display:block; margin:0 auto;" alt="Detection confusion matrix">
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Confusion matrix</em></figcaption>
</figure>

The 4 background false positives are filtered by the 0.80 live threshold, since false positives tend to carry lower confidence scores.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/labels.jpg" style="width:70%; border-radius:6px; display:block; margin:0 auto;" alt="Labels distribution">
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Labels distribution</em></figcaption>
</figure>

---

## Live Inference

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <div style="width:80%; position:relative; padding-bottom:45%; height:0; overflow:hidden; border-radius:8px;">
    <iframe src="https://www.youtube.com/embed/oqTGMXniG10"
      style="position:absolute; top:0; left:0; width:100%; height:100%; border:0;"
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
      allowfullscreen>
    </iframe>
  </div>
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Demo: YOLO26n detection live inference</em></figcaption>
</figure>


The confidence observed on live frames was 94–98% consistently with no confidence fluctuations. 

---

## Model 2: YOLO26n Gripper State Classification

### Dataset

| Property | Value |
|---|---|
| Images per class | 3,600 |
| Total images | 7,200 |
| Classes | `empty`, `holding` |
| Captured at | 200–600 mm clearance height, overhead closed-gripper view |

### Training Parameters

Same as detection except:

| Parameter | Value |
|---|---|
| Task | `classify` |

### Results

> Find my results here: [results](https://github.com/MyuWaiShin/Final_Year_Project_2026/tree/main/full_pipeline/models/classification/yolo26n_cls_V1)
{: .prompt-info }

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/results.png" style="width:70%; border-radius:6px; display:block; margin:0 auto;" alt="Results">
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Training results</em></figcaption>
</figure>

By epoch 1, accuracy_top1 reached 1.00. Training loss dropped from 0.117 on epoch 1 to ~0.003 by epoch 2. Validation loss reached effectively zero by epoch 4. Early stopping was triggered at epoch 16 as there was nothing left to improve.

The two classes are visually distinct from overhead. Empty shows a symmetrical closed gripper with nothing between the fingers and holding shows a visible object protruding between the jaws. The model had no difficulty separating them.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/confusion_matrix_normalized.png" style="width:90%; border-radius:6px; display:block; margin:0 auto;" alt="Normalised confusion matrix">
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Normalised confusion matrix</em></figcaption>
</figure>

**Confusion matrix:** 1.00 on both diagonal cells. No misclassifications and no background rejections.

---

## Model 3: CLIP Gripper State Classification

I retrained CLIP on the same dataset of 3600 for each class for gripper state classification. However, when testing it on live camera frames, I found that it was performing fairly the same as the old model which was trained on 200 images for each class.

## CLIP vs YOLO26 Classification Comparison

I compared the two models on live camera frames. The live inference uses a 0.90 confidence threshold.

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <div style="width:80%; position:relative; padding-bottom:45%; height:0; overflow:hidden; border-radius:8px;">
    <iframe src="https://www.youtube.com/embed/jEafNMICNm8"
      style="position:absolute; top:0; left:0; width:100%; height:100%; border:0;"
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
      allowfullscreen>
    </iframe>
  </div>
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Demo: CLIP vs YOLO26n live classification comparison</em></figcaption>
</figure>


CLIP still sits around 75% accuracy with the new dataset. It occasionally flips between `empty` and `holding` in false-positive scenarios (object visible nearby but not in the gripper). YOLO26n stays solid at near-100% and only misclassifies in the extreme edge case where the bottle is less than 100–150 mm from the gripper but not gripped. This is a very good result still, and edge case can be avoided by lifting TCP to enough clearance height before classifying.

So I decided to drop CLIP and go with YOLO26n for classification. A 75% accurate classifier is not good enough and less than my proposed objective of 85% for a robotic verification step. 


## Summary

| | Detection | Classification |
|---|---|---|
| Model | yolo26n (detect) | yolo26n (classify) |
| Training images | 6,140 | 7,200 (3,600 × 2) |
| Classes | 1 (`object`) | 2 (`empty`, `holding`) |
| Live threshold | 0.80 | 0.90 |

With reliable detection and classification now in place, I can move on to writing the navigation stage of the pipeline.

---

## Next

With trained models for both detection and classification, the next step is implementing `navigate.py` that takes the the detected object pose and moves to hover pose above it.
