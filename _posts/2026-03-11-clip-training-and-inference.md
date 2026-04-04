---
title: "CLIP Linear Probe: Training, Inference, and Retraining"
date: 2026-03-11 12:00:00 +0000
categories: [Project Updates, Failure Detection]
tags: [clip, linear probe, logistic regression, inference, confidence threshold, failure detection]
author: myuwaishin
pin: false
last_modified_at: false
---


With the dataset collected, this session covers training the linear probe on the frozen CLIP encoder, running live inference, and retraining after the classifier struggled on the blue workshop floor.

> **Note:** This post is written in April, backdated to the session date of 11 March.
{: .prompt-info }

---

## Training the Linear Probe

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="/assets/img/clip_training_fig.png" alt="CLIP training and inference" style="display:block; margin:0 auto; width:90%; border-radius:8px;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>CLIP training and inference</em></figcaption>
</figure>

A linear probe is a small, single-layer classifier trained on top of a frozen feature extractor. Here the classifier is logistic regression, which learns a decision boundary in the 512-dimensional feature space to separate HOLDING from EMPTY. It outputs a probability for each class rather than a hard label, which is what makes the confidence threshold work.

> Find the training script at [train_clip_probe.py](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/CLIP_post_grab/train_clip_probe.py).
{: .prompt-info }

**Feature extraction using CLIP ViT-B/32:**

```python
model, preprocess = clip.load("ViT-B/32", device=device)
with torch.no_grad():
    feature = model.encode_image(image_input)
    feature = feature / feature.norm(dim=-1, keepdim=True)  # L2 normalise
```

Each cropped image maps to a 512-dimensional embedding vector. 

**Logistic regression on top:**

```python
clf = LogisticRegression(class_weight='balanced', max_iter=1000)
clf.fit(X_train, y_train)
```

- Input: 512-d CLIP feature vectors
- Output: binary label `1 = Holding`, `0 = Empty`
- `class_weight='balanced'` handles any sample count imbalance automatically

The trained probe is saved as `clip_probe.pkl` and contains the logistic regression weights, around 5 KB.

**Full training flow:**

```
Offline training
  ├─ CLIP ViT-B/32 encodes every cropped image → 512-d vector
  ├─ Logistic Regression fitted on those vectors
  └─ Saved as clip_probe.pkl
```

---

## Live Inference

> Find the inference script at [live_clip_verify.py](https://github.com/MyuWaiShin/Final_Year_Project_2026).
{: .prompt-info }

At runtime, inference is three steps: crop the frame, encode with CLIP, classify with the loaded probe.

```python
feature = model.encode_image(image_input)
feature = feature / feature.norm(...)
prediction    = clf.predict(feature_np)[0]        # 0 or 1
probabilities = clf.predict_proba(feature_np)[0]  # [prob_empty, prob_holding]
confidence    = probabilities[prediction]          # confidence of the winning class
```

`predict_proba` returns one probability per class that always sums to 1.0. So a result looks like:

```
probabilities = [0.12, 0.88]
                  ↑        ↑
                Empty   Holding
prediction = 1  (Holding)
confidence = 0.88  →  confident result
```

It assigns a probability to each class and the winning class's probability is used as the confidence score. A Holding result at 51% means Empty is at 49%, in this case the model is nearly unsure.

**Confidence threshold:**

As my objective set initially is to achieve ≥ 85% accuracy, I set the confidence threshold to 0.85. So if the confidence is below 0.85, it is treated as uncertain and no recovery is triggered. Only a confident EMPTY result (p ≥ 0.85) should triggers the recovery sequence.

```python
if confidence < 0.85:
    predicted_label = f"Uncertain ({predicted_label})"
    color = (0, 165, 255)  # Orange
```

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <video controls muted loop style="width:90%; border-radius:8px;" id="clip-inference-vid">
    <source src="/assets/img/clip_life_inference.mp4" type="video/mp4">
  </video>
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Live CLIP inference showing Holding / Empty / Uncertain with confidence scores</em></figcaption>
</figure>
<script>
  document.getElementById('clip-inference-vid').playbackRate = 1.0;
</script>

**Problem 1: Low confidence**

From the video, we can see that the clip model is able to classify both holding and empty correctly by showing which class has higher probability. However, the confidence is usually lower than 0.85, so most of the time it is treated as UNCERTAIN. This is something I need to improve on as I want to achieve ≥ 85% accuracy.

**Problem 2: False positive**

The other behavious is that when objects enter the cropped region, the model confidence for empty drops significantly as the model gets confused. This could cause a potential false positive issue for the empty class, where it outputs holding instead of empty when the gripper is empty but there is an object between the jaws.

As I have identified these issues, the solution is to collect more data with the same scenarios to increase the model's confidence and accuracy.

**Problem 3: Distribution gap**

After the first training run, live inference on the blue lab floor showed lower confidence. Predictions on those frames regularly fell below 0.85.

The issue was a distribution gap: the initial dataset of 150 images per class was not collected enough with the blue floor visible, so the CLIP could not capture the features of the blue floor well in the training dataset.

So I collected 50 additional images per class specifically with the blue floor visible, then the probe was retrained on the expanded dataset (~200 images per class).

After retraining, confidence on blue-floor frames increased and predictions consistently cleared the 0.85 threshold. The classifier now handles the actual workspace reliably.

---

## Summary

```
Live inference
  └─ crop frame → CLIP ViT-B/32 → LR classifier → Holding / Empty + confidence or UNCERTAIN
```

The linear probe on CLIP features is fast, lightweight, and required very little training data. The `.pkl` file is 5 KB. It generalises well across gripper positions and backgrounds once the training set covers the relevant distribution.

---

## Next

With the classifier working reliably, the next step is integrating it into the full failure detection pipeline, wiring up the two CLIP checkpoints (post-lift and slip-monitor re-verify) into the pick-and-place sequence.
