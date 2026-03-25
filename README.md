# SCTQ: Self-Consistent Tracking Quality

SCTQ is a complete, modular, research-grade Python framework for evaluating multi-object tracking (MOT) quality **without requiring ground-truth annotations**. It achieves this by assessing the internal self-consistency and physical plausibility of the generated tracks.

This project supports:
1. Real-video evaluation using YOLO detections and multiple configurable trackers.
2. Synthetic benchmark generation with controllable noise and perturbations.
3. Automated corruption suites to test tracker robustness.
4. Evaluation of standard MOT datasets.
5. Extensive reporting (CSV, JSON, Markdown) and automated metric plotting.

## 🌟 The SCTQ Metric

SCTQ is composed of four core proxy metrics and a stability aggregator for noisy environments:
*   **Persistence ($S_{pers}$):** Rewards long, sustained tracks.
*   **Dynamic Plausibility ($S_{dyn}$):** Penalizes physically implausible motion (extreme sharp turns, massive sudden accelerations).
*   **Fragmentation Sensitivity ($S_{frag}$):** Detects and penalizes artificially broken tracklets that are spatially, temporally, and angularly "bridgeable."
*   **Intra-track Consistency ($S_{cons}$):** Penalizes tracks with highly erratic bounding box sizes.
*   **Stability ($S_{stab}$):** Measures the robustness of tracking statistics under various corruptions compared to a clean baseline.

---

## 🛠 Installation

### Prerequisites
*   Python 3.10 or higher.
*   A functional C++ compiler (sometimes required for compiling underlying tracker dependencies).
*   (Optional but recommended) A GPU and CUDA toolkit for running Ultralytics YOLO models efficiently.

### Setup Instructions

1.  **Clone the repository:**
    ```bash
    git clone https://github.com/your-username/sctq.git
    cd sctq
    ```

2.  **Create and activate a virtual environment (recommended):**
    ```bash
    python3.10 -m venv venv
    source venv/bin/activate
    ```

3.  **Install dependencies and the package itself:**
    ```bash
    pip install -e .
    ```

    *Note on Tracker Dependencies:*
    The framework is built to be extensible. By default, it supports fast internal robust implementations for classical trackers (SORT, IOUTracker, Centroid, CentroidKF) and uses `ultralytics` for advanced trackers (ByteTrack, BoT-SORT).
    *   Ultralytics YOLO requires `torch` and `torchvision`.
    *   If you are running in `strict_mode` (default), the benchmark will fail safely if a requested Ultralytics tracker cannot be loaded, preserving the integrity of your results.

---

## 🚀 Usage Guide

The framework is driven by a master CLI entry point located at `src/sctq/cli/main.py`.

### 1. Running the Synthetic Benchmark
This mode generates synthetic mathematically-defined trajectories (linear, curved, crossing, stop-and-go), applies noise, runs the configured trackers across multiple seeds and scenes, and evaluates them.

```bash
# Basic run
python -m sctq.cli.main --mode synthetic

# Using a custom configuration file
python -m sctq.cli.main --mode synthetic --config configs/default.yaml
```

### 2. Running Real-Video Evaluation
This mode runs a YOLO detector on an input `.mp4` video, caches the detections, runs the configured trackers, and generates annotated output videos.

```bash
python -m sctq.cli.main --mode real --video data/raw/my_video.mp4
```

### 3. Running the Corruption / Robustness Suite
This mode evaluates tracker robustness by applying various severities of detection-level and image-level noise (e.g., Gaussian jitter, random drop, Gaussian noise) to the video, rerunning detection and tracking across multiple seeds to generate robustness curves.

```bash
python -m sctq.cli.main --mode corruption --video data/raw/my_video.mp4
```

### 4. Running an Ablation Study
Run an ablation study to test different SCTQ weight combinations alongside heuristic baselines (average track length, track count).

```bash
python -m sctq.cli.main --mode ablation
```

### 5. Running MOT-Format Benchmarks
If you already have detections formatted in the standard MOTChallenge format (`.txt` files), you can evaluate trackers on them directly.

```bash
python -m sctq.cli.main --mode benchmark --det_file data/processed/mot_detections.txt
```

### 6. Exporting Markdown Reports
After running an experiment, you can aggregate the resulting JSON summaries into a readable Markdown report.

```bash
python src/sctq/cli/export_report.py \
    --input data/outputs/synthetic/per_tracker/synthetic_summary.json \
    --output_dir data/outputs/synthetic/reports \
    --title "Synthetic Benchmark Tracker Comparison"
```

---

## ⚙️ Configuration

The framework is highly configurable via YAML files located in the `configs/` directory.

*   `default.yaml`: Master configuration mapping to the other files, and execution flags like `strict_mode`.
*   `metrics.yaml`: Tune the mathematical thresholds for SCTQ (e.g., $\alpha_p$, $\tau_{short}$, $\beta_1$).
*   `trackers.yaml`: Define which trackers to run and their internal hyperparameters.
*   `synthetic.yaml`: Configure synthetic scene generation (number of objects, scene size, motion types).
*   `corruptions.yaml`: Enable/disable specific corruptions and set their severities.
*   `real_video.yaml`: Configure the YOLO detector (model size, confidence threshold, target classes).

---

## 📂 Project Output Structure

All outputs are saved to the `data/outputs/` directory by default, structured by the experiment run:

```text
data/outputs/<experiment_name>/
├── per_track/                 # Detailed features and scores for EVERY single track (CSV & JSON)
├── per_run/                   # Aggregated statistics for a single tracker on a single video/scene
├── per_tracker/               # Master summaries ranking all trackers across the whole experiment
├── validation/                # (Synthetic only) Ground truth comparison metrics (e.g. assignment purity)
├── plots/                     # Auto-generated PNGs (Track length histograms, SCTQ component bar charts, Robustness curves)
└── videos/                    # Rendered .mp4 videos with bounding box overlays
```

### Interpreting the Outputs

*   **`sctq_core`**: The main proxy metric score (0.0 to 1.0). Higher is better.
*   **`stability_score`**: How well the tracker maintained its performance under corrupted detections compared to clean detections.
*   **`sctq_final`**: A weighted combination of `sctq_core` and `stability_score`.

---

## 📝 Extending the Framework

**Adding a New Tracker:**
1.  Create a new adapter class inheriting from `BaseTrackerAdapter` in `src/sctq/tracking/`.
2.  Implement the `reset()` and `update(frame_detections)` methods.
3.  Ensure `update()` returns a list of unified `TrackedObject` instances.
4.  Register the new adapter in `TrackerFactory` inside `src/sctq/tracking/tracker_factory.py`.
5.  Add the tracker to `configs/trackers.yaml`.

**Adding a New Metric:**
1.  Implement the logic in `src/sctq/metrics/`.
2.  Update the `SCTQEngine.compute_sctq_core()` method in `src/sctq/metrics/sctq.py` to calculate and aggregate your new feature.

## License
MIT License
