# 🎯 Adaptive Tolerance System – Full Technical Walkthrough (Code-Accurate)

This document precisely reflects the formulas and logic implemented in the current codebase (`ToleranceUpdater.cs`, `TrialManager.cs`, detector scripts, and `DoctorPageController.cs`). It corrects earlier drafts so it is **100% aligned with what actually runs**.

---

## 🧠 Concept Overview

Motor rehab needs a careful balance:
- Too **strict** → frustration.
- Too **easy** → no progress.

This system:
- Adapts **after every round** (post‑attempt aggregation).
- Personalizes using a **per‑patient adaptability** \(c\) (doctor‑overrideable).
- **Never exceeds** the clinician’s ideal target (policy‑aware clamping).
- Keeps a **safety floor** so tolerance never collapses to zero.

> Paper equivalence note: the paper defines a tolerance \(\tau\) and a threshold \(T = 1-\tau\). The shipped code updates **tolerance** directly; the mapping to a threshold UI is cosmetic.

---

## ⚙️ Key Terms (exact runtime meaning)

| Term | Meaning (runtime) |
|---|---|
| **Standard** `s` | Clinician-set ideal target for the metric (from the step/metric standard). |
| **Trial** `t` | The *current* adaptive threshold value used to judge success this round (persisted in PlayerPrefs). |
| **Actual** `a` | Best value achieved in this round (the detectors write this). |
| **Tolerance** `tol` | Allowed deviation band stored per `(user, step, metric)`; updated every round. |
| **Policy** | One of `MinValue`, `MaxValue`, `ClosestToTarget` controlling directionality/goal. |
| **Adaptability** `c` | Patient-specific coefficient in `[0.12, 0.30]` (doctor override or computed). |
| **Scale** | Normalizer: `scale = max(|effectiveTarget|, |s|, |t|, 0.001)` (see below). |

---

## 📐 Core Formulas (as coded)

### 1) Effective target (policy-aware)
```text
if policy == MinValue:   effectiveTarget = max(t, s)
elif policy == MaxValue: effectiveTarget = min(t, s)
else:                    effectiveTarget = (|t - s| > 1e-4) ? t : s
```

### 2) Performance score P (normalized and clamped)
```text
scale = max(|effectiveTarget|, |s|, |t|, 0.001)
P = clamp(1 - 2 * |a - s| / scale, -1, 1)      # note the factor 2 and reference to STANDARD s
```
- `P = 1` when `a == s`
- `P = 0` when `|a - s| = scale/2`
- `P = -1` when very far from the standard (after clamping)

> There is also an internal `rawP = 1 - |a - s| / scale` that is **not** used for updates; the update uses the **clamped** `P` above.

### 3) Tolerance update (paper-consistent form)
```text
newTol = prevTol * (1 - (1 - c) * P)           # not prevTol * (1 - c * P)
newTol >= tolFloor = 0.01 * max(|effectiveTarget|, 1.0)
```
- Positive `P` (performed near standard) → tolerance **shrinks**.
- Negative `P` (performed far from standard) → tolerance **expands** mildly.
- `c` moderates the effect; higher `c` = safer/slower change.
- A **floor** prevents collapse and scales with problem magnitude.

### 4) Trial update (EMA + policy + clamps)
With `ALPHA = 0.20`:
```text
if policy == MinValue:
    targetMin = max(a, s)
    if targetMin > t:    newT = Lerp(t, targetMin, 0.20)        # make harder
    else:                newT = Lerp(t, max(s, t * 0.985), 0.10) # relax 1.5% toward s
    newT = max(newT, s)

elif policy == MaxValue:
    targetMax = min(a, s)
    if targetMax < t:    newT = Lerp(t, targetMax, 0.20)        # make harder (downward)
    else:                newT = Lerp(t, min(s, t * 1.015), 0.10) # relax 1.5% toward s
    newT = min(newT, s)

else: # ClosestToTarget
    prevDist = |t - s|; newDist = |a - s|
    if newDist < prevDist: newT = Lerp(t, a, 0.20)               # only if a is closer
    if |a - s| < 1e-4:   newT = s                                # snap if perfect
```
All NaN/Inf are sanitized back to `s`.

---

## 🧪 Worked Example (code-accurate)

Let’s assume **MinValue** policy (higher is better).

**Setup**  
` s = 10.0,  t = 7.0,  a = 7.5,  prevTol = 3.0,  c = 0.25`

**Step 1 — effective target & scale**  
`effectiveTarget = max(t, s) = 10.0`  
`scale = max(|10|, |10|, |7|, 0.001) = 10.0`

**Step 2 — P**  
`P = clamp(1 - 2*|7.5 - 10|/10, -1, 1) = clamp(1 - 0.5, -1, 1) = 0.5`

**Step 3 — tolerance**  
`newTol = 3.0 * (1 - (1 - 0.25)*0.5) = 3.0 * (1 - 0.375) = 1.875`  
`toward floor?` `tolFloor = 0.01 * max(|10|, 1) = 0.1` → `1.875` is OK

**Step 4 — trial (MinValue branch)**  
`targetMin = max(a, s) = 10.0` > `t (=7.0)` → make harder  
`newT = Lerp(7.0, 10.0, 0.20) = 7.6`  
`clamp to ≥ s?` after Lerp it’s ≤ s, so stays `7.6`

**Result**: tolerance **tightens** to `1.875`, trial **nudges up** to `7.6`.

---

## 🏁 When does it run?

- Called **after each round** completes for a step/metric.
- Detectors (e.g., `StopGestureDetector`, `DownGestureDetector`, `MedianNerveGlidePolished`, `TendonGlideDetect`) record `a` and invoke the updater.
- `TrialManager` also mirrors the updated trial into UI-facing keys for the clinician dashboard/export.

---

## 🏗️ Scripts & Responsibilities

| Script | What it does |
|---|---|
| `ToleranceUpdater.cs` | Implements the formulas in this document and persists `tol`, `trial`, and `actual` per `(user, step, metric)`; enforces floor & policy rules. |
| `TrialManager.cs` | Orchestrates rounds, surfaces metrics to UI, mirrors trial values to display-friendly keys. |
| Detectors (`StopGestureDetector`, `DownGestureDetector`, `MedianNerveGlidePolished`, `TendonGlideDetect`) | Capture per‑round best `a` and call the updater. |
| `DoctorPageController.cs` | Lets clinician preview/set an override `c` and saves it to `PlayerPrefs` as `{user}_c_value`. |
| `PatientInfo` | Provides fallback for `c` (age, gender, ethnicity, severity) if no doctor override exists. |

---

## 🎚️ Adaptability coefficient `c` (exact behavior)

Source of truth:
1) **Doctor override**: `PlayerPrefs["{user}_c_value"]` → clamped to `[0.12, 0.30]`  
2) **Else fallback**: `c = 0.21 * ageFactor * genderFactor * severityFactor`, then clamped to `[0.12, 0.30]`  
   - `ageFactor`: linear from `1.00` (age 20) → `0.95` (age 80)  
   - `genderFactor`: `0.98` for Female/Non‑binary, else `1.00`  
   - `severityFactor`: linear `1.00` (0) → `0.85` (1.0)

Interpretation: higher severity/age → **slower** change; doctor can always override.

---

## ⚖️ Metric handling

- Each metric is updated **independently** per step.
- No hardcoded cross‑metric weights. The “influence” is through policy and the proximity of `a` to `s` (via `P` and trial EMA).

---

## ✅ Quick Summary (truth table)

| Mechanism | What happens |
|---|---|
| Good round (P > 0) | `tol` shrinks; `t` moves toward a stricter target per policy. |
| Mediocre (P ≈ 0) | `tol` barely changes; `t` stabilizes. |
| Poor (P < 0) | `tol` expands a bit; `t` relaxes gently toward `s` with 1.5% easing. |
| Safety | `tol ≥ 1%` of target magnitude; `t` never overshoots `s`. |
| Personalization | `c` set by doctor or computed from demographics + severity. |

---

This version is aligned with the **exact implementation** in your repository as of today.
