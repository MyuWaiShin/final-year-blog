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


Now, it was time to test the pipeline as a whole and see how it performs. There are three main components to the pipeline: the binary classifier, the navigation system, and the recovery routine. I then started by testing each component of the pipeline separately.

---

## Binary Classifier Testing

For the binary classifier, I tested it with 50 random positions in the workspace, including different tcp heights and orientations.

| | Predicted HOLDING | Predicted EMPTY |
|---|---|---|
| Actual HOLDING | 23 (TP) | 0 (FN) |
| Actual EMPTY | 4 (FP) | 23 (TN) |

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="{{ '/assets/img/classifier_confusion_matrix.png' | relative_url }}" alt="Classifier Confusion Matrix" style="width:80%; border-radius:6px;">
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Confusion matrix (n = 50)</em></figcaption>
</figure>

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="{{ '/assets/img/tcp_height_vs_accuracy.png' | relative_url }}" alt="TCP Height vs Accuracy" style="width:80%; border-radius:6px;">
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Accuracy vs TCP height (mm)</em></figcaption>
</figure>

| Metric | Result |
|---|---|
| Accuracy | 92% |
| Recall | 100% |
| F1 Score | 0.92 |

From the results, the recall is 100%, meaning every real grasp was correctly identified, which is what we need. The 4 false positives all occurred below 150 mm TCP-to-object distance, where the object fills the inter-jaw region without being physically gripped. Above 150 mm the classifier is clean. In normal pipeline operation the gripper classifies at clearance height (400 mm), so these low-height false positives do not occur.

---

## Navigation Testing

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="{{ '/assets/img/fig_navigation_(2).png' | relative_url }}" alt="Navigation Workspace" style="width:80%; border-radius:6px;">
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Navigation test: 6×2 grid at 150 mm pitch, showing hover positions vs target</em></figcaption>
</figure>

| Metric | Result |
|---|---|
| Aligned | 11 / 12 |
| Navigation accuracy | 92% |
| Failures | 1 (P1 — offset exceeded graspable zone) |


For the navigation system, I tested it with a 6×2 grid at 150 mm pitch. 11 of 12 positions aligned within the bottle base radius (33 mm). The single failure was at P1, the far corner of the workspace, where cumulative calibration error pushed the hover position outside the graspable zone. Accuracy was consistent across the full workspace width and depth for all other positions.

---

## Recovery Testing

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <img src="{{ '/assets/img/fig_recovery.png' | relative_url }}" alt="Recovery Workspace" style="width:80%; border-radius:6px;">
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Recovery test: 14 random positions across the workspace</em></figcaption>
</figure>

| Metric | Result |
|---|---|
| Recovered | 13 / 14 |
| Recovery success rate | 93% |
| Objects undetectable during sweep | 0 |

For the recovery routine, I tested it with 14 random positions across the controlled workspace. Out of 14 trials, 13 trials re-navigated successfully above the object after a simulated grasp failure. The 1 failure was a contact failure on the first retry attempt, but re-navigation succeeded on the second attempt. Zero objects were undetectable during the circular sweep, confirming that the recovery routine is reliable when the ArUco tag is visible.

### Conclusion

Overall, the pipeline is reliable and can be used for autonomous pick-and-place operations. The binary classifier is accurate and can reliably verify grasps success and failure with high precision, hitting 100% recall with 92% accuracy, which is above the 85% target. The navigation system is accurate and can reliably navigate to objects with high precision, with 92% accuracy. The recovery routine is reliable and can recover from grasp failures with high success rate of 93%, exceeding the 75% target.


## Next

Next post will cover the discussion and evaluation of the pipeline.
