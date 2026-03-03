---
title: "Setting Up the OAK-D Lite & Understanding Stereo Depth"
date: 2026-02-14 18:00:00 +0000
categories: [Project Updates, Perception]
tags: [oak-d, depthai, camera, stereo-depth, perception, ur10]
author: myuwaishin
pin: false
last_modified_at: false
---

Week 4 was supposed to be the week I finished the perception pipeline. That didn't happen. Getting the OAK-D Lite camera actually working on Windows took much longer than expected. I'm behind on the Gantt chart, but once the perception is solid the rest should go faster.

This post documents everything I figured out this week: getting the camera connected, exploring what it outputs in the DepthAI Viewer, and then properly understanding the physics and maths behind how the depth values are computed. This is background I need before I can write reliable pose estimation code.

---

## 1. Getting the OAK-D Lite Connected

The first hurdle was just getting Windows to recognise the camera in a way that Python could actually use.


**The boot timeout problem**

When `dai.Device()` is called, DepthAI uploads firmware to the Myriad X chip on the OAK-D and waits for it to boot. The default timeout is only a few seconds, which is sometimes not enough on Windows because the USB enumeration and firmware upload can be slower, and the SDK would throw a timeout error before the camera had finished booting.

The fix is to add this line at the very top of every script, before importing depthai:

```python
import os
os.environ["DEPTHAI_BOOT_TIMEOUT"] = "30000"  # wait up to 30 seconds
import depthai as dai
```

This tells the SDK to wait 30 seconds before giving up, which is more than enough time even on slower connections.

---

## 2. DepthAI Viewer

Before writing any code, I installed the DepthAI Viewer to verify the hardware was actually working and to understand what the camera outputs:

```bash
pip install depthai-viewer
python -m depthai_viewer
```

![DepthAI Viewer screenshot showing RGB, depth, mono streams and 3D point cloud](/assets/img/depthaiviewer.png)
*DepthAI Viewer connected to the OAK-D Lite via USB 2.1 (not USB 3.0 — limits bandwidth and frame rate)*

The viewer shows four panels simultaneously:

| **Panel** | **What It Shows** |
|---|---|
| Top-left (Color CAM_A 3D) | Live 3D point cloud. Coloured dots are real 3D positions computed from disparity. The pyramid wireframe is the camera's field of view. |
| Top-right (Left CAM_B 2D) | Greyscale feed from the left mono camera. Used for stereo depth, not detection. |
| Bottom-left (Color CAM_A 2D)(2) | RGB feed with built-in YOLO running on the Myriad X chip detects keyboard (80%) and mouse (84%) on the desk. |
| Bottom-centre (Color CAM_A 2D) | The RGB view colourised by depth using the JET colourmap (cyan = close, red = far). Black = no valid stereo match. |
| Bottom-right (Right CAM_C 2D) | Greyscale feed from the right mono camera. The slight horizontal offset between Left and Right is the key to stereo calculations. |
| Right panel (Device Settings) | All three cameras active. Color at 1080p, Left and Right mono at 400p, all at 15 FPS. |

The bottom-centre panel shows the depth map colourised with the JET colourmap. Close objects (small mm value) map to the low end of JET = cyan/blue. Far objects (large mm value) map to the high end of JET = orange/red. The cup (closest object on the desk) appears cyan. The keyboard and monitor (further away) appear orange and red. Black pixels = zero depth (no stereo match found).

This confirmed the hardware was working and I familiarised myself with the depth colour map, disparity output, and confidence filtering controls.

---

## 3. How the OAK-D Lite Actually Works

> **DepthAI version note:** I'm running DepthAI v2.28.0.0
```bash
python -c "import depthai; print(depthai.__version__)"
```
This is the v2 API. All code uses `dai.Pipeline()` and `dai.node.*`. 
Docs: [docs.luxonis.com/software/api/python/](https://docs.luxonis.com/software/api/python/)
{: .prompt-info }

### The Three Cameras

The OAK-D Lite has three physically separate cameras on one rigid board:

| Camera | Type | Sensor | Role |
|---|---|---|---|
| **CAM_A** | Centre | 13MP Rolling-Shutter Colour | RGB output, YOLO detection, pose estimation |
| **CAM_B** | Left | 1MP Global-Shutter Mono | Left stereo input (depth only) |
| **CAM_C** | Right | 1MP Global-Shutter Mono | Right stereo input (depth only) |

The mono cameras (CAM_B and CAM_C) are global-shutter, meaning the entire frame is captured at a single instant. This matters for stereo depth because if they were rolling-shutter, moving objects would appear slightly skewed differently in the left vs. right image, which would break the stereo matching algorithm.

**Rolling shutter vs. global shutter:**

<div style="display:flex; flex-direction:column; align-items:center; text-align:center;">
  <img src="/assets/img/rolling and global image.webp" alt="Rolling vs global shutter" style="display:block; float:none; width:50%; margin:0 auto;" />
  <em style="font-size:0.85rem; color:#666; margin-top:0.3rem;"><br>
  Source: <a href="https://kodifly.com/shutter-types-in-cameras-rolling-vs-global-shutters">kodifly.com</a></em>
</div>

<div style="display:flex; flex-direction:column; align-items:center; text-align:center; margin-top:1rem;">
  <img src="/assets/img/rolling-shutter-vs-global-shutter.jpg" alt="rolling vs global shutter" style="display:block; float:none; width:50%; margin:0 auto;" />
  <em style="font-size:0.85rem; color:#666; margin-top:0.3rem;"><br>
  Source: <a href="https://thesmartphonephotographer.com/mobile-camera-shutter-speed/">thesmartphonephotographer.com</a></em>
</div>

For the stereo pair used for depth, the global shutter ensures both cameras see the scene in a consistent, undistorted frame at the same moment, which is critical for accurate disparity computation.

> Read more: [Luxonis — Shutter Type](https://docs.luxonis.com/hardware/platform/sensors/shutter-type/)
{: .prompt-info }

---

### The Baseline

<img src="/assets/img/baseline_distance_oakdlite.png" alt="OAK-D Lite baseline distance diagram" style="display:block; margin:0 auto; width:50%;" />
<p style="text-align:center; font-size:0.85rem; color:#666; margin-top:0.3rem;"><em>Source: <a href="https://github.com/luxonis/oak-hardware/blob/master/DM9095_OAK-D-LITE_DepthAI_USB3C/Datasheet/OAK-D-Lite_Datasheet.pdf">OAK-D Lite Datasheet</a></em></p>

The baseline **BL** is the horizontal distance between the optical centres of the two mono cameras. On the OAK-D Lite it is 7.5 cm. A larger baseline = better accuracy at long range but a larger blind spot **Dv** up close. 

<img src="/assets/img/baseline_blindspot.png" alt="Baseline blind spot diagram" style="display:block; margin:0 auto; width:50%;" />
<p style="text-align:center; font-size:0.85rem; color:#666; margin-top:0.3rem;"><em>Source: <a href="https://docs.luxonis.com/software/depthai-components/nodes/stereo_depth#StereoDepth-Limitation">Luxonis — StereoDepth Limitation</a></em></p>

---

### Disparity — The Raw Measurement

![Disparity diagram from Luxonis DepthAI documentation](/assets/img/disparity.png)
*Source: [Luxonis — Configuring Stereo Depth](https://docs.luxonis.com/hardware/platform/depth/configuring-stereo-depth#how-baseline-distance-and-focal-length-affect-depth)*

The stereo matching algorithm (running on the Myriad X chip) scans both images simultaneously. For each pixel in the left image, it searches for the best-matching pixel in the right image along the same horizontal scanline. The horizontal distance between those two matching pixels is the disparity for that point, measured in pixels.

- Close objects → large disparity (the angular difference between the two cameras is more pronounced)
- Far objects → small disparity (the two cameras converge on the same apparent position)
- No match found → disparity = 0 (shown as black)

Standard mode gives disparity values from 0 (infinite distance / no match) to 95 pixels (closest measurable object).

---

### Depth from Disparity — The Equation

> Source: [Luxonis — Configuring Stereo Depth](https://docs.luxonis.com/hardware/platform/depth/configuring-stereo-depth#how-baseline-distance-and-focal-length-affect-depth)
{: .prompt-info }

Depth is computed from disparity using similar-triangle geometry:

```
Z (depth in cm) = focal_length_px × baseline_cm / disparity_px
```

**Calculating the focal length for OAK-D Lite at 400P (640×400 pixels):**

The mono cameras have a horizontal field of view (HFOV) of 71.9°:

```
focal_length_px = (image_width / 2) / tan(HFOV / 2)
                = (640 / 2) / tan(71.9° / 2)
                = 320 / tan(35.95°)
                ≈ 441.25 pixels
```

With baseline = 7.5 cm, the full equation becomes:

```
Z = 441.25 × 7.5 / disparity_px
```

Some worked examples:

| Disparity (px) | Depth | Interpretation |
|---|---|---|
| 95 px (max) | ~34.8 cm | Closest measurable object |
| 50 px | ~66 cm | Typical close pick range |
| 20 px | ~165 cm | Medium range |
| 10 px | ~331 cm | Getting noisy |

**Why accuracy gets worse at distance:**

Because depth and disparity are inversely proportional, a 1-pixel error in disparity has a much bigger effect at low disparity (far objects):

![Depth accuracy error chart](/assets/img/depth_error.png)
*Source: [Luxonis — Depth Accuracy](https://docs.luxonis.com/hardware/platform/depth/depth-accuracy/)*

**Minimum depth (MinZ):** At maximum disparity (95px) the minimum depth is `441.25 × 7.5 / 95 ≈ 34.8 cm` (anything closer than ~35 cm is in the blind spot).

**Maximum depth (MaxZ):** Theoretical ceiling from the OAK-D Lite docs:
```
MaxZ = (baseline/2) × tan(90° − HFOV/HPixels) ≈ 38 metres
```
In practice, beyond ~10m the noise is too high to be useful.

**Ideal working range:**

The ideal working range stated by Luxonis for OAK-D Lite is ~40 cm to 12 m.

For this project, objects are typically 40–100 cm from the camera, firmly in the ideal zone.

> Further reading: [Stereo depth perception](https://docs.luxonis.com/hardware/products/OAK-D%20Lite#Stereo%20depth%20perception)
{: .prompt-info }

---

### Depth Confidence Threshold

Every pixel in the disparity map gets a confidence score from 0–255. Counterintuitively, 0 = highest confidence, 255 = lowest confidence (Luxonis convention — 0 means "best match found").

Two built-in presets:

```python
# HIGH_ACCURACY: threshold = 200 — fewer valid pixels, more reliable
stereo.setDefaultProfilePreset(dai.node.StereoDepth.PresetMode.HIGH_ACCURACY)

# HIGH_DENSITY: threshold = 245 — denser depth map, some noise
stereo.setDefaultProfilePreset(dai.node.StereoDepth.PresetMode.HIGH_DENSITY)
```

> Source: [Luxonis — Stereo Depth Confidence](https://docs.luxonis.com/projects/api/en/latest/components/nodes/stereo_depth/)
{: .prompt-info }

For pick-and-place I'm using `HIGH_ACCURACY`. It's more important that the depth reading at the object centre is trustworthy than that every pixel has a value. When sampling depth I use a small median patch (5–10px radius) around the bounding box centre to smooth out any remaining unreliable pixels.

---

### Placing textured mat on the floor

The stereo matching algorithm needs to find the same identifiable feature in both images. When a surface is completely uniform, like the plain blue lab floor, there are no features to match. Every patch looks identical to every other patch, so the algorithm has no way to tell how far any point has shifted horizontally. It returns disparity = 0 for all those pixels.

To tackle this, I placed a textured mat on the floor to give the algorithm something to match.

---

### Back-Projection: Getting Real-World X, Y from Pixels

Once depth Z is known for a pixel, real-world X and Y (relative to the camera) are computed using the camera's intrinsic parameters:

```python
X = (pixel_x - cx) * Z / fx   # positive = right
Y = (pixel_y - cy) * Z / fy   # positive = downward
# Z = depth from stereo (metres)
```

Where `fx`, `fy` are focal lengths in pixels and `cx`, `cy` is the principal point (roughly the image centre). These four values are factory-calibrated and stored on the OAK-D's onboard flash. They are read at runtime via `device.readCalibration()`.

---

### What Is a Point Cloud?

The 3D panel in the DepthAI Viewer shows a point cloud. Every pixel that has valid depth is converted into a real-world 3D position (X, Y, Z) using the back-projection equations above. Each pixel becomes one dot in 3D space; viewed together they form the 3D shape of the scene.

For this project I'm not using the full point cloud, just the single-pixel depth reading at each object's detected centre to get (X, Y, Z) for pose estimation. But understanding the full 3D reconstruction helps with debugging when depth values look wrong.

---

## Next Steps

Now that the camera is working and I understand the depth output properly, the next step is to test depth sampling with my own trained YOLO model, then integrate live object detection with depth readings to get 3D object positions. 

---

**Project repository:** [github.com/MyuWaiShin/Final_Year_Project_2026](https://github.com/MyuWaiShin/Final_Year_Project_2026)
