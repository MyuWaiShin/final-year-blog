---
title: "Discussion, Evaluation and Future Work"
date: 2026-04-01 12:00:00 +0000
categories: [Project Updates, Evaluation]
tags: [evaluation, discussion, objectives, results, conclusion, future work]
author: myuwaishin
pin: false
last_modified_at: false
---

> **Note:** This post is written in April, backdated to the session date of 1 April.
{: .prompt-info }

---

## Key Results

Classifier recall is 100% — a real grasp is never missed. The 4 false positives are fully explained by low TCP height and are eliminated by the clearance height design in normal operation.

The two-layer detection works as intended. Layer 1 (width check) catches complete misses instantly. Layer 2 (YOLO classification) catches the edge cases that hardware feedback alone cannot resolve.

---

## Limitations

**Depth-based pose estimation** from object detection needs further refinement. Depth error compounds into positioning error, meaning ArUco PnP remains the primary method for reliable navigation. YOLO + stereo depth functions as a fallback but is less accurate.

**OBB orientation estimation** was not completed within the time frame. Depth-based OBB from point cloud is identified as the path forward for a future iteration.

---

## Unexpected Findings

**Clearance height had a larger effect on classifier confidence than expected.** Raising from low height to 400 mm moved confidence from 55–85% to 99–100% consistently. The overhead view at clearance height is much cleaner than a low-angle partial view.

**Post-hover PnP correction meaningfully improved alignment.** The camera mount angle gives a practical advantage — the tag remains visible from hover without occlusion, making the correction reliable.

---

## Objectives Review

| Objective | Status | Notes |
|---|---|---|
| Failure detection | Completed | Two-layer detection using sensor fusion — gripper hardware feedback and YOLO visual classification |
| Recovery and retry | Completed | Full autonomous recovery loop — relocates object and retries without human input |
| Navigation | Proof of concept | ArUco PnP works well as primary method. Depth-based pose estimation from object detection needs refinement |
| Maximise grasp success | Proof of concept | Gripper orientation aligns to ArUco tag axis. OBB-based orientation from object detection not completed |
| Transit monitoring | Completed | Full object loss detected via live YOLO inference during carry. Grip force adjustment is outside scope |

---

## Conclusion

Built and demonstrated a full autonomous pick-and-place pipeline on a real UR10 with closed-loop grasp failure detection and recovery.

The two-layer detection works as intended. Layer 1 catches complete misses via hardware feedback. Layer 2 catches false positives via YOLO classification at clearance height. The recovery routine successfully relocates the object and retries without human input.

The classifier achieves 100% recall and 92% accuracy — zero missed grasps in testing. The clearance height design is directly validated by test data, eliminating all false positives above 150 mm.

---

## Future Work

- **Refine depth-based pose estimation** from object detection — make YOLO fallback navigation accurate enough to operate without ArUco.
- **Implement depth-based OBB** from OAK-D point cloud for orientation-aware grasping on objects without a marker.
- **Extend slip detection** to monitor gradual grip degradation, not just full object loss.
- **Test generalisation** to novel object classes with model retraining.

