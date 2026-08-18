# Hybrid Biomedical Image-Analysis Pipeline — Stained-Nuclei Segmentation

Assignment 3 submission: a hybrid pipeline combining a local multimodal LLM (`llama3.2-vision`), a
classical image-processing pipeline (Otsu + `regionprops`) interpreted by a local text LLM
(`llama3.2`), and a trained U-Net, run on a synthetic stained-nuclei fluorescence-microscopy
dataset.

```
raw image → segmentation → quantitative region features → structured JSON record → short narrative
```

**Educational use only.** None of the models here are validated for clinical use.

## Repository structure

```
.
├── README.md
├── .gitignore
├── Notebook/
│   └── Hybrid_Nuclei_Pipeline.ipynb   # executed notebook, all 4 tasks + robustness extension
├── Output/
│   ├── Figures/
│   │   ├── fig_eda_sample_grid.png    # Task 1 EDA
│   │   ├── fig_unet_curves.png        # Task 3 loss/Dice curves
│   │   ├── fig_unet_panels.png        # Task 3 input/GT/prediction panels
│   │   ├── fig_otsu_vs_unet.png       # Otsu vs U-Net mask comparison
│   │   └── fig_robustness_trace.png   # extension: corruption trace
│   ├── hybrid_pipeline_report.csv     # Task 4 aggregated JSON records (12 test images)
│   └── unet_val_metrics.csv           # per-image Dice/IoU on the 20-image validation split
└── Report/
    └── Report.pdf   # 4-page report (Task 5)
```

## Dataset

[`Nickolay-K/Assingnment-3-dataset`](https://github.com/Nickolay-K/Assingnment-3-dataset) —
synthetic stained-nuclei fluorescence microscopy, 256×256 RGB, with binary masks, 16-bit instance
labels, and a `metadata.csv` of ground-truth object counts and density regimes (`sparse`, `normal`,
`dense`, `clustered`). Not committed to this repo — the notebook clones it fresh on every run
(Setup cell 4).

## How to run

Open `Notebook/Hybrid_Nuclei_Pipeline.ipynb` in **Google Colab** and run all cells top to bottom on
a fresh runtime.

**Important — Ollama version pin.** The setup cell installs Ollama pinned to `OLLAMA_VERSION=0.22.1`
rather than "latest". As of Ollama v0.30.0, the `mllama` architecture that `llama3.2-vision` needs
was dropped from Ollama's new inference engine and has not been restored — a confirmed, still-open
upstream bug ([ollama/ollama#16547](https://github.com/ollama/ollama/issues/16547),
[#16490](https://github.com/ollama/ollama/issues/16490)) that surfaces as
`error loading model: unknown model architecture: 'mllama'`. Every release before 0.30.0 loads
`llama3.2-vision` correctly, so the install is pinned to a confirmed-working version. If a future
Ollama release restores `mllama` support, this pin can be removed.

The three Colab setup cells install Ollama, start the server, and pull `llama3.2-vision` (~7.8GB)
and `llama3.2` (~2GB) — this takes a few minutes on first run.

## Results summary

| Metric | Value |
|---|---|
| U-Net validation mean Dice (n=20) | 0.942 |
| U-Net validation mean IoU (n=20) | 0.891 |
| Otsu on train_004 (dense, gt n=85) | 45 connected components (undercounts — touching nuclei merge) |
| Ground-truth mask on train_005 (clustered, gt n=52) | 17 connected components (same limitation, not U-Net/Otsu-specific) |
| Task 4 audit: `n_objects` (LLM) vs `n_objects_measured` | matched exactly on 12/12 test images |
| Task 4: `density_class` field | read "sparse" on 11/12 test images regardless of true density or object count — unreliable |

The full discussion, figures, and answers to the five set questions are in `Report/Report.pdf`.

## Key finding worth flagging

The pipeline's audited numeric fields (`n_objects`, `mean_area`) were exact on every test image —
the LLM reliably copied grounded numbers rather than inventing them. Its own categorical judgement
on top of those same numbers (`density_class`, `quality_flag`) was not reliable: `density_class`
barely varied regardless of the true density label, and `quality_flag` never flagged a single image
for review, including two ground-truth "dense" images with 75+ true objects. This is discussed in
the report as a distinct failure mode from hallucination — the model isn't inventing a number, but
its judgement built on a correct number can still be wrong, and nothing in the JSON schema alone
would reveal that without checking against an independent ground truth or deterministic threshold
rule.

## References

- Ronneberger, O., Fischer, P. and Brox, T. (2015) 'U-Net: Convolutional Networks for Biomedical
  Image Segmentation', MICCAI 2015.
- Otsu, N. (1979) 'A Threshold Selection Method from Gray-Level Histograms', IEEE Trans. SMC.
- van der Walt, S. et al. (2014) 'scikit-image: image processing in Python', PeerJ 2:e453.
- Ollama (2024) local LLM runtime, https://ollama.com — models used: `llama3.2`, `llama3.2-vision`.
