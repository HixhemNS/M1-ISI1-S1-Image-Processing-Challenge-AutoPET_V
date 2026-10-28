# autoPET-V: Interactive Lesion Segmentation in Whole-Body PET/CT
## Iterative Correction Simulation & Evaluation on the DEEP-PSMA Cohort Using nnU-Net

**Benchmark & Task:** [autoPET-V Grand Challenge (MICCAI)](https://autopet-v.grand-challenge.org/) — *Category 1: Simulated Human-in-the-Loop Interaction*  
**Dataset:** [DEEP-PSMA (Zenodo: 15281784)](https://zenodo.org/records/15281784) — *20-Patient Resampled Subset (`train_0001`–`train_0020`), PSMA Tracer & Total Tumor Burden (TTB)*  
**Model & Pipeline:** [Official autoPET-V 3D Full-Res nnU-Net Baseline (`Dataset998`)](https://github.com/lab-midas/autoPETV) — *4-Channel Multi-modal Input (CT, PET, FG, BG)*  
**Evaluation Protocol:** *5-Iteration Simulated EDT Single-Voxel Clicks, AUC-Dice, FPV/FNV Error Analysis, and Morphological Post-Processing Ablation*
