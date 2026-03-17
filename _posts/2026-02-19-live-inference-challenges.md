---
title: "Live Inference on OAK-D Lite & Challenges"
date: 2026-02-19 12:00:00 +0000
categories: [Project Updates, Perception]
tags: [yolo, inference, deployment, oak-d, obb, post-processing]
author: myuwaishin
pin: false
last_modified_at: false
---

After training the YOLO models, the next step was getting them running live on the OAK-D Lite. While the models did well on the validation dataset, running them on a live video feed brought up several real world issues.

## Initial Deployment
I tested all the models I trained. YOLOv8n ended up being the best choice for deployment. It matched the strong training metrics and ran smoothly on the host CPU at 30fps.  

YOLOv26n looked promising during training, but when I dropped it into the Python inference script, I hit a tensor shape mismatch.

A tensor is a multidimensional array the AI outputs. The Python script (`detect_objects.py`) was written using the official YOLOv8 documentation, which expects an output shaped like `(1, 84, 8400)`. The code then slices that grid to get the x, y, width, height, and confidence scores.

Older models like YOLOv5n use a completely different output structure of `(1, 25200, 85)`. 

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/raw_detection_1773009406404.png" alt="YOLOv5n Tensor Mismatch" style="display:block; float:none; width:90%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>YOLOv5n Tensor Mismatch</em></figcaption>
</figure>

When I run YOLOv5n through the script, it draws thousands of tiny boxes packed into the top left corner with impossible 84,000% confidence scores as seen in the image above. This happens because the script is trying to apply YOLOv8 math to a YOLOv5 grid.

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/raw_detection_1773005968475.png" alt="YOLOv26n Tensor Mismatch" style="display:block; float:none; width:90%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>YOLOv26n Tensor Mismatch</em></figcaption>
</figure>

The YOLOv26n ONNX file had a similar shape issue that resulted in massive random boxes.

I would have had to write completely different parser functions for v5, v8, and v26 to handle this. However, it made more sense to just stick with YOLOv8n since it was already the most accurate and stable model.

### Host vs Edge Inference
The original plan was to compile the ONNX model into a `.blob` file to run inference directly on the OAK-D Lite's VPU. This would take the load off the computer's CPU.

However, exporting to the `.blob` format requires stripping away the Non-Maximum Suppression (NMS) layer. Since the pipeline relies heavily on NMS and custom Python logic to calculate grasping angles using OBB (detailed below), I decided to run host side inference using ONNX Runtime instead in the meantime.

I also kept the RGB video streaming from the OAK-D at full 1080p resolution. Running at 1080p gives a clear visual feed for debugging and gives the post processing steps a high resolution image to work with. The depth reading is not affected by this because the OAK-D uses its separate dual 400p mono sensors to calculate depth.

By running inference on the host, the PC handles the YOLO forward pass, overlap filtering, and orientation math without any issues.

## Real World Challenges
The images below are from zero shot inference tests in a home setup to highlight edge cases. This setup was slightly different from the training environment to show why post processing is necessary.

During the first few live tests, the model struggled with a few things.

**1. False Positives** 
The calibration mat has hundreds of small black dots. Under normal lighting, the raw YOLOv8n model would misidentify these clusters of dots as cubes with low confidence (20 to 40%).

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/raw_detection_1773005806709.png" alt="YOLOv8n detecting black dots as cubes" style="display:block; float:none; width:90%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>YOLOv8n detecting black dots as cubes</em></figcaption>
</figure>

Without filtering, the screen was filled with false detections. The robot would not know which cube was real.

Why did this happen? This is because of the heavy augmentations applied during training, it was set to scale = 0.5 (shrinking objects by up to 50%) and Mosaic = 1.0 (tiling 4 images together and naturally shrinking the apparent size of objects further). It was applied because I wanted to generalise the environment and include as much variation as possible.

Because 1/3 of our training videos were recorded on this textured mat to guarantee good depth readings, the model saw these black dots constantly. Due to the shrinking augmentations, the small cubes were downscaled into tiny, low-res squares during the training passes. The model essentially learned to detect the black dots as cubes as well.

**2. Jitter and Flickering**
When the lighting shifted or the camera angle slightly changed, a detection would flash on the screen for one frame and disappear the next. This meant the robot cannot reliably target an object.

**3. Overlapping Boxes**
Sometimes a single physical cube would generate two or three overlapping boxes. The system would think there were multiple objects in the exact same spot.

## Post Processing Fixes
To fix these issues, I added three filtering steps in `detect_objects.py`.

> Find the script at [Perception/detect_objects.py](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/perception/detect_objects.py).
{: .prompt-info }

### Confidence Thresholding
The simplest immediate fix for the black dot issue was raising the `conf_threshold` strictly to 80% (0.80). At this level, the model immediately ignored the background noise and only reported objects it was absolutely certain about.

    # Initialize detector
    detector = YOLODetector(model_path, conf_threshold=0.8)

However, relying on an 80% threshold creates a new problem for zero-shot environments. According to our training evaluation, the optimal F1 threshold for this YOLOv8n model, the exact mathematical point where it perfectly balances Precision (avoiding false positives) and Recall (finding all real objects), is ~0.57. 

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/v8n_F1_curve.png" alt="YOLOv8n F1 Confidence Curve" style="display:block; float:none; width:90%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>YOLOv8n F1 Confidence Curve</em></figcaption>
</figure>

By forcing the threshold to 0.80 to hide the black dots, we are also unintentionally ignoring valid detections in their confidence at 60-70%.

### Future Fix
While 0.57 is a very solid F1 peak for a Nano model, the root cause of mistaking dots for cubes needs to be fixed at the training level so confidence threshold can be safely set to 80% without losing any valid detection. The action plan when retraining is:

- **Fix the Augmentations:** Decrease scale from 0.5 to 0.2 and disable mosaic entirely. This stops the model from shrinking physical cubes down to the size of tiny dots during training.
- **Domain Diversity:** Collect a small dataset at home to introduce lighting and background variations so the model learns not to panic when tested at non-identical lab setups. 
- **Upgrade Architecture:** Train a slightly larger YOLOv8s (Small) model instead of YOLOv8n (Nano). It has a better architectural capacity for isolating objects from complex backgrounds, which naturally pushes the optimal F1 threshold higher, possibly towards 0.75+.


### Non-Maximum Suppression (NMS)
To fix the overlapping boxes, I added an NMS algorithm. It checks the Intersection over Union (IoU) of two boxes representing the same class. If two boxes overlap by more than 45%, the script discards the box with the lower confidence score. 

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/overlapping_bounding_boxes.gif" alt="Overlapping Bounding Boxes example" style="display:block; float:none; width:90%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Overlapping bounding boxes on the same object before NMS is applied</em></figcaption>
</figure>

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/NMS.png" alt="Non-Maximum Suppression logic" style="display:block; float:none; width:70%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Non-Maximum Suppression logic. Source: <a href="https://blog.cubed.run/nms-non-maximum-suppression-157be5bc61ca" target="_blank">blog.cubed.run</a></em></figcaption>
</figure>

`IoU = (Area of Overlap) / (Area of Box 1 + Area of Box 2 - Area of Overlap)`

If the IoU is over 0.45, the NMS algorithm assumes both bounding boxes are looking at the exact same physical object. It deletes the duplicates and keeps the best one.

    def apply_nms(self, detections):
        """Apply Non-Maximum Suppression"""
        if len(detections) == 0:
            return []
        
        # Sort by confidence
        detections = sorted(detections, key=lambda x: x['confidence'], reverse=True)
        
        keep = []
        while len(detections) > 0:
            best = detections[0]
            keep.append(best)
            detections = detections[1:]
            
            # Remove overlapping boxes
            detections = [
                det for det in detections
                if self.iou(best['bbox'], det['bbox']) < self.iou_threshold # self.iou_threshold is 0.45
            ]
        
        return keep

I went with 0.45 because a high threshold like 0.90 will not catch boxes that are slightly offset, and a low threshold like 0.10 will accidentally delete detections of two separate cubes sitting close together.

### Temporal Smoothing
To stop the flickering, I wrote a `DetectionSmoother()` class. The logic works on a 3 frame rule.

        smoother = DetectionSmoother(max_age=15, min_hits=3)


An object is only reported to the robot if it has been detected in the exact same spot for 3 consecutive frames. If the object vanishes for a split second due to glare, the system remembers its last known location for up to 15 frames (about half a second) before dropping it. This created a completely stable output for the robot to use.

## The Orientation Challenge
Standard YOLO models draw Axis Aligned Bounding Boxes (AABB). These are rectangles that always align flat with the screen edges.

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/HBBplusFlicking.png" alt="AABB failing on rotated objects" style="display:block; float:none; width:90%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem; text-align:center;"><em>Axis-Aligned Bounding Box (AABB) drawing a large square</em></figcaption>
</figure>

This is a major issue for grasping, especially for cylinders in my project. If a cylinder is lying diagonally, the standard YOLO box will draw a giant square to cover the whole length. The gripper could either not open wide enough or close without actually grasping the object.

We need to know the true angle of the object to rotate the gripper properly, and a tight box to know the orientation of the object.

## Implementing Oriented Bounding Boxes (OBB)
There are two main ways to draw bounding boxes. Horizontal Bounding Boxes (HBB) are simple but ignore rotation. Oriented Bounding Boxes (OBB) wrap tightly around the object and give you the rotation angle, which is exactly what a gripper needs.

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/HBB_OBB%20.png" alt="Comparisons between HBB and OBB" style="display:block; float:none; width:90%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Comparisons between (a) OBB and (b) HBB representations. Source: <a href="https://www.researchgate.net/figure/Comparisons-between-(a)-OBB-and-(b)-HBB-representations-for-objects-a-OBB-representation-b_fig3_349583747" target="_blank">ResearchGate</a></em></figcaption>
</figure>

I looked into how modern research solves this. Many papers use massive deep learning models like SAM or GraspNet to predict pixel perfect grasping poses for multiple objects in clutter. While these work incredibly well, they require heavy GPU compute. I could not run them locally on my laptop CPU alongside real time object detection.

For example, recent studies like [22] Multi-object robotic grasping setup based on RGB-D sensor ([see literature review page](https://myuwaishin.github.io/final-year-blog/literature-review/)), tackle similar orientation challenges in complex clutter using OBB.

Since my workspace has brightly colored objects on a mostly plain background, one possible method is to use a classical computer vision approach. I take the YOLO crop and apply OpenCV hue thresholding to calculate the OBB. 

>See documentation here [OBB: Object Detection](https://docs.ultralytics.com/tasks/obb/)
{: .prompt-info}

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/OBB_cube.webp" alt="OBB Masking Process" style="display:block; float:none; width:70%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>OBB calculation using OpenCV</em></figcaption>
</figure>

Here is how the logic works.

1. I expand the YOLO bounding box crop by 6 pixels to make sure the whole object edge is included.
2. The script converts the crop to HSV color space (see picture below) and finds the dominant color in that region. It ignores the white/grey background. This means it adapts automatically whether the object is a red cube or a blue cylinder.
3. It creates a binary mask of only the pixels matching that dominant color.
4. I run OpenCV's `cv2.minAreaRect()` on those pixels. This function perfectly traces the physical edges of the object and returns the angle.

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/HSV-Color-Model-18.png" alt="HSV Color Model" style="display:block; float:none; width:60%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>HSV Color Model. Source: <a href="https://www.researchgate.net/figure/HSV-Color-Model-18_fig1_368231676" target="_blank">ResearchGate</a></em></figcaption>
</figure>

### Fixing the Text Label Bug
When I first ran this, the OBB was sometimes drawing boxes around the "cube 91%" text label. Because the text was drawn in bright green before the OBB logic ran, the green text pixels fell into the color range the thresholding was looking for. The `minAreaRect()` function wrapped around both the physical cube and the text label above it.

To fix this, I made a copy of the frame `analysis_frame = frame.copy()` at the very start of the loop. The OBB analyzer now runs its HSV masking on a clean frame without any UI text drawn on it.

## Results
The OBB logic works perfectly once the object is detected.

<div style="display:flex; justify-content:center; gap:2rem; flex-wrap:wrap; margin-bottom:0.5rem;">
  <figure style="display:flex; flex-direction:column; align-items:center; flex:1; min-width:280px; max-width:45%;">
  <img src="/assets/img/OBB-deployment2.png" alt="External view with OBB" style="display:block; float:none; width:100%;" />
    <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem; text-align:center;"><em>External eye-to-hand view with OBB</em></figcaption>
  </figure>
  <figure style="display:flex; flex-direction:column; align-items:center; flex:1; min-width:280px; max-width:45%;">
     <img src="/assets/img/OBB_deployment.png" alt="Eye-in-hand view with OBB" style="display:block; float:none; width:100%;" />
    <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem; text-align:center;"><em>Eye-in-hand view with OBB</em></figcaption>
  </figure>
</div>

Looking at the first image, the orange cylinder lying diagonally, the script calculated an angle of 95 degrees and an aspect ratio of 2.9. The upright green cylinder returned an angle of 77 degrees and an aspect ratio of 1.9. 

The yellow center axis arrows in the images confirm the math matches the physical orientation. The robot now gets the exact X and Y pick coordinates and the precise gripper angle needed to approach without colliding.

>Note: The blue cubes in the second image struggle with detection as the model is confused between the object and the plain blue background.

## Next Steps
Now that the 2D object detection is stable and outputting accurate angles, the pipeline is almost complete. 

The last step is moving from 2D pixel coordinates to actual 3D millimeter space. My next post will cover camera calibration, pose estimation, and calculating the camera to robot transformation.

---

**Datasets and Model Weights**
- Find the datasets here: [Datasets V3](https://livemdxac-my.sharepoint.com/:u:/r/personal/ms3433_live_mdx_ac_uk/Documents/FYP%20Datasets/Datasets_V3.zip?csf=1&web=1&e=1txfEI)
- Find the V2 model weights and results here: [Models V2](https://livemdxac-my.sharepoint.com/:f:/r/personal/ms3433_live_mdx_ac_uk/Documents/FYP%20Datasets/Models_V2?csf=1&web=1&e=MiEYdw)

**Project repository:** [github.com/MyuWaiShin/Final_Year_Project_2026](https://github.com/MyuWaiShin/Final_Year_Project_2026)
