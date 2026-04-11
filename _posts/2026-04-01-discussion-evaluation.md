---
title: "Discussion, Evaluation and Future Work"
date: 2026-04-01 12:00:00 +0000
categories: [Project Updates, Evaluation]
tags: [evaluation, discussion, objectives, results, conclusion, future work]
author: myuwaishin
pin: false
last_modified_at: false
---

With the pipeline tested and the results back, this post steps back and looks at what actually worked, what fell short, and what it all means in context.

---

## Pipeline Posts

Here is every stage of the pipeline documented across the blog:

| Stage | Topic | Post |
|---|---|---|
| Stage 1 | ArUco Sweep — Explore | [Stage 1: ArUco Sweep](/final-year-blog/posts/stage1-aruco-sweep/) |
| Stage 2 | Navigate to Object | [Stage 2: Navigation](/final-year-blog/posts/stage2-navigation/) |
| Stages 3 & 4 | Grasp and Verify | [Stages 3 & 4: Grasp and Verify](/final-year-blog/posts/stages3-4-grasp-verify/) |
| Stages 5 & 6 | Transit and Release | [Stages 5 & 6: Transit and Release](/final-year-blog/posts/stages5-6-transit-release/) |
| Recovery | Failure and Recovery Routine | [Recovery](/final-year-blog/posts/recovery/) |
| Testing | Results and Evaluation | [Testing and Results](/final-year-blog/posts/testing-results/) |

---

## Key Results

The classifier came back with 100% recall, not a single real grasp was missed across 50 trials. The 4 false positives are fully accounted for by low TCP height, and they disappear entirely above 150 mm. Since the pipeline always classifies at clearance height (400 mm), the false positive rate in practice is zero. The design choice is validated directly by the data.

The two-layer detection held up under testing. Layer 1 (gripper width check) catches complete misses the instant the gripper closes. Layer 2 (YOLO classification) catches the edge cases that hardware feedback alone cannot resolve, the cases where the grip registers but the object is not actually between the jaws.

---

## Limitations

**Depth-based pose estimation** from YOLO object detection was not accurate enough to replace ArUco as the primary navigation method. Depth error from the OAK-D stereo pair compounds into positioning error that exceeds the graspable zone radius at the workspace distances used. ArUco PnP remains the reliable method. The YOLO + depth path functions as a fallback but needs further refinement to stand alone.

**OBB-based orientation estimation** was not completed within the project timeline. The current implementation aligns the gripper to the ArUco tag axis, which works for the test object but is not generalisable. Depth-based OBB from the OAK-D point cloud is the identified path forward for a future iteration.

---

## Unexpected Findings

**Clearance height had a much larger effect on classifier confidence than anticipated.** At low TCP heights (below 150 mm), confidence dropped to 55–85%. At 400 mm clearance height, it was consistently 99–100%. The overhead view is simply a cleaner classification frame, with less perspective distortion and less partial occlusion. This finding directly informed the pipeline design: always classify at clearance, never at hover height.

**Post-hover PnP correction was more effective than expected.** Rerunning ArUco detection from hover height before descending added a meaningful alignment improvement, the camera mount angle keeps the marker fully visible from hover without occlusion, which makes the correction reliable enough to be worth the extra step.

---

## Objectives Review

| Objective | Status | Notes |
|---|---|---|
| **Failure detection** | Completed | Two-layer detection using gripper hardware feedback and YOLO visual classification with 92% accuracy, exceeding the 85% target |
| **Recovery and retry** | Completed | Full autonomous recovery loop with 93% success rate, exceeding the 75% target |
| **ArUco Navigation** | Completed | 92% accuracy achieved |
| **Markerless Navigation** | Proof of concept | YOLO + stereo depth fallback implemented but depth error not resolved |
| **Grasp maximisation** | Proof of concept | Gripper aligns to ArUco tag axis but markerless object orientated grasp not completed within scope |
| **Transit monitoring** | Completed | Full object loss detected via live YOLO inference during carry |

---

## Conclusion

A full autonomous pick-and-place pipeline was built and demonstrated on a real UR10, with closed-loop grasp failure detection, visual verification, and recovery. Three primary objectives of failure detection, navigation, and recovery were met and exceeded their targets. The two proof-of-concept objectives of markerless navigation and grasp maximisation fell short of full completion within the timeline, but both have a clear path forward.

The clearance height design is the standout finding. It was not obvious going in that classifier confidence would be sensitive to height, but the data makes it conclusive: the pipeline is reliable precisely because it classifies from a clean overhead view, and not from a cluttered low-angle one.

---

## Future Work

- **Refine depth-based pose estimation** to make YOLO fallback navigation accurate enough to operate without ArUco, enabling the pipeline to handle objects without markers.
- **Implement depth-based OBB** from OAK-D point cloud for orientation-aware grasping, so gripper angle matches the actual object axis rather than the tag axis.
- **Extend slip detection** to monitor gradual grip degradation during transit, and not just full object loss.
- **Extend to multiple objects** with model retraining on a broader dataset.

## Successful Pick and Place Demonstration

<figure style="display:flex; flex-direction:column; align-items:center; margin:1.5rem 0;">
  <div style="width:80%; position:relative; padding-bottom:45%; height:0; overflow:hidden; border-radius:8px;">
    <iframe src="https://www.youtube.com/embed/7Vez4ncucxg"
      style="position:absolute; top:0; left:0; width:100%; height:100%; border:0;"
      allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
      allowfullscreen>
    </iframe>
  </div>
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Successful pick and place</em></figcaption>
</figure>

## Next

See the final demonstration of the pipeline in action on the next post.