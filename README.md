# Leakage-Free Federated and Private Malware Family Attribution on CIC-MalMem-2022

Code, raw results and figures for the paper:

> **Leakage-Free Evaluation of Federated and Privacy-Preserving Deep Learning for Memory-Based Malware Family Attribution and Unseen-Family Recognition**
> Submitted to *Computers, Materials & Continua* (Tech Science Press), special issue "Intelligent and Privacy-Preserving Malware Detection: Advances in Deep Learning, Memory Forensics, and Federated Security".

CIC-MalMem-2022 has about ten memory dumps per malware sample. Most published work splits the **dumps** at random, so dumps of the same sample appear in both training and test data. This repository re-evaluates the benchmark with a split **grouped by sample hash (SHA-256)**. Under that split it also measures federated learning, DP-SGD and the detection of unseen families.

![Protocol](figures/fig_pipeline.png)

## Main results (grouped split, mean of 3 seeds)

| Experiment | Finding |
|---|---|
| **E1: Leakage** | A random split inflates family macro-F1 by **10.09 pp** (RF) and **10.63 pp** (XGBoost), and category macro-F1 by 4.13–4.53 pp. Binary detection is unaffected (macro-F1 ≥ 0.9995). |
| **E2: Models** | Family macro-F1: XGBoost **0.4412**, RF 0.4309, MLP 0.3352, FT-Transformer 0.3278, 1D-CNN 0.3052 |
| **E3: Federated** (10 clients) | FedAvg keeps 95.2% / 88.4% / 78.4% of centralised macro-F1 (IID / Dir α=0.5 / Dir α=0.1). FedProx is within ±0.003 of FedAvg. Local-only training reaches 0.0974–0.2044. |
| **E4: DP-SGD** | Macro-F1 falls from 0.3007 to about 0.20 for ε between 12.92 and 91.76 (δ = 10⁻⁵) |
| **E5: Unseen families** (leave-one-family-out) | 99.69% of unseen-family dumps are still flagged as malware. Recognising a dump as a new family is harder: the best score (Energy) reaches mean AUROC 0.7845, and Mahalanobis is the weakest at 0.5933. |

## Repository layout

```
notebooks/malmem_kaggle.ipynb   full pipeline (E1–E5 + result figures), Kaggle-ready
src/malmem_kaggle.py            same pipeline as a plain Python script
analysis/stats.py               recomputes every number/p-value in the paper from results/*_raw.csv
results/                        per-seed raw CSVs, summary CSVs and results.json from the reported run
figures/                        all paper figures (+ make_fig_pipeline.py for Fig. 1)
paper/                          manuscript PDF and LaTeX source (TSP template)
```

## Dataset

The dataset is **not** redistributed here. You can get it from either source:

- Official (CIC, University of New Brunswick): https://www.unb.ca/cic/datasets/malmem-2022.html
- Kaggle mirror: https://www.kaggle.com/datasets/luccagodoy/obfuscated-malware-memory-2022-cic

Use the file `Obfuscated-MalMem2022.csv` (58,596 rows, 57 columns). The loader looks for this CSV under the data directory. It reads the category, the family and the sample hash from the `Category` field (`Category-Family-SHA256-k.raw`).

## How to run

**Kaggle (the setup used for the reported results):**
1. Create a notebook from `notebooks/malmem_kaggle.ipynb`.
2. Add the Kaggle dataset above as input.
3. Set Accelerator to **GPU T4 ×1** and turn Internet **On**. Internet is only used to `pip install opacus` for the privacy accountant.
4. Run all cells. Outputs are written to `/kaggle/working/malmem_results/`.

**Local:**
```bash
pip install -r requirements.txt
export MALMEM_DATA=/path/to/folder/with/csv
export MALMEM_OUT=./my_results
python src/malmem_kaggle.py            # full run (GPU recommended)
MALMEM_FAST=1 python src/malmem_kaggle.py   # quick smoke test: 1 seed, few epochs
```

**Reproduce the paper's statistics from the shipped results:**
```bash
python analysis/stats.py results/
```

## Experimental settings (summary)

- **Splits:** a 60/20/20 random split by record vs. a GroupShuffleSplit by SHA-256. Benign dumps have no sample id, so each benign dump forms its own group. Seeds 0, 1, 2.
- **Features:** 52 Volatility features. Three constant columns are dropped. Features are transformed with sign·log1p and standardised.
- **Models:** RF (300 trees), XGBoost (400 trees, depth 8, GPU hist), MLP (256-128-64, BatchNorm), FT-Transformer, 1D-CNN. Neural models use AdamW with early stopping.
- **Federated:** 10 clients, 60 rounds, 2 local epochs, FedAvg and FedProx (μ = 0.01). Partitions are IID and Dirichlet α ∈ {0.5, 0.1} over families, assigned by whole sample.
- **DP-SGD:** per-sample clipping with C = 1 via `torch.func`, noise σ ∈ {0.6, 1.0, 1.5}, and an Opacus RDP accountant with δ = 10⁻⁵.
- **Unseen families:** each of the 15 families is held out in turn. Scores compared: MSP, MSP+temperature scaling, Energy, MC-Dropout mutual information, EDL vacuity, one-vs-rest (DOC), kNN, Mahalanobis and FedAvg MSP.
- **Statistics:** paired t-tests over seeds (E1, E2) and Wilcoxon signed-rank tests over families (E5).

## Citation

See `CITATION.cff`. The full reference will be added after publication.

## License

The code is released under the MIT License. CIC-MalMem-2022 is subject to the license terms of the Canadian Institute for Cybersecurity. Please cite the dataset authors:

Carrier T, Victor P, Tekeoglu A, Lashkari AH. *Detecting obfuscated malware using memory feature engineering.* ICISSP 2022, pp. 177–188.
