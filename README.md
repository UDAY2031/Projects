# CHITTI 2.0 — Fault-Tolerant Hexapod

ESP32 + 2× PCA9685 + MPU6050 · Auto-recovery with cloud gait update

---

## Hardware

| Component | Address / Channel |
|-----------|------------------|
| ESP32 | main controller |
| PCA9685 Left  | I²C 0x40 — legs L1, L2, L3 |
| PCA9685 Right | I²C 0x41 — legs R1, R2, R3 |
| MPU6050 | I²C 0x68 — roll / pitch / yaw |

Leg numbering:

```
  L1 ──┐     ┌── R1     (Front)
  L2 ──┤ BOT ├── R2     (Middle)
  L3 ──┘     └── R3     (Back)
```

---

## Files

| File | Purpose |
|------|---------|
| `Chitti.ino` | Main ESP32 firmware — upload this |
| `Core-logic.ino` | Original baseline (no fault detection, kept for reference) |
| `cloud_bridge.py` | Laptop script — polls robot, pushes recovery gait |

---

## Flashing the firmware

1. Open `Chitti.ino` in Arduino IDE
2. Board: **ESP32 Dev Module**
3. Required libraries (install via Library Manager):
   - `Adafruit PWM Servo Driver Library`
   - `MPU6050` by Electronic Cats (or Jeff Rowberg)
4. Upload — robot auto-starts on boot

---

## Connecting

The robot creates its own WiFi access point:

```
SSID     : CHITTI-2.0
Password : hexapod123
```

Connect your laptop/phone to that network, then open:

```
http://192.168.4.1
```

Login password: **alien**

> The robot has no internet when you're connected to its WiFi.
> If you need internet on the laptop at the same time, use a USB-ethernet adapter
> or a second WiFi adapter for your home network.

---

## Control UI tabs

| Tab | URL | What it does |
|-----|-----|-------------|
| Control | `/` | D-pad movement, leg toggle, speed |
| Telemetry | `/telemetry` | Live roll / pitch graph + horizon |
| Config·AI | `/config` | Fault state, confidence scores, recovery params |

---

## Single-leg fault demo (step by step)

### 1. Start the robot
Power on → waits 1–2 s → stands up automatically.

### 2. Start walking
Open `http://192.168.4.1` → tap **FWD** — robot walks forward with all 6 legs.

### 3. Run the laptop bridge
On your laptop (still connected to CHITTI-2.0 WiFi), open a terminal:

```bash
pip install requests        # one-time install
python cloud_bridge.py
```

You should see live roll/pitch numbers scrolling.

### 4. Simulate a leg fault
In the browser UI, tap one of the leg buttons (e.g. **L2 Middle**).

What happens:
- Robot **stops walking**, smoothly lifts that leg up and tucks it
- Resumes walking on the remaining 5 legs
- Body tilts slightly toward the missing leg
- MPU detects the tilt (roll/pitch changes)

### 5. Cloud recovery
The Python script sees `tilt > 3.5°` and `fault = 1` (L2):
- Calls `compute_recovery()` — rule-based by default, replace with your model
- POSTs new gait parameters to `/gait` on the robot
- Robot applies: shorter stride, adjusted coxa offsets to shift COG
- Tilt gradually reduces back toward 0°

### 6. Re-enable the leg
Tap the same leg button again in the UI → leg comes back down → normal 6-leg gait resumes.

---

## Cloud / ML integration

The `compute_recovery()` function in `cloud_bridge.py` is the swap-in point for your model:

```python
def compute_recovery(fault_leg, roll, pitch):
    # Replace this block with your cloud API call:
    result = requests.post("https://your-model-api/predict",
                           json={"fault": fault_leg, "roll": roll, "pitch": pitch})
    return result.json()
    # Must return dict with keys: swing, lift, fm, tb, step_ms, cx0..cx5
```

### `/gait` endpoint (POST)

The robot accepts gait parameter updates at `POST http://192.168.4.1/gait` with form fields:

| Field | Range | Description |
|-------|-------|-------------|
| `swing` | 5–40 | Stride angle (degrees) |
| `lift` | 5–35 | Leg lift height (degrees) |
| `fm` | −15 to +15 | Femur stand bias |
| `tb` | −15 to +15 | Tibia stand bias |
| `step_ms` | 200–900 | Step duration (ms) — lower = faster |
| `cx0`–`cx5` | −30 to +30 | Per-leg coxa offset for COG shift |

### `/state` endpoint (GET)

Returns JSON with current robot state:

```json
{
  "status": "WALK FORWARD",
  "legs": [1, 0, 1, 1, 1, 1],
  "roll": 14.32,
  "pitch": 0.51,
  "yaw": -2.1,
  "fault": 1,
  "faultConf": 0.87,
  "pcaL": true,
  "pcaR": true,
  "mpu": true,
  "autoRecovery": true
}
```

`fault`: index 0–5 (L1–R3), or -1 if none detected.

---

## Serial commands (USB debug)

Connect at **115200 baud**:

| Key | Action |
|-----|--------|
| F / B / L / R | Walk / Turn |
| S | Stop |
| 1 / 2 / 3 | Slow / Normal / Fast |
| + / - | Stride longer / shorter |
| a c d e g h | Toggle legs L1 L2 L3 R1 R2 R3 |
| Z | Calibrate MPU (hold flat) |
| X | Toggle auto-recovery on/off |

---

## Known behaviour

- On boot the robot **auto-stands** (smooth startup animation).
- MPU calibration runs once on boot. Re-calibrate with the **Zero MPU** button on the Telemetry page if the robot is not level.
- Auto-recovery (`updateFaultDetection`) kicks in automatically if `tilt > 3.5°` for 500 ms — independent of the laptop bridge.
- The laptop bridge is only needed for the cloud/ML gait update; the robot will still do basic rule-based recovery on its own.
