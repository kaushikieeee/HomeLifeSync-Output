# Health Monitoring — Wearable Simulation & Condition Detection

> Placeholder system. There is **no real wearable**. Both the elder device
> (native Android) and the caretaker web app run the **same** simulated
> vitals stream and the **same** detection algorithm. Simulating a condition
> on either device triggers a loud, automatic warning to the caretaker.

## 1. Simulated vitals (continuous stream, updated every 2 s)

| Vital | Unit | Normal range |
|-------|------|--------------|
| Heart rate (`heartRate`) | bpm | 60 – 100 |
| Oxygen saturation (`spo2`) | % | 95 – 100 |
| Temperature (`temperature`) | °C | 36.1 – 37.2 |
| Respiratory rate (`respiratoryRate`) | breaths / min | 12 – 20 |
| Blood pressure (`systolic` / `diastolic`) | mmHg | ~120 / 80 (90–120 sys, 60–80 dia) |
| Blood glucose (`glucose`) | mg/dL | 70 – 180 |

Each vital is a **random walk** around the current scenario's target value
(a per-vital spread: HR/BP/glucose ±4, RR ±2, SpO₂ ±1, temperature ±0.15 °C,
all clamped), plus occasional variance spikes for arrhythmia/AFib looks. The
heart-rate variability (`hrv`) used by arrhythmia rules is the standard
deviation of the last 6 heart-rate samples.

When the stream is left on **NORMAL**, all values hover in the normal range
and nothing alerts.

## 2. Detection algorithm

Detection runs **every tick (2 s)** on the latest vital snapshot. Each rule
below either passes or not for that sample. A condition is only "confirmed"
when:

- **Hard rule (debounce 1, ~immediate):** an extreme value on a single sample
  (e.g. SpO₂ < 88, HR < 28).
- **Sustained rule (debounce 2, ~4 s):** the value stays in an abnormal band
  for **2 consecutive samples**, avoiding false alarms from a single spike.

If a rule confirms, its severity is `WARNING` or `CRITICAL`. When multiple
rules confirm at once, the highest severity wins (CRITICAL > WARNING); ties
are broken by rule order below.

### Rules (evaluated in order)

| # | Condition | Vital | WARNING threshold (debounce 2) | CRITICAL threshold (debounce 1) |
|---|-----------|-------|--------------------------------|---------------------------------|
| 1 | **Hypoxia** | SpO₂ | < 94 | < 88 (hard) |
| 2 | **Fever** | Temp | ≥ 38.0 | ≥ 39.5 (hard) |
| 3 | **Hypothermia** | Temp | ≤ 35.0 | ≤ 33.5 (hard) |
| 4 | **Hypertensive crisis** | BP | — | sys ≥ 180 or dia ≥ 120 |
| 5 | **Hypertension-urgent** | BP | sys ≥ 160 or dia ≥ 100 (debounce 2) | — |
| 6 | **Hypotension** | BP | sys < 90 (debounce 2) | sys < 80 or dia < 50 (hard) |
| 7 | **Tachypnea** | RR | ≥ 24 | ≥ 32 (hard) |
| 8 | **Bradypnea** | RR | ≤ 10 | ≤ 6 (hard) |
| 9 | **Hyperglycemia** | Glucose | ≥ 180 | ≥ 300 (hard) |
| 10 | **Hypoglycemia** | Glucose | < 70 | < 54 (hard) |
| 11 | **Myocardial infarction** (suspected) | HR + BP | — | HR ≤ 28 (hard) **or** HR ≤ 40 **and** sys < 95 |
| 12 | **Atrial fibrillation** | HR + HRV | — | HR ≥ 110 and HRV > 25 |
| 13 | **Tachycardia** | HR | ≥ 120 | ≥ 160 |
| 14 | **Bradycardia** | HR | ≤ 50 | ≤ 40 |
| 15 | **Arrhythmia** | HRV | HRV > 18 | HRV > 25 |

Rules 11–14 use heart rate; 15 uses variability only. HRV needs a history
window, so it is available after 6 samples (~12 s).

## 3. What happens when a condition is confirmed

### Elder device (native Android)
1. Loud **alarm** rings on the elder's phone (loops ~20 s).
2. **Full-screen high-priority notification** appears on the elder's screen
   with the condition name, severity and current vitals.
3. A **HEALTH alert is written to Firebase**:
   `devices/{deviceId}/alerts/{alertId}` with
   `{ type:"HEALTH", condition, severity, hr, spo2, temperature,
   respiratoryRate, systolic, diastolic, glucose, ts }`.
4. The elder app also streams all vitals into `devices/{deviceId}/status`
   (`heartRate`, `spo2`, `temperature`, …) every 2 s tick.

### Caretaker app (web / Firebase listener)
1. Subscribes to `devices/{deviceId}/alerts` via `onChildAdded`.
2. On any new HEART/HEALTH alert it raises a **loud warning**:
   - full-screen red **Medical Alert overlay** (pulsing),
   - repeating **siren** (Web Audio oscillators),
   - **vibration** pattern on supporting devices,
   - error toast with the condition + vitals.
3. The overlay stays until the caretaker presses **Acknowledge**.

### Local simulation (on the caretaker app itself)
Pressing a scenario button on the caretaker app:
1. Sets this device's simulated vitals to the scenario's abnormal targets
   (same engine runs locally), and
2. if an elder device is connected, **also sends the matching command** so
   the elder device goes into the same state — both sides then alert. Local
   and elder alerts are de-duplicated (same condition + severity within 20 s).

## 4. Simulation scenarios (buttons)

| cMd | Scenario | Targeted abnormal vitals | Expected detection |
|-----|----------|--------------------------|--------------------|
| `HRMI` | MI (Heart Attack) | HR 30, SpO₂ 92, BP 85/55 | Possible myocardial infarction (CRITICAL) |
| `HRAFIB` | Atrial fibrillation | HR 135, high variance | Atrial fibrillation (CRITICAL) |
| `HRTACHY` | Tachycardia | HR 165 | Tachycardia (CRITICAL) |
| `HRBRADY` | Bradycardia | HR 42 | Bradycardia (CRITICAL) |
| `HRARRHY` | Arrhythmia | HR 95, elevated variance | Arrhythmia (WARNING) |
| `HYPOXIA` | Hypoxia | SpO₂ 87 | Hypoxia (CRITICAL) |
| `FEVER` | High fever | Temp 39.5 | Fever (CRITICAL) |
| `HYPOTHERMIA` | Hypothermia | Temp 33.0 | Hypothermia (CRITICAL) |
| `BPCRISIS` | Hypertensive crisis | BP 188/128 | Hypertensive crisis (CRITICAL) |
| `HYPOTENSION` | Hypotension | BP 82/50 | Hypotension (CRITICAL) |
| `TACHYPNEA` | Tachypnea | RR 32 | Tachypnea (CRITICAL) |
| `BRADYPNEA` | Bradypnea | RR 5 | Bradypnea (CRITICAL) |
| `HYPERGLYCEMIA` | Hyperglycemia | Glucose 300 | Hyperglycemia (CRITICAL) |
| `HYPOGLYCEMIA` | Hypoglycemia | Glucose 48 | Hypoglycemia (CRITICAL) |
| `HRNORMAL` | Reset to normal | all normal | none |

Note: a few scenarios deliberately land on a **hard** threshold (debounce 1)
so the elder→caretaker warning fires almost immediately; the warm-band
thresholds exist to make realistic "borderline" behavior possible via the
simulator's random walk.

## 5. Cross-app command channel

Caretaker → elder uses the existing Firebase command path
(`devices/{id}/commands/{cmdId}`); the command strings above (`HRMI`, …)
are handled by the elder's `HealthHandler.simulate()` and are listed in
`IMPLEMENTED_COMMANDS` so the caretaker UI treats them as available.
