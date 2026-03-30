---
title: "Slip Monitor Implementation"
date: 2026-02-25 12:00:00 +0000
categories: [Project Updates, Hardware]
tags: [slip detection, ur10, rg2, gripper, force feedback]
author: myuwaishin
pin: false
last_modified_at: false
---

This post covers the implementation of a slip monitor during the transfer phase of the pick-and-place pipeline.

---

## The Problem: No Continuous Force Feedback

The RG2 gripper has a built-in force sensor that stops the jaws from closing once the target contact force is reached. This is enough to grasp an object, but it is not enough to know if the grasp is still holding. Once the gripper stops closing, the system gets no further updates from the sensor.

This is the core issue for slip detection. The RG2 only gives a binary signal, the `DI8` digital pin goes HIGH when force contact is reached, and that is the end of the feedback. It does not continuously stream a live force value. There is no analogue voltage that updates in real time to say "force is now dropping" or "the grip is weakening".

Without continuous feedback, if an object slips after the initial grasp, the system has no way of knowing. The gripper will just sit at its last commanded position.

---

## First Attempt: FSR Sensors

My first thought was to add Force Sensitive Resistors (FSRs) to the gripper fingers. Research in this area, such as [[23] Calandra et al. (2018)](https://arxiv.org/abs/1802.10153) used tactile arrays embedded in the gripper to produce a continuous surface-pressure signal. When an object slips, the contact area shifts and the resistance distribution changes, providing a direct and immediate slip signal.

The appeal of FSRs is that they give exactly this kind of continuous readout. As more force is applied, the resistance of the sensor drops according to its characteristic curve, producing a changing voltage I can read live.

I bought a set of FSRs and wired up a simple test circuit: FSR in series with a 10k pull-down resistor, reading the voltage divider output on the Arduino's analogue pin `A0`.

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/fsr_circuit_arduino.jpeg" alt="FSR Circuit" style="display:block; float:none; width:70%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>FSR voltage divider circuit on Arduino</em></figcaption>
</figure>

The script reads the raw ADC value from the Arduino over serial, converts it to voltage, and calculates the resistance live:

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/fsr_code.png" alt="FSR Circuit" style="display:block; float:none; width:70%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>FSR Circuit</em></figcaption>
</figure>

```python
import serial
import time

ser = serial.Serial('COM5', 9600, timeout=1)
time.sleep(2)

print("Reading FSR... press Ctrl+C to stop")
print("-" * 50)

while True:
    try:
        ser.reset_input_buffer()
        line = ser.readline().decode('utf-8').strip()
        if not line:
            continue
        raw = int(line)
        voltage = raw * (5.0 / 1023.0)
        if raw > 10:
            resistance = 10000.0 * (5.0 - voltage) / voltage
            print(f"Raw: {raw:4d}  Voltage: {voltage:.2f}V  Resistance: {resistance:.0f} ohms  ← CONTACT")
        else:
            print(f"Raw: {raw:4d}  — no contact")
    except (ValueError, UnicodeDecodeError):
        pass
    except KeyboardInterrupt:
        print("\nStopped.")
        ser.close()
        break
```

My laptop's Arduino IDE serial plotter was not available during this test, so I could not view the resistance as a graph but only as raw terminal output. The video below shows the live readings:

<figure style="display:flex; flex-direction:column; align-items:center;">
  <video width="80%" controls autoplay loop muted playsinline>
    <source src="/final-year-blog/assets/img/fsr_reading_vid.mp4" type="video/mp4">
    <p>Video not supported — <a href="/final-year-blog/assets/img/fsr_reading_vid.mp4">click here to download</a>.</p>
  </video>
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>FSR readings showing resistance changes with applied force</em></figcaption>
</figure>

The output showed that resistance does drop as force increases, the principle works. When tested by pinching the sensor with my fingers, it responded clearly. However, several practical problems ruled it out for this application:

- **Threshold behaviour:** The FSR does not give a gradual response. The reading stays at zero until a significant minimum force is applied, then jumps suddenly. A light touch produces nothing, which makes it useless for detecting the small force changes during a slip.
- **Noisy signal:** Even with a consistent grip, the resistance fluctuated significantly rather than settling, making it unreliable as a trigger.
- **Coverage mismatch:** The FSR's active area is smaller than the gripper jaw face. Contact outside that area produces no reading at all.
- **Hardware overhead:** Using FSRs requires an external microcontroller (Arduino) wired to the gripper, adding hardware complexity and a separate connection to manage.

The FSR could not reliably distinguish between "object is slipping" and "object is barely in contact". It was eliminated.


## Working Alternative: Monitoring Gripper Width

After the FSR tests, I thought about using the gripper's own internal measurement instead.

The RG2 already outputs the jaw separation as a voltage on its analogue line (`Analog In 2` on the UR controller). This maps linearly to the physical gap between the jaws in millimetres. If an object is held, this width stays stable. If the object slips out, the jaws will follow the object down and the width will drop.

**The key insight:**

> If the gripper is continuously commanded to close and an object is holding the jaws apart, the measured width is stable. If that object slips away, the jaws have nothing to resist them and the width will fall. Width drop = object loss.

This only works if the gripper is under an active, repeated close command after the initial grasp. The RG2 does not have a built-in "maintain grip" mode, once it stops closing, it sits idle. So I needed to program the robot to keep issuing close commands in a loop.

---

## Gripper Close Loop

I wrote a short program on the UR teach pendant called `close_loop.urp`. The logic is as follows:

```
┌──────────────────────────────────────┐
│         close_loop.urp starts        │
└──────────────┬───────────────────────┘
               │
               ▼
     ┌─────────────────────┐
     │  Command: RG2 Close │◄─────────────────────┐
     └──────────┬──────────┘                      │
                │                                 │
                ▼                                 │
     ┌─────────────────────────────┐              │
     │ Force contact detected?     │              │
     ├─────────────────────────────┤              │
     │ YES → Jaws stop             │              │
     │       (safety sensor halt)  │              │
     │                             │              │
     │ NO  → Jaws continue to 0mm  │              │
     └──────────┬──────────────────┘              │
                │                                 │
                ▼                                 │
     ┌─────────────────┐                          │
     │   Wait 0.2s     │                          │
     └────────┬────────┘                          │
              │                                   │
              └──────────── repeat ───────────────┘
```

The loop runs indefinitely. The gripper closes, the safety sensor stops it on contact, it waits 0.2s, then commands a close again. If the object ever slips, the very next close command finds empty jaws and the gripper closes further — the width drops, and that event is what the Python script detects.

---

## Controller Script

The controller uses multithreading because two things must happen in parallel: one thread continuously reads the robot state (jaw width and force flag) from port 30002, while another thread monitors that data for slip events. Control commands (open/close via `.urp` files) are sent separately through the Dashboard Server on port 29999.

| Thread | Port | Role |
|--------|------|------|
| `_recv_thread` | 30002 (Secondary Client) | Reads robot state: jaw width from `AI2`, force flag from `DI8` |
| `_monitor_thread` | — | Checks for slip conditions every 0.3s |
| Main thread | 29999 (Dashboard Server) | Sends `stop` / `load` / `play` commands to run `.urp` files |

The full implementation is here: [grip_control_with_slip_detection.py](https://github.com/MyuWaiShin/Final_Year_Project_2026/blob/main/UR10/grip_control_with_slip_detection.py)

**Detection logic:**

1. `start_closing()` plays `grip_close.urp` and waits 2.5s before monitoring begins (so the initial close motion is not mistaken for a slip event).
2. The monitor thread checks state every 0.3s. When `DI8 HIGH` and `width > 12mm`, it records that an object is being held.
3. If at any later point the jaws fully closed against nothing, it flags `[SLIP DETECTED]` and halts the loop.

---

## Terminal Outputs

**Scenario 1: Gripper misses the object entirely**

Jaws close to ~10.5mm with no object. `DI8` fires on jaw-to-jaw contact.

```text
Enter command (c/o/s/q): c
Closing gripper...
  Fully closed (10.5 mm) — stopping loop.

--- GRASP RESULT ---
Width:           10.5 mm
Voltage (AI2):   0.354 V
DI8 Force Limit: True  (HIGH = jaws touching something)

NO OBJECT — gripper fully closed (10.5 mm)
--------------------
```

**Scenario 2: Successful grasp**

Object held between jaws. `DI8` HIGH and width is well above 11mm.

```text
Enter command (c/o/s/q): c
Closing gripper...
  Holding object at 34.2 mm — loop running.

--- GRASP RESULT ---
Width:           34.2 mm
Voltage (AI2):   0.892 V
DI8 Force Limit: True  (HIGH = jaws touching something)

OBJECT DETECTED  (34.2 mm)
  Slip detection active — will auto re-close if object drops.
--------------------
```

**Scenario 3: Object slips during transfer**

Object was held, then yanked free. On the next loop, the jaws close fully and the slip is flagged.

```text
Enter command (c/o/s/q): c
Closing gripper...
  Holding object at 34.2 mm — loop running.

--- GRASP RESULT ---
Width:           34.2 mm
Voltage (AI2):   0.892 V
DI8 Force Limit: True  (HIGH = jaws touching something)

OBJECT DETECTED  (34.2 mm)
  Slip detection active — will auto re-close if object drops.
--------------------

[SLIP DETECTED] Object dropped — gripper closed fully.
```

---

## Slip Detection Pipeline: Stages Overview

This slip monitor completes Stage 1 of the failure detection pipeline (purely physical sensing). Stage 2 integrates a binary gripper state classifier to verify gripper state from the camera. Together they form the full multi-modal failure detection system.

<figure style="display:flex; flex-direction:column; align-items:center;">
  <img src="/assets/img/failure_detection_flowchart.png" alt="Failure Detection Flowchart" style="display:block; float:none; width:90%; margin:0 auto;" />
  <figcaption style="font-size:0.85rem; color:#666; margin-top:0.4rem;"><em>Failure detection pipeline: Stage 1 (physical) and Stage 2 (visual)</em></figcaption>
</figure>

**Stage 1 — Physical signals:**

| Check | Pass condition | Fail action |
|-------|---------------|-------------|
| DI8 HIGH and width > 12mm | Force contact confirmed with object between jaws | No contact or fully closed → re-attempt |
| Width during transfer | Stable | Width drops → slip detected, re-grasp |

**Stage 2 — Visual classification:**

| Check | Pass condition | Fail action |
|-------|---------------|-------------|
| Gripper image | Classified as holding | Classified as empty → re-attempt |

---

---

## Comparison Table: 3 Methods

| | Tactile Sensor Array | FSR Sensor | Gripper Width Monitor |
|---|---|---|---|
| **Signal type** | Continuous pressure map | Continuous resistance (voltage) | Continuous analogue voltage (jaw position) |
| **Slip sensitivity** | High, detects contact shift across surface | Low, threshold jump with no gradual reading | Medium, detects jaw movement directly |
| **Noise** | Low | High, fluctuates under light load | Low, stable analogue line |
| **Hardware needed** | Dedicated tactile array and controller | FSR + microcontroller + wiring to robot | None, signal already available from RG2 |
| **Response speed** | Fast (~ms) | Fast (~ms) but unreliable below threshold | Bounded by close-loop interval (0.2s) |
| **Complexity** | High, custom integration | Medium, external circuit required | Low, reads existing robot I/O |
| **Used in this project** | No (cost and integration overhead) | Tested, rejected | ✓ Yes |

The width monitor is not a perfect analogue of pressure, it does not measure force directly. But it uses the analogue signal that is already available from the RG2 (`Analog In 2`, which encodes jaw separation as a voltage) as a proxy. If an object is held, the width is stable. If it slips, the width drops. That is the same logical outcome that a tactile sensor provides, using existing hardware with no additional components.

---

## Next Steps

The slip monitor logic is validated in isolation. The next step is integrating Stage 1 (width monitoring) with Stage 2 (visual classification) into a unified failure detection pipeline that runs automatically during every pick-and-place cycle.
