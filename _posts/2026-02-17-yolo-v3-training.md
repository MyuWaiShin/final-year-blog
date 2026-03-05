---
title: "Training YOLO Models V2 for OAK-D Lite Deployment"
date: 2026-02-17 18:00:00 +0000
categories: [Project Updates, Perception]
tags: [yolo, object-detection, training, oak-d, dataset, depthai]
author: myuwaishin
pin: false
last_modified_at: false
---

After familiarising myself with the OAK-D Lite camera and confirming the stereo depth, the next step is to run my own detection model on it. The dataset used for the V1 model was not trained with the OAK-D camera, and testing it revealed that the depth readings were unreliable on the plain lab floor. So I went back and built a new dataset (V3) by collecting data using the OAK-D Lite directly. I then trained three models across different architectures.

---

>All the scripts used for this project can be found in the [Object Detection Pipeline](https://github.com/MyuWaiShin/Final_Year_Project_2026/tree/main/Object%20Detection%20Pipeline) folder and the dataset is available at [Datasets_V3](https://github.com/MyuWaiShin/Final_Year_Project_2026/tree/main/Datasets_V3).
{: .prompt-info }

## Data Collection

The objects being detected are small styrofoam cubes and cylinders ranging in size from roughly 25 mm to 85 mm.

One thing I had to account for from the depth testing was the plain blue lab floor. The OAK-D stereo depth system needs texture in the scene to compute disparity between its two cameras. On a uniform surface the stereo matching fails, and depth values were missing or noisy. To tackle this, I placed a textured mat under the objects to give the stereo matching something to work with. I recorded in two conditions, some clips on the bare blue floor (which is fine for colour detection on its own) and some with a textured mat placed under the objects (which is needed for reliable depth readings). This means the detection model works whether or not a mat is present.

For each object class, I recorded around 20-second clips using `record_data.py` from multiple perspectives and three distances (30 cm, 60 cm, and 90 cm from the ground), with different object orientations at each distance (left-facing, right-facing, top view). A few extra recordings at random heights and angles were added to increase variety. In total, I recorded 12 videos without the mat and 6 videos with the mat, covering both classes.

Because the OAK-D pipeline was running on my laptop CPU at the time of recording, the actual frame throughput was slower than native. Each 20-second clip produced roughly 7 seconds worth of real footage, but every frame was still usable for training.

---

## Data Preparation V3

>Find the scripts at [Object Detection Pipeline/Data_Preparation_V3/scripts](https://github.com/MyuWaiShin/Final_Year_Project_2026/tree/main/Object%20Detection%20Pipeline/Data_Preparation_V3/scripts).
{: .prompt-info }

**Step 1 — Frame Extraction**

The first step was extracting every single frame from all recordings using `1_extract_and_combine.py`. This script extracts all frames (no skipping) from the new recordings as well as the earlier V1 and V2 recordings, then merges them into one large combined dataset. Pulling in the earlier data means the model still has exposure to a wider range of conditions from previous experiments.

**Step 2 — Train/Val/Test Split**

The images were split using `2_split_dataset.py` at a 75 / 15 / 15 ratio (train / val / test). The val set is used to monitor training during runs. The test set is held out entirely for final evaluation.

**Step 3 — Auto-Annotation with Grounding DINO**

Labelling was done automatically using Grounding DINO, using the script `3_auto_annotation.py`. Grounding DINO is a zero-shot open-vocabulary detection model hosted on Hugging Face. It takes a text prompt and finds bounding boxes in images without needing to be pre-trained on those specific classes. The script queries it with prompts for "cube" and "cylinder" and outputs YOLO-format `.txt` label files (normalised bounding box coordinates) for each image. This saved an enormous amount of time compared to manual labelling at the scale of my dataset.

All dependencies for the data preparation pipeline are in `requirements_dataprep.txt` and `requirements_annotation.txt`. Setup instructions are in `SETUP.md` in the Object Detection Pipeline folder.

**Verifying the Dataset**

At any point I could run `dataset_stats.py` to check the current state of the dataset. This script prints a breakdown of image counts per class per split, annotation coverage (how many labels exist vs how many images), and a readiness checklist. This is a useful step for catching missing label files and fiding out total amount of datatset before starting a training run.

My dataset ended up with 18,942 annotated instances across the two classes, fully split and ready to train.

---

## Training Three Models

I trained three models for different deployment targets:

| Model | Why |
|---|---|
| **YOLOv8n** | Best accuracy baseline. Anchor-free architecture, consistently best mAP in benchmarks. Native Ultralytics support. |
| **YOLOv5n** | Most documented and widely tested model for the OAK-D Lite. Established guides for converting to `.blob` via blobconverter. |
| **YOLO26n** | Optimised for faster CPU inference. Targets laptop CPU inference directly. |

Note: YOLOv6n was originally planned as it is also documented for OAK-D compatibility, but I dropped it as I faced compatibility issues with the outdated v6 repo when training.


All three models were trained on the NVIDIA RTX 5080 available in the Ritterman lab. Total training time for all three was approximately 4 hours. Each model ran for 100 epochs with early stopping at a patience of 20, meaning training stops if validation loss does not improve for 20 consecutive epochs.

---

## Dataset Configuration

The `data.yaml` file is shared across all three models:

```yaml
path: <path to Datasets_V3>/data   # path to the dataset
train: images/train
val:   images/val
test:  images/test
nc: 2
names: ['cube', 'cylinder']
```
---

## Training Script

All three models are trained through a single script `train.py`.

>Find the script at [Train/train.py](https://github.com/MyuWaiShin/Final_Year_Project_2026/tree/main/Train).
{: .prompt-info }

At the top of the file there is a small configuration section, where I set the model, dataset, image size, batch size, epochs, patience, and device.

```python
DATA_YAML   = r"path/to/Train/data.yaml"
IMG_SIZE    = 640
BATCH       = 32        # RTX 5080 handles this comfortably
EPOCHS      = 100
PATIENCE    = 20        # early stopping if val loss does not improve
DEVICE      = 0         # 0 = first GPU
```

**CLI commands:**

```bash
# Train a single model
python train.py --train v8
python train.py --train v26
python train.py --train v5

# Train all three sequentially
python train.py --train all

# Train then export to ONNX
python train.py --train all --export

# Skip training, just export already-trained weights
python train.py --train none --export

# Compare metrics across all trained models
python train.py --train none --compare

# Full pipeline
python train.py --train all --export --compare
```

The `--compare` flag reads the `results.csv` from each model and prints a side-by-side summary of mAP@0.5 and mAP@0.5:0.95, with a flag showing which models support OAK-D blob conversion.

---

## Augmentation

The same augmentation settings were used across all three models so that performance differences reflect the architecture, not the training conditions.

```python
AUG = dict(
    degrees     = 45,     # random rotation up to 45 degrees
    translate   = 0.2,    # translate by up to 20% of the image
    scale       = 0.5,    # zoom in/out by up to 50%
    fliplr      = 0.5,    # horizontal flip, 50% probability
    flipud      = 0.1,    # vertical flip, 10% probability
    perspective = 0.001,  # subtle perspective warp
    hsv_h       = 0.1,    # hue jitter
    hsv_s       = 0.4,    # saturation jitter
    hsv_v       = 0.3,    # brightness jitter
    mosaic      = 1.0,    # mosaic always on
    mixup       = 0.05,   # mixup on 5% of batches
    erasing     = 0.1,    # random erasing on 10%
)
```

The heavy rotation and scale augmentation are use because objects were recorded at varying heights (30cm, 60cm, 90cm) and different orientations, so the model needs to generalise across quite different viewpoints. The colour jitter (HSV) helps account for the lighting differences between the textured matt surface and the plain blue floor recordings. Mosaic tiles four images together per training sample, which is especially useful for smaller nano-scale models.

For YOLOv8n and YOLO26n, `AUG` is passed directly as keyword arguments via the Ultralytics Python API. For YOLOv5n, the same values are written out to a `hyp_v5.yaml` file, since YOLOv5 reads its hyperparameters from YAML rather than accepting them as arguments.

---

## ONNX Export

After training, all models are exported to ONNX using the `--export` flag:

```python
model.export(format='onnx', imgsz=640, opset=12, simplify=True)
```

The ONNX file is saved alongside the weights at `runs/train/<model>/weights/best.onnx`.

Trained weights are stored at [Models_V2.zip](https://livemdxac-my.sharepoint.com/:u:/r/personal/ms3433_live_mdx_ac_uk/Documents/FYP%20Datasets/Models_V2.zip?csf=1&web=1&e=mn5tBQ)

---

## Training Curves

>All model results can be found at:
[Models_V2.zip](https://livemdxac-my.sharepoint.com/:u:/r/personal/ms3433_live_mdx_ac_uk/Documents/FYP%20Datasets/Models_V2.zip?csf=1&web=1&e=mn5tBQ)
{: .prompt-info }

**YOLOv8n**

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/v8n_results.png" alt="YOLOv8n training curves" style="display:block; float:none; width:90%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>YOLOv8n training curves over 100 epochs</em></figcaption>
</figure>

Box loss, classification loss, and DFL loss all drop steadily across 100 epochs. There is no divergence between training and validation loss, which means the model generalised well and did not overfit. Precision approaches near 1.0, recall stabilises around 0.92 to 0.95, and mAP@0.5 reaches approximately 0.99 by the end of training.

---

**YOLO26n**

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/26n_results.png" alt="YOLO26n training curves" style="display:block; float:none; width:90%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>YOLO26n training curves over 100 epochs</em></figcaption>
</figure>

YOLO26n shows a very similar convergence pattern to v8n. One visible difference is that the DFL loss values for v26n are in a much smaller numerical range (around 0.001 to 0.005) compared to v8n (around 0.8+). This reflects how YOLO26's bounding box regression head is structured differently internally. Final precision and recall numbers are comparable to v8n.

---

**YOLOv5n**

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/v5n_results.png" alt="YOLOv5n training curves" style="display:block; float:none; width:90%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>YOLOv5n training curves over 100 epochs</em></figcaption>
</figure>

YOLOv5n converges notably faster than the other two. The loss curves flatten well before epoch 60 and all metrics reach their final values earlier. This is partly because v5n uses an anchor-based detection head, which tends to converge faster on well-structured datasets. The mAP@0.5:0.95 is slightly lower than v8n, which is expected since anchor-free models like v8 typically perform better at the stricter IoU thresholds.

---

## Dataset Labels

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/v8n_Labels.jpg" alt="Dataset label distribution" style="display:block; float:none; width:80%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Label distribution and bounding box statistics for the V3 dataset</em></figcaption>
</figure>

The dataset has 11,428 cube instances and 7,514 cylinder instances, giving cubes roughly a 60/40 majority. The class imbalance is moderate and not extreme enough to cause serious training issues, but it is a possible reason the model performs slightly better on cubes than cylinders.

The bounding box centre positions are spread broadly across the frame, reflecting the variety of angles and positions from the recordings.

Bounding box sizes are tightly clustered in the width/height scatter, which makes sense as the objects were always a small portion of the 640x640 frame when recorded from typical working distances.

---

## Confusion Matrix Analysis

**YOLOv8n**

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/v8n_confusion_matrix_norm.png" alt="YOLOv8n confusion matrix" style="display:block; float:none; width:65%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>YOLOv8n normalised confusion matrix</em></figcaption>
</figure>

**YOLO26n**

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/26n_confusion_matrix_norm.png" alt="YOLO26n confusion matrix" style="display:block; float:none; width:65%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>YOLO26n normalised confusion matrix</em></figcaption>
</figure>

**YOLOv5n**

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/v5n_confusion_matrix.png" alt="YOLOv5n confusion matrix" style="display:block; float:none; width:65%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>YOLOv5n confusion matrix</em></figcaption>
</figure>

All three models have near-zero inter-class confusion between cube and cylinder. The main shared weakness is background false positives, where the model fires on empty background and labels it as an object, particularly as a cylinder.

| Model | Cube Recall | Cylinder Recall | Notes |
|---|---|---|---|
| YOLOv8n | 0.98 | 0.97 | Background FP split: 0.29 cube / 0.71 cylinder |
| YOLO26n | 0.98 | 0.97 | Background FP split: 0.27 cube / 0.73 cylinder |
| YOLOv5n | 0.98 | 0.94 | Cylinder recall slightly lower, consistent with anchor-based head |

YOLOv8n and YOLO26n are nearly identical in recall numbers. The difference is marginal. YOLOv5n matches on cube recall (0.98) but cylinder recall drops to 0.94, reflecting the anchor-based head reaching a slightly lower ceiling on the less-represented class.

---

## Test Inference

Both models detect the targets reliably across different positions, orientations, and distances during the test set inference.

<div style="display:flex; justify-content:center; gap:2rem; flex-wrap:wrap; margin-bottom:0.5rem;">
  <figure style="display:flex; flex-direction:column; align-items:center; flex:1; min-width:280px; max-width:45%;">
    <img src="/assets/img/v8n_test_inference.gif" alt="YOLOv8n test inference" style="display:block; float:none; width:100%;" />
    <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem; text-align:center;"><em>YOLOv8n test inference</em></figcaption>
  </figure>
  <figure style="display:flex; flex-direction:column; align-items:center; flex:1; min-width:280px; max-width:45%;">
    <img src="/assets/img/26n_test_inference.gif" alt="YOLO26n test inference" style="display:block; float:none; width:100%;" />
    <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem; text-align:center;"><em>YOLO26n test inference</em></figcaption>
  </figure>
</div>

The main issue visible in the inference is background false positives. The textured mat gets picked up and labelled as a cylinder in some frames. This matches what the confusion matrix showed, where the model tends to fire on textured backgrounds and classify them as cylinder.

The good thing is that the confidence scores on these false positives are usually low, around 0.3 to 0.5, whereas actual object detections come in at 0.85 or above. So setting a higher confidence threshold when deploying on the OAK-D Lite should filter most of them out without dropping the real detections.

## Next Steps

The three trained models are ready. The next step is to deploy the detection models on the OAK-D Lite and fine-tune parameters to get reliable detection on the actual targets in the lab environment.

---

**Project repository:** [github.com/MyuWaiShin/Final_Year_Project_2026](https://github.com/MyuWaiShin/Final_Year_Project_2026)
