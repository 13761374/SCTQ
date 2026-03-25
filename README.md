````markdown
# SCTQ: Self-Consistent Tracking Quality

**SCTQ** is a research-grade Python framework for evaluating **multi-object tracking (MOT)** quality **without requiring ground-truth annotations**.

Instead of comparing tracker outputs against labeled identities, SCTQ evaluates the **self-consistency, temporal coherence, physical plausibility, and fragmentation behavior** of the generated tracks. This makes it useful in real deployment settings where annotations are unavailable, delayed, or too expensive to obtain.

SCTQ supports:

- **Synthetic validation** against GT-based metrics such as IDP, IDR, and IDF1
- **Real-video evaluation** without annotations
- **Robustness analysis** under image-level and detection-level corruptions
- **Calibration and ablation studies**
- **Research-grade reporting** with CSV, JSON, Markdown summaries, and plots

---

## Overview

The goal of SCTQ is to answer a practical question:

> Can tracking quality be estimated directly from tracker outputs, even when no ground truth is available?

SCTQ does this through four interpretable components:

- **Persistence**: longer, sustained tracks are better
- **Dynamics**: physically plausible motion is better
- **Fragmentation**: fewer artificial track splits are better
- **Consistency**: internally stable trajectories are better

The final score is designed to be **interpretable**, **annotation-free**, and **validated against standard MOT metrics** in controlled synthetic experiments.

---

## The SCTQ Metric

Let a tracker produce a set of trajectories over a video. SCTQ computes four component scores:

### 1. Persistence (`P`)
Rewards trajectories that remain active over time instead of collapsing into many short tracklets.

### 2. Dynamic Plausibility (`D`)
Penalizes trajectories with unrealistic motion, such as:
- abrupt turns
- large acceleration spikes
- unstable motion patterns

### 3. Fragmentation (`F`)
Penalizes cases where one physical object is likely broken into multiple tracklets.  
This uses a **bridge-aware formulation** based on:
- temporal gap
- spatial extrapolation
- motion direction continuity

### 4. Consistency (`C`)
Measures intra-track regularity, such as stable bounding-box behavior over time.

### Gated Consistency
To avoid over-rewarding fragmented but locally smooth tracks, SCTQ uses:

$$
C_{\mathrm{eff}} = C \sqrt{PF}
$$

### Final Score

$$
\mathrm{SCTQ} = w_p P + w_d D + w_f F + w_c C_{\mathrm{eff}}
$$

This gives a final annotation-free tracking quality score in `[0,1]`, where **higher is better**.

---

## What This Repository Provides

This repository includes a full experimental framework for:

1. **Synthetic benchmark generation**

   * controlled trajectories
   * multiple scenes and random seeds
   * direct comparison with GT-based metrics

2. **Real-video evaluation**

   * YOLO-based detections
   * multiple configurable trackers
   * clip-based benchmarking without ground truth

3. **Corruption robustness analysis**

   * image-level noise and blur
   * detection-level jitter, drops, false positives
   * degradation curves and robustness summaries

4. **Calibration**

   * search over SCTQ component weights
   * comparison against GT-based metrics

5. **Ablation**

   * remove components
   * compare against heuristic baselines
   * study the effect of metric design choices

---

## Repository Structure

Typical output structure:

```text
data/outputs/
├── real_article/
│   ├── plots/
│   ├── reports/
│   ├── protocol_snapshot.json
│   ├── corruption_all_runs_flat.csv
│   ├── robustness_by_corruption.csv
│   ├── robustness_by_severity.csv
│   ├── robustness_by_video.csv
│   ├── robustness_overall.csv
│   ├── clean_all_runs_flat.csv
│   ├── clean_by_video.csv
│   ├── per_tracker/
│   ├── per_video/
│   ├── per_run/
│   ├── per_track/
│   └── validation/
│
├── ablation_study/
│   ├── plots/
│   ├── reports/
│   ├── ablation_pivot.csv
│   ├── ablation_results.csv
│   ├── baseline_summary.json
│   ├── heuristic_baselines.csv
│   ├── per_tracker/
│   ├── per_run/
│   ├── per_track/
│   └── validation/
│
├── calibration/
│   ├── best_weights.json
│   ├── calibration_results.csv
│   ├── highest_pearson_weights.json
│   └── reports/
│
└── synthetic/
    ├── config_snapshot.json
    ├── plots/
    ├── reports/
    ├── all_runs_flat.csv
    ├── correlations.json
    ├── per_run/
    ├── per_track/
    ├── per_tracker/
    └── validation/
```

---

## Installation

### Requirements

* Python 3.10+
* pip
* virtualenv recommended
* optional GPU for faster YOLO inference

### Setup

```bash
git clone https://github.com/your-username/sctq.git
cd sctq

python -m venv venv
source venv/bin/activate    # On Windows: venv\Scripts\activate

pip install -e .
```

If you use Ultralytics-based trackers or YOLO detectors, make sure the required dependencies are installed properly.

---

## Quick Start

### 1. Run tests

```bash
pytest -q
```

### 2. Run the synthetic benchmark

```bash
python -m sctq.cli.run_synthetic --config configs/default.yaml
```

### 3. Run calibration

```bash
python -m sctq.cli.run_calibration --config configs/default.yaml
```

### 4. Run ablation

```bash
python -m sctq.cli.run_ablation --config configs/default.yaml
```

### 5. Run clean real-video evaluation

```bash
python -m sctq.cli.run_real_article \
  --config configs/default.yaml \
  --dataset-root . \
  --clips-per-video 3 \
  --max-frames 400 \
  --skip-corruptions
```

### 6. Run full real-video protocol with corruptions

```bash
python -m sctq.cli.run_real_article \
  --config configs/default.yaml \
  --dataset-root . \
  --clips-per-video 3 \
  --max-frames 400
```

---

## Recommended Full Pipeline

If you want to reproduce the main experimental workflow:

```bash
pytest -q

python -m sctq.cli.run_synthetic --config configs/default.yaml
python -m sctq.cli.run_calibration --config configs/default.yaml
python -m sctq.cli.run_ablation --config configs/default.yaml

python -m sctq.cli.run_real_article \
  --config configs/default.yaml \
  --dataset-root . \
  --clips-per-video 3 \
  --max-frames 400 \
  --skip-corruptions

python -m sctq.cli.run_real_article \
  --config configs/default.yaml \
  --dataset-root . \
  --clips-per-video 3 \
  --max-frames 400
```

---

## How to Use SCTQ to Evaluate a Tracking Method

This is the most important practical question.

### Case 1: You want to evaluate a tracker **without ground truth**

Use SCTQ directly on real videos.

#### Workflow

1. Prepare one or more videos
2. Run the tracker(s) through the framework
3. Compute:

   * `P` (Persistence)
   * `D` (Dynamics)
   * `F` (Fragmentation)
   * `C_eff` (Gated Consistency)
   * final `SCTQ`
4. Compare trackers by:

   * **overall SCTQ**
   * **component breakdown**
   * **robustness under corruption**

#### Interpretation

* **Higher SCTQ** → better overall tracking quality
* **High Persistence** → trajectories remain alive longer
* **High Dynamics** → motion is more plausible
* **High Fragmentation score** → less artificial splitting
* **High Consistency** → more regular internal behavior

This mode is useful when:

* no labels exist
* you want to compare trackers on deployment videos
* you want to monitor tracker quality over time
* you want to study robustness under degraded inputs

---

### Case 2: You want to validate SCTQ scientifically

Use the **synthetic benchmark**.

#### Workflow

1. Generate synthetic scenes with exact ground truth
2. Run all trackers on the same detections
3. Compute both:

   * **SCTQ**
   * **GT-based metrics** such as IDP, IDR, IDF1
4. Measure:

   * correlation
   * ranking agreement
   * ablation behavior
   * calibration behavior

This mode answers:

* does SCTQ agree with standard MOT metrics?
* does SCTQ rank strong trackers above weak ones?
* which metric components matter most?

---

### Case 3: You want to test robustness

Use the **corruption suite**.

#### Workflow

1. Run a clean baseline
2. Apply corruption types and severity levels
3. Re-run tracking
4. Measure degradation in SCTQ and its components

Typical corruption types:

* Gaussian noise
* salt-and-pepper noise
* blur
* Gaussian center jitter
* random drop
* false positives

This allows you to compare:

* clean quality
* robustness quality
* component-specific failure modes

---

## How to Read the Main Output Files

### Synthetic benchmark

* `data/outputs/synthetic/all_runs_flat.csv`
  Per-run results for all trackers and synthetic sequences

* `data/outputs/synthetic/correlations.json`
  Correlation between SCTQ and GT-based metrics

* `data/outputs/synthetic/per_tracker/`
  Tracker-level summaries and rankings

* `data/outputs/synthetic/validation/`
  GT-based validation metrics such as IDP, IDR, IDF1

---

### Calibration

* `data/outputs/calibration/calibration_results.csv`
  All tested weight configurations and their performance

* `data/outputs/calibration/best_weights.json`
  Selected final SCTQ weights

* `data/outputs/calibration/highest_pearson_weights.json`
  Best Pearson-correlation configuration

---

### Ablation

* `data/outputs/ablation_study/ablation_results.csv`
  Results for Full SCTQ and reduced variants

* `data/outputs/ablation_study/ablation_pivot.csv`
  Pivot summary for easy comparison

* `data/outputs/ablation_study/heuristic_baselines.csv`
  Comparison with simple baselines such as track length

---

### Real clean benchmark

* `data/outputs/real_article/clean_all_runs_flat.csv`
  All clean real-video runs

* `data/outputs/real_article/clean_by_video.csv`
  Summary by video / scene type

---

### Robustness benchmark

* `data/outputs/real_article/corruption_all_runs_flat.csv`
  All corrupted runs

* `data/outputs/real_article/robustness_by_corruption.csv`
  Mean degradation by corruption type

* `data/outputs/real_article/robustness_by_severity.csv`
  Degradation across severity levels

* `data/outputs/real_article/robustness_by_video.csv`
  Robustness summary by video

* `data/outputs/real_article/robustness_overall.csv`
  Final tracker robustness comparison

---

## Practical Evaluation Recipe

If you are testing a new tracking method, the recommended workflow is:

### A. For scientific comparison

Run synthetic validation:

```bash
python -m sctq.cli.run_synthetic --config configs/default.yaml
```

Then inspect:

* `synthetic/correlations.json`
* `synthetic/all_runs_flat.csv`
* `synthetic/per_tracker/`
* `synthetic/validation/`

### B. For annotation-free real evaluation

Run clean real benchmark:

```bash
python -m sctq.cli.run_real_article \
  --config configs/default.yaml \
  --dataset-root . \
  --clips-per-video 3 \
  --max-frames 400 \
  --skip-corruptions
```

Then inspect:

* `real_article/clean_all_runs_flat.csv`
* `real_article/clean_by_video.csv`

### C. For robustness evaluation

Run full real benchmark with corruptions:

```bash
python -m sctq.cli.run_real_article \
  --config configs/default.yaml \
  --dataset-root . \
  --clips-per-video 3 \
  --max-frames 400
```

Then inspect:

* `real_article/robustness_overall.csv`
* `real_article/robustness_by_corruption.csv`
* `real_article/robustness_by_severity.csv`

---

## Adding a New Tracker

To integrate a new tracker:

1. Create a new adapter in `src/sctq/tracking/`
2. Inherit from `BaseTrackerAdapter`
3. Implement:

   * `reset()`
   * `update(frame_detections)`
4. Return unified tracked objects
5. Register the tracker in `TrackerFactory`
6. Add configuration in `configs/trackers.yaml`

---

## Adding a New Metric Component

To add a new metric:

1. Implement the metric logic in `src/sctq/metrics/`
2. Update the SCTQ aggregation code in:

   * `src/sctq/metrics/sctq.py`
3. Add configuration parameters to `configs/metrics.yaml`
4. Update reports and plots if needed

---

## Reproducibility

The framework is designed to produce article-grade outputs:

* configuration snapshots
* flat CSV summaries
* JSON summaries
* Markdown reports
* plots for ranking, correlation, and robustness

Typical snapshots include:

* `config_snapshot.json`
* `protocol_snapshot.json`

These make it possible to trace each result back to the exact experimental configuration.

---

## License

MIT License

````

---

A few notes so you use it smoothly:

- Replace `your-username` in the clone URL.
- If your repo has a different CLI entrypoint naming style, I can adjust those command lines exactly.
- If you want, I can also give you a **short GitHub repo description** for the box at the top of GitHub, like this:

```text
Annotation-free multi-object tracking evaluation via self-consistency, synthetic validation, and robustness analysis under corruption.
````
