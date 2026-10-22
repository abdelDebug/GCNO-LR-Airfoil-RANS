# GCNO-LR for Airfoil RANS Field Prediction

Research implementation for **GCNO-LR**, a geometry-conditioned spectral surrogate with masked Cartesian local feature refinement for predicting sampled two-dimensional airfoil RANS fields.

This repository contains seven research notebooks. Notebooks 01–06 document benchmark preparation, model development, baseline comparisons, and the frozen official-test evaluation. Notebook 07 adds a separate, retrospective development-validation ablation of the solid-path rejection rule. The manuscript itself is intentionally not included in this repository. 

## Contribution overview

GCNO-LR combines a multiscale convolutional encoder-decoder, a Fourier spectral core, operating-condition conditioning, and a full-resolution masked local refinement mechanism. The local refiner samples latent feature differences at two pixel radii, rejects stencil paths that cross the solid mask or leave the sampled domain, applies learned adaptive weights, and performs two shared-weight refinement iterations before the final correction head.

The final selected model is the **`no_alignment`** configuration from the development study. It retains the masked local refinement mechanism but uses fixed Cartesian stencil directions rather than flow-aligned directions.

Under the fixed evaluation protocol reported in the study, the official test contains **90 operating conditions on 30 unseen airfoil identities**, with three trained seeds per architecture. GCNO-LR achieved a mean relative field L1 of **0.021061**, compared with **0.027080** for the prespecified FNO comparator, corresponding to a **22.23% relative reduction**. Mean pressure relative L2 decreased from **0.152762 to 0.118074**, and vector-velocity relative L2 decreased from **0.041274 to 0.031163**.

The gain is accompanied by additional inference cost. On a Tesla T4, measured batch-one median forward latency was **15.008 ms** for GCNO-LR versus **4.174 ms** for FNO under the reported timing protocol.

## Notebook sequence

Run or inspect the notebooks in this order:

| # | Notebook | Purpose |
|---|---|---|
| 01 | `01_dataset_preparation_deep_flow_prediction.ipynb` | Prepares the Deep Flow Prediction benchmark representation, preserves the study normalization conventions, constructs geometry-derived features, and records split and cache provenance. |
| 02 | `02_training_geometry_conditioned_neural_operator_refactored.ipynb` | **Defines the GCNO-LR architecture and the main comparison architectures.** Implements the geometry-conditioned spectral encoder-decoder, FiLM conditioning, masked local stencil refiner, training loop, validation metrics, and initial ablations. |
| 03 | `03_airfoil_validation_and_next_experiments.ipynb` | Audits the completed ablation study, analyzes seed-wise effects, and prepares the next controlled comparisons. |
| 04 | `04_publication_decisive_experiments.ipynb` | Implements the decisive development experiments, including the nearly parameter-matched convolutional-refiner control and diagnostic analyses. |
| 05 | `05_decisive_control_and_baseline_training.ipynb` | Completes the controlled training of the convolutional refiner and the DFP, residual U-Net, and FNO baselines under the fixed budget and validation protocol. |
| 06 | `06_frozen_official_unseen_airfoil_test.ipynb` | Freezes the selected `no_alignment` configuration as GCNO-LR, evaluates the official unseen-airfoil test, performs paired uncertainty analysis, measures inference cost, and generates fixed qualitative examples. |
| 07 | `07_mask_path_rejection_ablation_fixed.ipynb` | Performs a controlled retrospective comparison of solid-path rejection enabled versus disabled in the selected Cartesian-stencil refiner, using matched development data, three training seeds, paired regional metrics, and uncertainty analysis. |

## Architecture provenance

The GCNO-LR architecture is designed in Notebook 02. The central implementation is the geometry-conditioned residual operator together with the local stencil refiner. The model contains:

1. geometry and freestream feature maps plus standardized operating conditions,
2. residual convolutional encoding at multiple spatial resolutions,
3. a 32 x 32 spectral core with four Fourier residual blocks,
4. FiLM conditioning using the operating variables,
5. decoder skip connections and a coarse field prediction,
6. masked local feature refinement with radii of 1 and 3 pixels,
7. eight Cartesian stencil offsets in the final selected configuration,
8. two shared-weight refinement iterations,
9. a learned correction head producing the final scaled velocity and pressure fields.

Notebook 06 does not redesign the architecture. It freezes the selected `no_alignment` configuration and evaluates it using the prespecified official-test plan. Notebook 07 retains this selected architecture and changes only the stencil path-validity rule to isolate its contribution.

## Data and benchmark

The workflow is based on the publicly available **Deep Flow Prediction** airfoil RANS benchmark and software by Thuerey, Weißenow, Prantl, and Hu.

Original project:

https://github.com/thunil/Deep-Flow-Prediction

The notebooks expect the source dataset or its prepared derivative to be available in the paths configured at the beginning of the workflow. Google Colab and Google Drive paths used during the original experiments are retained in the notebooks for traceability and can be changed to local paths when reproducing the study elsewhere.

## Experimental protocol

The reported comparison uses a common training budget of **10,000 optimizer steps** and three seeds: **42, 123, and 2026**. Checkpoints are selected by minimum validation scaled L1. The final architecture and comparator plan were fixed before official-test evaluation.

The primary official comparison is GCNO-LR versus FNO on relative field L1. Pressure and vector-velocity relative L2 are reported as physical-field guardrails. Residual U-Net and DFP/TurbNetG are secondary comparisons.

Paired uncertainty estimates use crossed bootstrap resampling over training seeds and unseen airfoil groups, with operating conditions averaged within each airfoil. This preserves the matched comparison and avoids treating multiple operating conditions from the same airfoil as independent geometries.

## Supplementary solid-path rejection ablation

Notebook 07 studies the **independent contribution of solid-path rejection** to the selected Cartesian-stencil GCNO-LR model. This additional study was performed after the frozen official test had already been evaluated. It is a **retrospective development-validation analysis**, not a replacement for the prespecified official-test comparison or a basis for selecting a new final model.

Two configurations are compared: `masked` rejects sampled feature paths whose bilinear supports intersect the solid airfoil or lie outside the sampled domain; `no_path_rejection` permits paths across the solid but retains the out-of-domain checks. **Both configurations retain the learned local refinement branch, its final fluid-mask update, the same two refinement iterations, and 5,690,558 trainable parameters.** Three seeds (42, 123, 2026) were trained for 10,000 optimizer steps each, giving six matched runs.

The seed-averaged, case-weighted results on the 400 development-validation cases are:

| Metric (lower is better) | Path rejection enabled | Path rejection disabled |
|---|---:|---:|
| Scaled field L1 | 0.003664 | 0.003671 |
| Relative field L1 | 0.018354 | 0.018393 |
| Pressure relative L2 | 0.112548 | 0.113337 |
| Vector-velocity relative L2 | 0.029304 | 0.029388 |

Separate **airfoil-balanced paired estimates** average conditions within each airfoil before averaging across airfoils and seeds. For near-airfoil pressure MAE, the masked configuration reduced the mean from 0.031739 to 0.031414 (approximately 1.02%); the exploratory 95% crossed-bootstrap interval for the *absolute reduction* was **[−0.000400, 0.001100]**, which includes zero. The airfoil-balanced global relative field L1 reduction was approximately 0.17%, also with an interval including zero. The wake-velocity error was slightly worse with path rejection in this comparison. Gradient and divergence *errors relative to the sampled CFD targets* showed small exploratory reductions; these are not demonstrations of physical conservation.

**Interpretation:** The independent accuracy contribution of solid-path rejection is small or inconclusive for the principal prediction metrics. This does **not** contradict the separate development ablation showing that the **complete learned local-refinement branch** is useful. The development split is sample-level and has substantial overlap in airfoil identities between optimization training and validation; these supplementary results do not establish transfer to new geometry identities. No official-test scores were used for reselection.

The notebook defaults to `ACTION="status"` to inspect saved study progress without training. Set `ACTION="train"` only to run or resume the six training configurations, or `ACTION="summarize"` to regenerate reports from completed local study records. The raw benchmark, training checkpoints, and larger derived artifacts are not committed here.

## Reproducibility notes

For repository hygiene, stored execution outputs and execution counters have been cleared from the published notebooks. The code cells, markdown cells, notebook order, and experimental logic are unchanged. Running the notebooks in sequence regenerates the corresponding outputs when the required data and environment are available.

These notebooks preserve the study workflow and recorded experimental logic. Several implementation and experiment identity checks are intentionally embedded in the later notebooks. If exact reproduction of the reported results is required, keep the original prepared split identities, experiment configuration, training budget, seed values, checkpoint-selection rule, and frozen-test plan unchanged.

GPU execution may not be bitwise deterministic because some CUDA grid-sampling backward operations can remain nondeterministic even when deterministic settings are requested.

## Scope and limitations

This repository supports prediction and evaluation of sampled RANS velocity and kinematic-pressure fields on the fixed Cartesian benchmark. The reported results do **not** establish:

- conservation enforcement,
- improved turbulence closure,
- validated lift or drag accuracy,
- wall-shear accuracy,
- mesh-independent surrogate convergence,
- resolution transfer,
- unrestricted generalization to other aerodynamic datasets or flow regimes.

The result should therefore be interpreted as a controlled **accuracy-versus-computational-cost** evaluation of local refinement within the studied benchmark and training budget.

## Authors

- **Abdelaziz Triki**, Berlin School of Business and Innovation
- **Amir Teimourian**, Berlin School of Business and Innovation

## Citation

The associated manuscript is not included in this repository. If you use this implementation in academic work, please cite the corresponding paper once its final bibliographic record is available.
