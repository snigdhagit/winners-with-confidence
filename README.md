# Post-Selection Inference for Top-K Selection

Reference implementation and reproduction code for:
> **Flexible Inference for Winners with Conditional Validity** (under revision JRSSB)
> Soham Bakshi, Lingjun Gao, Zijun Gao, Snigdha Panigrahi
> [arXiv:2607.18545](https://arxiv.org/abs/2607.18545)

The repository compares five inference procedures across Gaussian, Binomial, Bradley–Terry–Davidson, and variable-importance experiments. It also contains a real-data application to the SPRINT trial, where age subgroups are ranked by estimated treatment benefit.

---

## Methods compared

| Method | Label in code | Description |
|---|---|---|
| **Randomized PSI** (ours) | `Randomized PSI` | Randomizes top-`k` selection and constructs conditional confidence intervals by inverting a grid-based selective pivot. |
| **Polyhedral PSI** | `Polyhedral PSI` | Conditions on the deterministic top-`k` selection event and performs truncated-Gaussian selective inference. |
| **Standard** | `Standard` | Selects and estimates on the full dataset, then reports ordinary confidence intervals without a selection correction. |
| **Data Splitting** | `Data Splitting` | Uses one part of the data for selection and the held-out part for inference. |
| **Zoom Correction** | `Zoom Correction` | Applies a step-down correction to account for winner or top-`k` selection. |

Within each Monte Carlo replication, the methods can be run on the same generated dataset by setting `share_same_data_across_methods=True`. This makes their selection quality, coverage, interval length, and computation time directly comparable.

---

## Main experiments

The notebook contains four simulation families.

| Experiment | Main target | Current dimensions/settings | Figure produced |
|---|---|---|---|
| Gaussian (Conditional coverage) | Top-`k` coordinates of a Gaussian mean vector | `M=20`, `k=3`, fix heteroskedastic diagonal covariance | `example1.pdf` |
| Gaussian | Top-`k` coordinates of a Gaussian mean vector | `M=20`, `k=3`, plug-in heteroskedastic diagonal covariance | `Gaussian_Example.pdf` |
| Binomial / dosage | Top-`k` success probabilities | `M=10`, `k=3`, plug-in delta-method covariance | `Binomial_Example.pdf` |
| Bradley–Terry–Davidson | Top-`k` latent item strengths | `M=10`, `k=3`, ties allowed | `BTD_Example.pdf` |
| Variable importance | Top-`k` variable-importance targets | `p=10`, `k=3`, fixed-nuisance holdout construction | `VI_Example.pdf` |

Each PDF contains three panels:

1. selection quality;
2. marginal coverage rate;
3. average confidence-interval length.

In all four simulation families, **signal strength** is defined as the
standardized separation between the true `k`-th and (`k+1`)-th largest targets. The data-generating parameters are calibrated so that this signal strength is
exactly **0.3 in the weak-signal setting** and **2.0 in the strong-signal
setting**. These values are standardized gaps rather than raw differences
between two means or probabilities.

The randomization level is generally set to $\varepsilon = \log \binom{M}{k}.$

---

## Installation

This project requires Python 3.10 or later. Clone the repository and install the required packages:

```bash
git clone https://github.com/snigdhagit/winners-with-confidence.git
cd winners-with-confidence

python3 -m venv .venv
source .venv/bin/activate

python3 -m pip install --upgrade pip
python3 -m pip install numpy pandas scipy matplotlib seaborn \
    scikit-learn statsmodels joblib tqdm jupyter
```

---

## Repository layout

The refactored repository is organized into four main directories:

```text
src/                              inference methods
├── Standard.py
├── Randomized_PSI.py
├── Polyhedral_PSI.py
├── Data_Splitting.py
├── Zoom_Correction.py
└── Inverse_Pivot.py

simulation/                       data generators and simulation drivers
├── Combined_Sim.py
├── Standard_Sim.py
├── Randomized_PSI_Sim.py
├── Polyhedral_PSI_Sim.py
├── Data_Splitting_Sim.py
├── Zoom_Correction_Sim.py
├── data_generator.py
└── simulation_utils.py

plot/                             plotting functions and figure drivers
├── Plot_Function.py
├── Fig1.py
├── Fig5.py
└── figures/

result/                           SPRINT tables and real-data figures

Top-K_Post_Selection_Inference.ipynb
README.md
```

Run scripts from the repository root so that imports from `src`, `simulation`, and `plot` resolve correctly.

---

## Quick start with the notebook

Start Jupyter from the repository root:

```bash
jupyter notebook Top-K_Post_Selection_Inference.ipynb
```

The notebook is organized as follows:

1. imports;
2. confidence-interval primitives;
3. model implementations;
4. data generators;
5. common statistic, covariance, and signal-strength helpers;
6. method-specific simulation drivers;
7. plotting utilities;
8. randomized-PSI numerical fixes and the faster CI implementation;
9. final Gaussian, Binomial, Bradley–Terry–Davidson, and variable-importance experiments;
10. SPRINT real-data application.

Run Sections 1–8 once. Then run only the experiment block needed in Section 9, or continue to the SPRINT section after providing the required data files.

The current full settings use `B=500` for Gaussian, Binomial, and variable importance and `B=200` for Bradley–Terry–Davidson. The variable-importance experiment is especially expensive because its final holdout size and bootstrap count are `500_000` and `500`.

---

## Running the refactored figure scripts

From the repository root, use module execution rather than running a file from inside the `plot` directory:

```bash
python -m plot.Fig1
python -m plot.Fig5
```

Generated simulation figures should be written to `plot/figures/`. Before a full run, use a small replication count to verify imports, paths, and output creation.

---

## SPRINT real-data application

The SPRINT section ranks age subgroups by the adjusted difference in the probability of a favorable outcome under intensive versus standard treatment.

The code currently uses four mutually exclusive age groups:

- `50–64`;
- `65–69`;
- `70–74`;
- `75+`.

Within each subgroup, a logistic regression adjusts for demographics, treatment history, baseline blood pressure, kidney function, laboratory measurements, medication use, smoking, and cardiovascular history. Standardized success probabilities are computed under both treatment assignments, and their difference is used as the subgroup-specific treatment-effect estimate. A delta-method covariance estimate is then passed to the post-selection procedures.

### Required data

The SPRINT data are not included in this repository. The notebook expects the following files in its working directory:

```text
baseline.csv
outcomes.csv
```

Both files must contain a unique participant identifier named `MASKID`. The notebook performs a one-to-one merge and currently checks for exactly 9,361 matched participants.

At minimum, the analysis uses these variables:

```text
MASKID, INTENSIVE, AGE, FEMALE, RACE4, N_AGENTS, SMOKE_3CAT,
ASPIRIN, SUB_CLINICALCVD, SUB_SUBCLINICALCVD, STATIN, SBP, DBP,
EGFR, SCREAT, CHR, GLUR, HDL, TRR, UMALCR, BMI,
EVENT_PRIMARY, T_PRIMARY
```

Do not commit restricted individual-level SPRINT files to a public repository. Add their paths to `.gitignore` if they are stored inside the local project directory.

### Current inference settings

The SPRINT analysis currently uses:

- top-`k` size `k=2`;
- 90% confidence intervals (`alpha=0.10`);
- `epsilon = log(comb(4, 2))` for Randomized PSI;
- Data Splitting ratios 30/70, 50/50, and 70/30, where the first percentage is used for selection;
- 100 random seeds in the stability analysis.

The main SPRINT table reports the selected subgroup, rank, point estimate, 90% confidence interval, and interval length. The stability plot counts how often each subgroup is both selected and significant, where significance means that the selected subgroup's 90% confidence interval excludes zero.

### Follow-up-time note

The current code sets:

```python
FOLLOWUP_DAYS = 1000
```

Although some notebook labels say "3-year outcome," 1,000 days is not exactly three years. For an exact three-year endpoint, change the setting to `3 * 365` (1,095 days); otherwise, rename the outcome and figure labels to "1,000-day outcome."

---

## Reproducibility settings

The main simulation blocks currently use:

```python
ALPHA = 0.05
SEED = 123
K = 3
share_same_data_across_methods = True
sigma = "unknown"
zoom_sigma_mode = "mean"
```

Randomized PSI and Data Splitting depend on random seeds. Report both the main fixed-seed results and the stability analysis across repeated seeds. The deterministic full-data methods use the same selected top-`k` set when their selection rule is identical, although their confidence intervals differ.

---

## Outputs

The main notebook produces:

| Output | Contents |
|---|---|
| `example1.pdf` | Gaussian selection quality, coverage, CI length, and conditional coverage |
| `Gaussian_Example.pdf` | Gaussian selection quality, coverage, and CI length |
| `Binomial_Example.pdf` | Binomial selection quality, coverage, and CI length |
| `BTD_Example.pdf` | Bradley–Terry–Davidson selection quality, coverage, and CI length |
| `VI_Example.pdf` | Variable-importance selection quality, coverage, and CI length |
| SPRINT result table | Selected top-2 subgroups and 90% confidence intervals |
| SPRINT stability bar plot | Counts of selection plus significance across random seeds |

Unless an explicit path is supplied, the notebook writes the four simulation PDFs to the current working directory. The refactored scripts should instead save publication figures under `plot/figures/` and real-data outputs under `result/`.

---

## Citation


