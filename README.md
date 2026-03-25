# SCTQ: Self-Consistent Tracking Quality

SCTQ is a research-grade Python framework for evaluating **multi-object tracking (MOT)** quality **without requiring ground-truth annotations**.

Instead of comparing tracker outputs against labeled identities, SCTQ evaluates the **self-consistency, temporal coherence, physical plausibility, and fragmentation behavior** of trajectories.

This makes it suitable for real-world scenarios where annotations are unavailable or expensive.

---

## 🌟 Key Features

- Annotation-free tracking evaluation
- Synthetic validation against IDF1 / IDP / IDR
- Real-video evaluation (no GT required)
- Robustness analysis under corruption
- Calibration and ablation studies
- CSV / JSON / plots for research reporting

---

## 🧠 Core Idea

> A good tracker produces **self-consistent trajectories**.  
> A bad tracker breaks this consistency in observable ways.

SCTQ measures this through interpretable components.

---

## 📐 The SCTQ Metric

SCTQ consists of four components:

### 1. Persistence (P)
Longer tracks are better.  
Short fragmented tracks indicate instability.

---

### 2. Dynamics (D)
Measures physical plausibility of motion:
- smooth trajectories → good
- sudden jumps → penalized

---

### 3. Fragmentation (F)
Penalizes splitting one object into multiple tracklets.

Uses:
- temporal gap
- spatial extrapolation
- direction consistency

---

### 4. Consistency (C)
Measures internal stability:
- bbox size variation
- confidence variation

---

### ⚠️ Gated Consistency

To avoid rewarding fragmented trackers:

$$
C_{\mathrm{eff}} = C \sqrt{P \cdot F}
$$

---

### 🧮 Final Score

$$
\mathrm{SCTQ} = w_p P + w_d D + w_f F + w_c C_{\mathrm{eff}}
$$

Range: `[0, 1]` (higher is better)

---

## 🧪 What This Repo Provides

### 1. Synthetic Benchmark
- 20 scenes × multiple seeds
- controlled trajectories
- GT available
- validates SCTQ vs IDF1 / IDP / IDR

---

### 2. Real Video Evaluation (No GT)
- YOLO detections
- multiple trackers
- clip-based evaluation
- produces ranking without annotations

---

### 3. Corruption / Robustness
- image-level noise (Gaussian, blur)
- detection-level noise (jitter, drop, false positives)
- degradation curves
- robustness ranking

---

### 4. Calibration
- weight search
- correlation with GT metrics

---

### 5. Ablation
- remove components
- compare variants
- analyze metric behavior

---

## 📂 Output Structure

```text
data/outputs/
├── synthetic/
├── calibration/
├── ablation_study/
└── real_article/
```

Important files:

### Synthetic
- `all_runs_flat.csv`
- `correlations.json`
- `validation/`

### Calibration
- `best_weights.json`
- `calibration_results.csv`

### Ablation
- `ablation_results.csv`
- `heuristic_baselines.csv`

### Real Evaluation
- `clean_all_runs_flat.csv`
- `robustness_overall.csv`
- `robustness_by_corruption.csv`

---

## ⚙️ Installation

```bash
git clone https://github.com/your-username/sctq.git
cd sctq

python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate

pip install -e .
```

---

## 🚀 Usage

### Run tests

```bash
pytest -q
```

---

### Synthetic Benchmark

```bash
python -m sctq.cli.run_synthetic --config configs/default.yaml
```

---

### Calibration

```bash
python -m sctq.cli.run_calibration --config configs/default.yaml
```

---

### Ablation

```bash
python -m sctq.cli.run_ablation --config configs/default.yaml
```

---

### Clean Real Evaluation

```bash
python -m sctq.cli.run_real_article \
  --config configs/default.yaml \
  --dataset-root . \
  --clips-per-video 3 \
  --max-frames 400 \
  --skip-corruptions
```

---

### Full Real Evaluation (with corruption)

```bash
python -m sctq.cli.run_real_article \
  --config configs/default.yaml \
  --dataset-root . \
  --clips-per-video 3 \
  --max-frames 400
```

---

## 📊 How to Evaluate a Tracker Using SCTQ

### Step 1: Run evaluation
Run synthetic or real benchmark.

---

### Step 2: Check results

Main outputs:

- `sctq_core` → quality score
- `sctq_final` → final score
- `stability_score` → robustness

---

### Step 3: Compare trackers

Higher SCTQ = better tracking quality

Look at:

- overall SCTQ
- component scores (P, D, F, C)
- robustness degradation

---

### Interpretation

| Component | Meaning |
|----------|--------|
| P | track length |
| D | motion quality |
| F | fragmentation |
| C | internal stability |

---

## 🧪 Scientific Validation

SCTQ is validated by:

- strong correlation with IDF1 (synthetic)
- correct ranking of trackers
- robustness under corruption
- meaningful ablation behavior

---

## ➕ Adding a Tracker

1. Create adapter in `src/sctq/tracking/`
2. Implement:
   - `reset()`
   - `update()`
3. Register in `TrackerFactory`
4. Add to `configs/trackers.yaml`

---

## ➕ Adding a Metric

1. Implement in `src/sctq/metrics/`
2. Update aggregation in `sctq.py`
3. Add config parameters

---

## 📄 License

MIT License
