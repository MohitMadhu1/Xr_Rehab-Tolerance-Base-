# Hand Dual Logger — Quick Start Guide

This guide walks you through setting up and using the **HandDualLogger.cs** to record XR Hands + Ultraleap (Leap Motion) joint poses at a fixed rate and save them to a single JSON file.

---

## 1) Prerequisites
- Unity 2021+.
- **XR Hands** package (if you want HMD skeleton data).
- **Ultraleap/Leap** Unity Modules (if you want Leap camera data).

> The script includes two optional compile flags at the top:
> ```csharp
> //#define USE_XR_HANDS
> //#define USE_ULTRALEAP
> ```
> Leave them **commented** until each package is installed.

---

## 2) Add the script
- Save the file **exactly** as `HandDualLogger.cs` (not inside any `Editor/` folder).
- Add it to a persistent scene object (e.g., an empty GameObject named `DualHandLogger`).

---

## 3) Validate the Inspector
- With both flags commented, the component still shows up (providers disabled).
- If the Inspector is blank, open **Console** and fix compile errors (usually missing packages).

---

## 4) Enable providers after packages are installed
- Open `HandDualLogger.cs` and **uncomment** what you need:
  ```csharp
  #define USE_XR_HANDS
  #define USE_ULTRALEAP
  ```
- Save to recompile. The Inspector will now show the **Leap Provider** field.

---

## 5) Hook Ultraleap
- Add **LeapXRServiceProvider** (or **LeapServiceProvider**) to your rig.
- Drag that provider into the logger’s **Leap Provider** slot.

---

## 6) Configure sampling & output
- **Sample Hz**: 60 is a good start (try 90/120 if CPU allows).
- **Pretty Json**: off for speed/smaller files; on for readability.
- **File Base Name**: used as the prefix of the log file.
- **Session Tag**: any free text (user id, experiment code).

---

## 7) Run and what happens
- Press **Play**. The logger:
  - Starts a monotonic **Stopwatch** (session time).
  - Every `1 / SampleHz` seconds:
    - Reads XR joints (if enabled).
    - Reads the latest Leap frame (if enabled).
    - Stamps **session seconds**, **UTC ms**, and **IST** string.
    - Appends one object to a streaming **JSON array**.
- **Output path**: `Application.persistentDataPath/DualHandLog_YYYYMMDD_HHMMSS.json`.

---

## 8) Verify the output
Open the JSON after stopping Play. You should see:
```json
{
  "meta": { ... },
  "rows": [
    {
      "t_session_s": 0.016,
      "utc_unix_ms": 1730000000000,
      "ist": "2025-10-26 15:30:00.123 IST",
      "xr": { "available": true, "hands": [ ... ] },
      "leap": { "available": true, "leap_ts_us": 123456789, "hands": [ ... ] }
    }
  ]
}
```

---

## 9) Usage tips
- **Coordinates**: XR is in XR Origin/world; Leap is in the provider’s transform space (usually your rig). For research-grade comparison, add a calibration step later.
- **Performance**: JSON grows fast. Roll logs per scene or per N minutes for long sessions.
- **Timestamps**: You already log **UTC** and **IST** plus a monotonic **session seconds** value—perfect for later alignment.

---

## 10) Common pitfalls & fixes
- **Blank Inspector** → there’s a compile error. Fix missing packages or comment the `#define` flags.
- **No Leap data** → assign the **Leap Provider** and ensure the Ultraleap service is running; hands must be in FOV.
- **No XR data** → ensure the XR Hands subsystem is running (OpenXR project settings, hand tracking enabled).

---

## 11) Adding game/exercise context (optional)
If you want per-row context (user/session/exercise, `standard s`, `trial t`, `actual a`, `tol`, `P`, `c`, success flag), add this inside `WriteRow(...)` where noted:
```csharp
// inside WriteRow(...)
_sb.Append(","context":");
_sb.Append(GetContextJson()); // Implement to pull from TrialManager/PlayerPrefs
```
Keep the context object small and stable (flat JSON) for easier parsing.

---

## 12) Minimal analysis plan
- Parse `rows[*]` in Python/R/Matlab.
- For each time step and joint name, compute:
  - `pos_error_mm = 1000 * ||xr.p - leap.p||`
  - `rot_error_deg = 2 * acos(|dot(qxr, qleap)|) * 180/pi`
- Summarize by mean/median/IQR; plot errors across exercises for insight.
- For latency estimation later, also use Leap’s `leap_ts_us` if you decide to map device time to session time.

---

### That’s it
You’re set to record synchronized XR + Leap poses into one JSON for post-hoc comparison. If you want a **CSV** or **ND‑JSON** variant (better for huge logs) or an **exercise context** block baked in, I can extend the script.
