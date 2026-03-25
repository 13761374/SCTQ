SCTQ: Self-Consistent Tracking QualitySCTQ is a research-grade Python framework for evaluating multi-object tracking (MOT) quality without requiring ground-truth annotations.Instead of comparing tracker outputs against labeled identities, SCTQ evaluates the self-consistency, temporal coherence, physical plausibility, and fragmentation behavior of the generated tracks. This makes it useful in real deployment settings where annotations are unavailable, delayed, or too expensive to obtain.SCTQ supports:Synthetic validation against GT-based metrics such as IDP, IDR, and IDF1Real-video evaluation without annotationsRobustness analysis under image-level and detection-level corruptionsCalibration and ablation studiesResearch-grade reporting with CSV, JSON, Markdown summaries, and plotsOverviewThe goal of SCTQ is to answer a practical question:Can tracking quality be estimated directly from tracker outputs, even when no ground truth is available?SCTQ does this through four interpretable components:Persistence: longer, sustained tracks are betterDynamics: physically plausible motion is betterFragmentation: fewer artificial track splits are betterConsistency: internally stable trajectories are betterThe final score is designed to be interpretable, annotation-free, and validated against standard MOT metrics in controlled synthetic experiments.The SCTQ MetricLet a tracker produce a set of trajectories over a video. SCTQ computes four component scores:1. Persistence (P)Rewards trajectories that remain active over time instead of collapsing into many short tracklets.2. Dynamic Plausibility (D)Penalizes trajectories with unrealistic motion, such as:abrupt turnslarge acceleration spikesunstable motion patterns3. Fragmentation (F)Penalizes cases where one physical object is likely broken into multiple tracklets.This uses a bridge-aware formulation based on:temporal gapspatial extrapolationmotion direction continuity4. Consistency (C)Measures intra-track regularity, such as stable bounding-box behavior over time.Gated ConsistencyTo avoid over-rewarding fragmented but locally smooth tracks, SCTQ uses:$$C_{\mathrm{eff}} = C \sqrt{PF}$$Final Score$$\mathrm{SCTQ} = w_p P + w_d D + w_f F + w_c C_{\mathrm{eff}}$$This gives a final annotation-free tracking quality score in [0,1], where higher is better.What This Repository ProvidesThis repository includes a full experimental framework for:Synthetic benchmark generationcontrolled trajectoriesmultiple scenes and random seedsdirect comparison with GT-based metricsReal-video evaluationYOLO-based detectionsmultiple configurable trackersclip-based benchmarking without ground truthCorruption robustness analysisimage-level noise and blurdetection-level jitter, drops, false positivesdegradation curves and robustness summariesCalibrationsearch over SCTQ component weightscomparison against GT-based metricsAblationremove componentscompare against heuristic baselinesstudy the effect of metric design choicesRepository StructureTypical output structure:data/outputs/
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
InstallationRequirementsPython 3.10+pipvirtualenv recommendedoptional GPU for faster YOLO inferenceSetupgit clone [https://github.com/your-username/sctq.git](https://github.com/your-username/sctq.git)
cd sctq

python -m venv venv
source venv/bin/activate    # On Windows: venv\Scripts\activate

pip install -e .
If you use Ultralytics-based trackers or YOLO detectors, make sure the required dependencies are installed properly.Quick Start1. Run testspytest -q
2. Run the synthetic benchmarkpython -m sctq.cli.run_synthetic --config configs/default.yaml
3. Run calibrationpython -m sctq.cli.run_calibration --config configs/default.yaml
4. Run ablationpython -m sctq.cli.run_ablation --config configs/default.yaml
5. Run clean real-video evaluationpython -m sctq.cli.run_real_article \
  --config configs/default.yaml \
  --dataset-root . \
  --clips-per-video 3 \
  --max-frames 400 \
  --skip-corruptions
6. Run full real-video protocol with corruptionspython -m sctq.cli.run_real_article \
  --config configs/default.yaml \
  --dataset-root . \
  --clips-per-video 3 \
  --max-frames 400
Recommended Full PipelineIf you want to reproduce the main experimental workflow:pytest -q

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
How to Use SCTQ to Evaluate a Tracking MethodThis is the most important practical question.Case 1: You want to evaluate a tracker without ground truthUse SCTQ directly on real videos.WorkflowPrepare one or more videosRun the tracker(s) through the frameworkCompute:P (Persistence)D (Dynamics)F (Fragmentation)C_eff (Gated Consistency)final SCTQCompare trackers by:overall SCTQcomponent breakdownrobustness under corruptionInterpretationHigher SCTQ → better overall tracking qualityHigh Persistence → trajectories remain alive longerHigh Dynamics → motion is more plausibleHigh Fragmentation score → less artificial splittingHigh Consistency → more regular internal behaviorThis mode is useful when:no labels existyou want to compare trackers on deployment videosyou want to monitor tracker quality over timeyou want to study robustness under degraded inputsCase 2: You want to validate SCTQ scientificallyUse the synthetic benchmark.WorkflowGenerate synthetic scenes with exact ground truthRun all trackers on the same detectionsCompute both:SCTQGT-based metrics such as IDP, IDR, IDF1Measure:correlationranking agreementablation behaviorcalibration behaviorThis mode answers:does SCTQ agree with standard MOT metrics?does SCTQ rank strong trackers above weak ones?which metric components matter most?Case 3: You want to test robustnessUse the corruption suite.WorkflowRun a clean baselineApply corruption types and severity levelsRe-run trackingMeasure degradation in SCTQ and its componentsTypical corruption types:Gaussian noisesalt-and-pepper noiseblurGaussian center jitterrandom dropfalse positivesThis allows you to compare:clean qualityrobustness qualitycomponent-specific failure modesHow to Read the Main Output FilesSynthetic benchmarkdata/outputs/synthetic/all_runs_flat.csvPer-run results for all trackers and synthetic sequencesdata/outputs/synthetic/correlations.jsonCorrelation between SCTQ and GT-based metricsdata/outputs/synthetic/per_tracker/Tracker-level summaries and rankingsdata/outputs/synthetic/validation/GT-based validation metrics such as IDP, IDR, IDF1Calibrationdata/outputs/calibration/calibration_results.csvAll tested weight configurations and their performancedata/outputs/calibration/best_weights.jsonSelected final SCTQ weightsdata/outputs/calibration/highest_pearson_weights.jsonBest Pearson-correlation configurationAblationdata/outputs/ablation_study/ablation_results.csvResults for Full SCTQ and reduced variantsdata/outputs/ablation_study/ablation_pivot.csvPivot summary for easy comparisondata/outputs/ablation_study/heuristic_baselines.csvComparison with simple baselines such as track lengthReal clean benchmarkdata/outputs/real_article/clean_all_runs_flat.csvAll clean real-video runsdata/outputs/real_article/clean_by_video.csvSummary by video / scene typeRobustness benchmarkdata/outputs/real_article/corruption_all_runs_flat.csvAll corrupted runsdata/outputs/real_article/robustness_by_corruption.csvMean degradation by corruption typedata/outputs/real_article/robustness_by_severity.csvDegradation across severity levelsdata/outputs/real_article/robustness_by_video.csvRobustness summary by videodata/outputs/real_article/robustness_overall.csvFinal tracker robustness comparisonPractical Evaluation RecipeIf you are testing a new tracking method, the recommended workflow is:A. For scientific comparisonRun synthetic validation:python -m sctq.cli.run_synthetic --config configs/default.yaml
Then inspect:synthetic/correlations.jsonsynthetic/all_runs_flat.csvsynthetic/per_tracker/synthetic/validation/B. For annotation-free real evaluationRun clean real benchmark:python -m sctq.cli.run_real_article \
  --config configs/default.yaml \
  --dataset-root . \
  --clips-per-video 3 \
  --max-frames 400 \
  --skip-corruptions
Then inspect:real_article/clean_all_runs_flat.csvreal_article/clean_by_video.csvC. For robustness evaluationRun full real benchmark with corruptions:python -m sctq.cli.run_real_article \
  --config configs/default.yaml \
  --dataset-root . \
  --clips-per-video 3 \
  --max-frames 400
Then inspect:real_article/robustness_overall.csvreal_article/robustness_by_corruption.csvreal_article/robustness_by_severity.csvAdding a New TrackerTo integrate a new tracker:Create a new adapter in src/sctq/tracking/Inherit from BaseTrackerAdapterImplement:reset()update(frame_detections)Return unified tracked objectsRegister the tracker in TrackerFactoryAdd configuration in configs/trackers.yamlAdding a New Metric ComponentTo add a new metric:Implement the metric logic in src/sctq/metrics/Update the SCTQ aggregation code in:src/sctq/metrics/sctq.pyAdd configuration parameters to configs/metrics.yamlUpdate reports and plots if neededReproducibilityThe framework is designed to produce article-grade outputs:configuration snapshotsflat CSV summariesJSON summariesMarkdown reportsplots for ranking, correlation, and robustnessTypical snapshots include:config_snapshot.jsonprotocol_snapshot.jsonThese make it possible to trace each result back to the exact experimental configuration.LicenseMIT License
