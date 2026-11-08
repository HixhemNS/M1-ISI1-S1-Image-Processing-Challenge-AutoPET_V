# autoPET-V: Interactive Lesion Segmentation in Whole-Body PET/CT
## Iterative Correction Simulation and Evaluation on the DEEP-PSMA Cohort Using nnU-Net

**Benchmark:** [autoPET-V Grand Challenge (MICCAI)](https://autopet-v.grand-challenge.org/)  
**Task:** Category 1 — Simulated Human-in-the-Loop Interaction  
**Dataset:** [DEEP-PSMA](https://zenodo.org/records/15281784), a 20-patient resampled PSMA PET/CT subset (`train_0001`–`train_0020`)  
**Model:** [Official autoPET-V baseline](https://github.com/lab-midas/autoPETV), a pre-trained 3D full-resolution nnU-Net (`Dataset998`)  
**Input channels:** CT, PET, foreground clicks, and background clicks

This project explores whether sparse corrective input can improve a pre-trained model's lesion segmentation in whole-body PET/CT. It reproduces a small, local version of the autoPET-V interactive-segmentation setting: the model first generates a prediction, then receives simulated foreground and background clicks derived from its errors. The project also evaluates two classical post-processing operations.

> **Scope:** This is an educational image-processing project and a scaled-down experiment. It is not a clinical tool, and its results are not directly comparable with the official autoPET-V leaderboard.

## Project goals

- Prepare CT, PET, and reference-mask volumes for the four-channel nnU-Net input format.
- Run the official pre-trained baseline without training or fine-tuning.
- Simulate iterative corrective clicks using the prediction errors and the Euclidean Distance Transform (EDT).
- Measure segmentation quality over five correction rounds.
- Study the effects of 3D hole filling and a small-component volume filter.
- Discuss where simulated interaction helps, where it can make predictions worse, and what limits the conclusions.

## Dataset and evaluation split

The experiment uses 20 PSMA cases from the DEEP-PSMA subset. The cases are divided as follows:

| Partition | Cases | Use in this project |
|---|---|---|
| Development | `train_0001`–`train_0010` | Formatted for the pipeline; not used for training, fine-tuning, or threshold tuning |
| Evaluation | `train_0011`–`train_0020` | Baseline inference, simulated interaction, and reported evaluation |

The evaluation is performed against the supplied Total Tumor Burden (TTB) reference masks. The 1.0 mL component-filter threshold was fixed in advance rather than tuned on the evaluation cases.

## Method overview

The workflow has three main stages.

### 1. Data preparation and spatial alignment

CT and PET volumes do not necessarily have the same voxel dimensions or spatial grid. For each case:

- PET is kept on its original grid.
- CT is resampled onto the PET grid using linear interpolation.
- The CT resampling background value is set to **−1024 HU**.
- Two additional channels are initialized as empty volumes for foreground and background interaction.

The resulting nnU-Net input uses the following channel convention:

| Channel | Content |
|---|---|
| `0000` | CT, resampled to the PET grid |
| `0001` | PET |
| `0002` | Foreground click map |
| `0003` | Background click map |

The official pre-trained `Dataset998` checkpoint (fold 0) is used with test-time augmentation disabled. No model training or fine-tuning is performed.

### 2. Simulated interactive correction

The initial prediction is generated with empty click channels. Then, for up to five rounds, the simulator identifies errors in the current prediction by comparing it with the reference mask:

- **False negatives (FN):** reference-positive voxels missed by the prediction.
- **False positives (FP):** predicted-positive voxels outside the reference mask.

The error regions are split into 26-connected components. The largest false-negative component is selected for a foreground click, and the largest false-positive component is selected for a background click. Within each selected component, the click is placed at the voxel with the maximum Euclidean Distance Transform value. This places the click well inside the component rather than near its boundary.

Each round adds at most one foreground and one background **single-voxel** click. Clicks accumulate across rounds, and nnU-Net is run again after the click maps are updated.

The simulation is intentionally idealized: it uses the reference mask to locate errors. It does not represent a real clinician choosing clicks, and it does not use a quality-based stopping rule. A click is added whenever the corresponding error component exists.

### 3. Post-processing

Two classical 3D operations are evaluated on the predictions:

1. **Hole filling:** fills enclosed holes in a binary mask (`binary_fill_holes`).
2. **Small-component filtering:** removes 26-connected predicted components with a volume below **1.0 mL**.

The study compares raw predictions, hole-filled predictions, and hole-filled predictions followed by the size filter. These operations are applied offline: post-processed masks are not fed back into the next interactive round. Clicks are always generated from the raw prediction at that iteration.

## Evaluation metrics

The evaluation focuses on whole-volume binary-mask overlap and volume-based errors:

- **Dice score:** measures overlap between the predicted and reference masks.
- **AUC-Dice:** summarizes the mean Dice curve from iteration 0 to iteration 5 using the trapezoidal rule, normalized by the five intervals.
- **FPV (lesion-level):** volume of predicted connected components that do not overlap the reference.
- **FNV (lesion-level):** volume of reference connected components that do not overlap the prediction.
- **Voxel-level FP volume:** reported separately from lesion-level FPV.

FPV and FNV are implemented for this project and are not claimed to reproduce the official challenge evaluation script. The Detection Matching Metric (DMM) is not computed.

## Main results

On the 10 evaluation cases (`train_0011`–`train_0020`), the mean Dice increased after simulated correction:

| Result | Value |
|---|---:|
| Mean Dice, iteration 0 | 0.4977 |
| Mean Dice, iteration 5 | 0.5985 |
| Mean Dice improvement | +0.1008 |
| Median Dice, iteration 0 | 0.5912 |
| Median Dice, iteration 5 | 0.6046 |
| Mean-curve AUC-Dice | 0.5704 |

The mean Dice increased at each round, but the improvement was not uniform across cases: **7 of 10 cases improved and 3 deteriorated**. Much of the overall gain came from three cases, including two with very poor baseline predictions.

Post-processing had different effects:

- Hole filling had almost no effect on the mean Dice.
- The 1.0 mL size filter reduced mean Dice by approximately 0.010–0.012 compared with hole filling alone.
- Interactive correction reduced mean lesion-level FNV, while mean lesion-level FPV increased. In other words, the model recovered missed regions but also introduced additional false-positive regions.

These results illustrate why a single overlap score is not enough to describe interactive segmentation behavior.

## Limitations

- The evaluation contains only 10 cases and uses one tracer (PSMA).
- The dataset is a small DEEP-PSMA subset, not the official hidden autoPET-V test set.
- Clicks are generated from reference-mask errors, making the interaction more idealized than real user input.
- The prompts are single voxels rather than realistic scribbles.
- There is no stopping rule, so later clicks can sometimes make a prediction worse.
- The reported AUC-Dice and FPV/FNV values are not directly comparable with the official challenge leaderboard.
- DMM is not included, and the volume-based metrics are project implementations.

## Tools and environment

The notebooks use Python and the following main libraries and tools:

- [nnU-Net v2](https://github.com/MIC-DKFZ/nnUNet)
- [autoPET-V baseline repository](https://github.com/lab-midas/autoPETV)
- [Nibabel](https://nipy.org/nibabel/)
- [SimpleITK](https://simpleitk.org/)
- NumPy
- SciPy
- Matplotlib
- Kaggle notebooks and the supplied dataset

The experiments were run in Kaggle. The data and model-weight setup therefore follows the Kaggle input/working-directory conventions used in the notebooks. Reproducing the experiment requires access to the dataset and the official pre-trained checkpoint.

## Project materials

The project is documented through two notebook reports:

- **Part 1 — Data preparation and smoke test:** loading and inspecting CT/PET/reference volumes, preparing the environment and checkpoint, aligning spatial grids, and exporting the four-channel dataset.
- **Part 2 — Interactive pipeline and evaluation:** baseline inference, spatial diagnostics, simulated click rounds, post-processing, metric computation, and analysis of the results.

The accompanying project summary presents the motivation, method, results, limitations, and references.

## References

1. Isensee, F. et al. “nnU-Net: a self-configuring method for deep learning-based biomedical image segmentation.” *Nature Methods*, 2021. [DOI](https://doi.org/10.1038/s41592-020-01008-z)
2. Gatidis, S. et al. “Results from the autoPET challenge on fully automated lesion segmentation in oncologic PET/CT imaging.” *Nature Machine Intelligence*, 2024. [DOI](https://doi.org/10.1038/s42256-024-00912-9)
3. Wang, H. et al. “SAM-Med3D: A Vision Foundation Model for General-Purpose Segmentation on Volumetric Medical Images.” *IEEE TNNLS*, 2025. [DOI](https://doi.org/10.1109/TNNLS.2025.3586694)
4. Isensee, F. et al. “nnInteractive: Redefining 3D Promptable Segmentation.” arXiv, 2025. [arXiv](https://arxiv.org/abs/2503.08373)
5. [autoPET-V Grand Challenge](https://autopet-v.grand-challenge.org/)
6. [Official autoPET-V repository](https://github.com/lab-midas/autoPETV)
7. [DEEP-PSMA dataset record](https://zenodo.org/records/15281784)
