---
title: "Testing and Results"
date: 2026-03-30 12:00:00 +0000
categories: [Project Updates, Testing]
tags: [testing, results, pick-and-place, evaluation, classifier, navigation, recovery]
author: myuwaishin
pin: false
last_modified_at: false
---

> **Note:** This post is written in April, backdated to the session date of 30 March.
{: .prompt-info }

---

## Binary Classifier Testing


| Metric | Result |
|---|---|
| Accuracy | 92% |
| Recall | 100% |
| F1 Score | 0.92 |

**Confusion matrix (n = 50):**

| | Predicted HOLDING | Predicted EMPTY |
|---|---|---|
| Actual HOLDING | 23 (TP) | 0 (FN) |
| Actual EMPTY | 4 (FP) | 23 (TN) |

<img src="/assets/img/classifier_confusion_matrix.png" alt="Classifier Confusion Matrix" style="width:40%; border-radius:6px; display:block; margin:0 auto;">

<img src="/assets/img/tcp_height_vs_accuracy.png" alt="TCP Height vs Accuracy" style="width:40%; border-radius:6px; display:block; margin:0 auto;">

Recall is 100%, every real grasp was correctly identified. The 4 false positives all occurred below 150 mm TCP-to-object distance, where the object fills the inter-jaw region without being physically gripped. Above 150 mm the classifier is clean. In normal pipeline operation the gripper classifies at clearance height (400 mm), so these low-height false positives do not occur.

---

## Navigation Testing

| Metric | Result |
|---|---|
| Aligned | 11 / 12 |
| Navigation accuracy | 92% |
| Failures | 1 (P1 — offset exceeded graspable zone) |


Tested across a 6×2 grid at 150 mm pitch (rows 2 and 3 of the workspace). 11 of 12 positions aligned within the bottle base radius (33 mm). The single failure was at P1, the far corner of the workspace, where cumulative calibration error pushed the hover position outside the graspable zone. Accuracy was consistent across the full workspace width and depth for all other positions.

---

## Recovery Testing

<img src="/assets/img/fig_recovery.png" alt="Recovery Workspace" style="width:40%; border-radius:6px; display:block; margin:0 auto;">

| Metric | Result |
|---|---|
| Recovered | 13 / 14 |
| Recovery success rate | 93% |
| Objects undetectable during sweep | 0 |

14 random positions tested across the controlled workspace. 13/14 trials re-navigated successfully above the object after a simulated grasp failure. The 1 failure was a contact failure on the first retry attempt — re-navigation succeeded on the second attempt. Zero objects were undetectable during the circular sweep, confirming recovery is reliable when the ArUco tag is visible.
