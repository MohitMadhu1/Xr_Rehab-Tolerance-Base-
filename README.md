# 🎯 Adaptive Tolerance System – Full Technical Walkthrough with Examples

This markdown file provides a comprehensive yet approachable explanation of how the adaptive tolerance algorithm works in your VR rehab application. It walks through the purpose, core formulas, logic, flow, and illustrative examples.

---

## 🧠 Concept Overview

In motor rehab, it's critical to strike the right balance:
- Too **strict**, and the user gets frustrated.
- Too **easy**, and progress stalls.

This system ensures:
- Adaptation after every round.
- Personalization based on performance.
- Never makes it harder than clinician-defined ideal.

---

## ⚙️ Key Terms

| Term           | Meaning |
|----------------|---------|
| **Standard Value** (`s`) | Ideal clinician-set target. |
| **Trial Value** (`t`)    | The relaxed or adaptive threshold used in the current round. |
| **Actual Value** (`a`)   | Best performance achieved by the user this round. |
| **Tolerance** (`tol`)    | The allowable leeway around the trial value for success. |
| **Performance Score** (`P`) | How well the actual matched the trial (0 = poor, 1 = perfect). |
| **Adaptability Coefficient** (`c`) | Patient-specific rate at which tolerance shrinks. |

---

## 📐 Core Formulas

### 1. **Tolerance Update**
```text
newTol = prevTol × (1 – c × P)
```

- This shrinks tolerance gradually when user performs well (P → 1).
- Tolerance never collapses to 0; it has a floor based on target magnitude.

---

### 2. **Trial Value Update**

This follows **exponential moving average (EMA)** behavior with caps:

| Policy             | Behavior |
|--------------------|----------|
| **MinValue**       | Pull trial *up* toward actual (but no lower than ideal). |
| **MaxValue**       | Pull trial *down* toward actual (but no higher than ideal). |
| **ClosestToTarget**| Pull trial toward actual only if closer to ideal than before. |

All updates respect:
- Clamping to not exceed the standard.
- Smoothing using `ALPHA = 0.2`.

---

## 🧪 Performance Score Calculation

```text
P = 1 – |actual – effectiveTarget| / scale
```

Where:
- `effectiveTarget` is the most challenging of trial vs standard.
- `scale` ensures unit-safety and normalization.

> This makes P = 1 when actual = target, and decays toward 0 otherwise.

---

## 🔢 Worked Example

Let’s walk through a user trying to improve their thumb–index distance.

### ✋ Setup

| Parameter     | Value |
|---------------|-------|
| Ideal (`s`)   | 10.0  |
| Trial (`t`)   | 7.0   |
| Actual (`a`)  | 7.5   |
| Previous Tolerance | 3.0 |
| Adaptability Coefficient `c` | 0.25 |

### 📊 Step 1: Compute Performance Score
```text
scale = max(10.0, 7.0, 7.5) = 10.0
P = 1 – |7.5 – 10.0| / 10.0 = 1 – 0.25 = 0.75
```

### 📉 Step 2: Update Tolerance
```text
newTol = 3.0 × (1 – 0.25 × 0.75) = 3.0 × (1 – 0.1875) = 2.4375
```

### 🧮 Step 3: Update Trial Value (MinValue policy)
```text
targetMin = max(7.5, 10.0) = 10.0
newTrial = Lerp(7.0, 10.0, 0.2) = 7.6
```

➡ Result: Tolerance tightens and trial becomes slightly harder.

---

## 🏁 When Is This Run?

The algorithm runs **after every round is completed**, triggered inside each detector script (e.g., `StopGestureDetector`, `TendonGlideDetect`, etc.), or centralized in `TrialManager.cs`.

---

## 🏗️ Scripts Involved

| Script | Role |
|--------|------|
| `ToleranceUpdater.cs` | Contains and applies all formula logic. |
| `TrialManager.cs` | Calls update after final exercise or round. |
| `StopGestureDetector`, `DownGestureDetector`, `MedianNerveGlidePolished`, `TendonGlideDetect` | Send metric values to updater. |
| `DoctorPageController.cs` | Can override `c` value. |
| `PatientInfo.cs` | Provides fallback `c` calculation (based on age, gender, severity, ethnicity). |

---

## 🎚️ Adaptability Coefficient `c`

Derived from:
- **Doctor override** (if present in `PlayerPrefs`)
- Else computed as:
  ```text
  c = base × (age factor) × (gender factor) × (severity factor)
  ```
Typical range: `0.12 – 0.30`

| Factor     | Effect |
|------------|--------|
| Age        | Older age = slower adaptation. |
| Gender     | Slightly dampened for female/non-binary. |
| Severity   | Higher severity = slower progression. |

---

## ⚖️ Metric Weighting

- All metrics (e.g., PalmDot, AvgBend) are evaluated **individually**.
- No hardcoded weights like "60% accuracy, 40% bend".
- Their effect is driven by `BestPolicy` and how close actual is to trial.

---

## ✅ Summary

| Feature             | Behavior |
|---------------------|----------|
| Tolerance Update    | Shrinks when user performs well, stays if poor. |
| Trial Value Update  | Moves toward actual (never harder than ideal). |
| Custom per-patient  | Yes, via `c`. |
| Metric-specific     | Yes, based on gesture and policy. |
| Real-time feedback  | Reflected in next round. |
| Doctor control      | Override `c` via UI. |

---

This system ensures users are always nudged toward improvement, without setting them up to fail.
