---
title: "Post-Grasp Verification: Using CLIP and Collecting the Dataset"
date: 2026-03-09 12:00:00 +0000
categories: [Project Updates, Failure Detection]
tags: [clip, linear probe, failure detection, dataset, post-grasp, binary classification]
author: myuwaishin
pin: false
last_modified_at: false
---


With calibration working, I moved on to Stage 2 failure detection which is verifying gripper state visually after a grasp. I had been looking into how vision models could be used for this without needing a large labelled dataset, and came across CLIP. Without fine-tuning large neural network models, this can be used as a feature extractor and train a lightweight classifier based on the feature vectors. I spent this session setting up clip for training and building the dataset.

> **Note:** This post is written in April, backdated to the session date of 9 March.
{: .prompt-info }

---

## Why CLIP

The gripper state classifier is built on CLIP's ViT-B/32 image encoder, used as a fixed feature extractor. The encoder stays frozen, it is not retrained or fine-tuned. Each input image produces a 512-dimensional feature vector. A logistic regression linear probe can be trained on top of these features to classify between two states: **HOLDING** (object present between the jaws) and **EMPTY** (no object).

CLIP's pre-training already captures high-level visual distinctions, including the presence or absence of an object in a gripper. A reliable classifier can be built with a few hundred labelled images. A CNN like ResNet would need significantly more data to learn the same features from scratch. The linear probe can also be retrained quickly as the dataset grows.

---

## First Test: Zero-Shot CLIP

Before collecting any data, I first tested whether CLIP could classify gripper state out of the box using only text prompts and no training at all.

The idea: encode a set of text descriptions for each class and the live camera image into the same vector space, then pick whichever text prompt the image is most similar to.

```python
PROMPTS = {
    "holding": [
        "a robot gripper holding an object between its fingers",
        "a robotic hand grasping an object",
        "gripper with something in it",
    ],
    "empty": [
        "a robot gripper with nothing in it",
        "an empty robotic gripper",
        "gripper with no object",
    ],
}

logits = (100.0 * image_features @ text_features_n.T)  # cosine similarity × 100
probs  = logits.softmax(dim=-1)   # 6 values summing to 1.0
best_idx = probs.argmax()
label = "holding" if best_idx < 3 else "empty"
confidence = probs[best_idx]
if confidence < threshold:
    label = "uncertain"
```

When I tested it live, the classifications were correct, it was picking the right class. The problem was that confidence scores were low. Unfortunately, I forgot to record this initial observation so I can't show the exact confidence scores.

However, the zero-shot result was enough to confirm that CLIP genuinely understands the distinction between a loaded and empty gripper. That gave me confidence the linear probe approach would work. Training a classifier on top of those same features with labelled examples should push confidence up significantly. So I moved on to collecting a dataset.

---

## ROI Crop: Isolating the Gripper Area

The classifier does not run on the full camera frame. The full OAK-D Lite frame is 1920×1080, which includes the robot arm, table, background, and any other objects in the scene. Feeding this to the encoder would cause the model to learn irrelevant features and produce false positives whenever something appears near the gripper in the background.

Instead, a fixed rectangular crop is extracted from the bottom of the frame, isolating only the jaw area:

```python
crop_w = 1400   # wide — spans both open fingers
crop_h = 600    # tall enough to see the object being grasped
cx = w // 2
cy = h - (crop_h // 2) - 10    # anchored to the bottom edge
crop_frame = frame[y1:y2, x1:x2]
```

The crop is anchored to the bottom of the frame where the gripper sits in the eye-in-hand view. A blue rectangle is drawn on the live display to show the crop region in real time during collection.

<div style="display:flex; justify-content:center; gap:0.4rem; margin:1.5rem 0; flex-wrap:wrap;">
  <figure style="margin:0; text-align:center; width:48%;">
    <img src="/assets/img/ROI_fully_opened.png" alt="ROI crop — gripper open" style="width:100%; height:240px; object-fit:contain; border-radius:6px; background:#f5f5f5;" />
  </figure>
  <figure style="margin:0; text-align:center; width:48%;">
    <img src="/assets/img/ROI_fully_closed.png" alt="ROI crop — gripper closed" style="width:100%; height:240px; object-fit:contain; border-radius:6px; background:#f5f5f5;" />
  </figure>
</div>
<p style="text-align:center; font-size:0.85rem; color:#666; margin-top:-0.5rem;"><em>ROI crop region showing gripper opened and closed</em></p>




---

## What Was Collected

I collected 150 images per class. Both the cropped frame and the full frame are saved per capture.

> Find the dataset [here](https://livemdxac-my.sharepoint.com/:u:/g/personal/ms3433_live_mdx_ac_uk/IQCODYf4qeAvSI6oeZUwqrOWAbQKfj6O6BIVS5_mDV5-XQM?e=m5fa2a).
{: .prompt-info }

**HOLDING** class captured at different lift heights, different wrist orientations, different backgrounds, and with the object at different angles and positions within the jaws, including cases of partial occlusion by the jaw faces.

<div style="display:flex; gap:0.5rem; margin:1rem 0 0.2rem;">
  <div style="width:50%;"><img src="/assets/img/post_grab_holding1.png" style="width:100%; border-radius:6px;" /></div>
  <div style="width:50%;"><img src="/assets/img/post_grab_holding2.png" style="width:100%; border-radius:6px;" /></div>
</div>
<p style="text-align:center; font-size:0.8rem; color:#666; margin:0.2rem 0 1rem;"><em>Uncropped HOLDING dataset</em></p>

<div style="display:flex; gap:0.5rem; margin:1rem 0 0.2rem;">
  <div style="width:50%;"><img src="/assets/img/post_grab_holding3.png.jpg" style="width:100%; border-radius:6px;" /></div>
  <div style="width:50%;"><img src="/assets/img/post_grab_holding4.png.jpg" style="width:100%; border-radius:6px;" /></div>
</div>
<p style="text-align:center; font-size:0.8rem; color:#666; margin:0.2rem 0 1rem;"><em>ROI cropped HOLDING dataset</em></p>

**EMPTY** class captured at different heights and orientations across different backgrounds. Deliberately included hard negatives scenarios: objects appearing near the gripper but not between the jaws, objects partially in frame, partially between the fingers, and gripper at various widths from fully closed to fully open. These edge cases are the most likely to cause false positives and need to be well-represented in training.

<div style="display:flex; gap:0.5rem; margin:1rem 0 0.2rem;">
  <div style="width:50%;"><img src="/assets/img/post_grab_empty1.png" style="width:100%; border-radius:6px;" /></div>
  <div style="width:50%;"><img src="/assets/img/post_grab_empty2.png" style="width:100%; border-radius:6px;" /></div>
</div>
<p style="text-align:center; font-size:0.8rem; color:#666; margin:0.2rem 0 1rem;"><em>Uncropped EMPTY dataset</em></p>

<div style="display:flex; gap:0.5rem; margin:1rem 0 0.2rem;">
  <div style="width:50%;"><img src="/assets/img/post_grab_empty3.png.jpg" style="width:100%; border-radius:6px;" /></div>
  <div style="width:50%;"><img src="/assets/img/post_grab_empty4.png.jpg" style="width:100%; border-radius:6px;" /></div>
</div>
<p style="text-align:center; font-size:0.8rem; color:#666; margin:0.2rem 0 1rem;"><em>ROI cropped EMPTY dataset</em></p>

---

## Next

With 300 labelled images collected, the next session covers training the linear probe, running live inference, and handling low-confidence predictions on the workshop background.
