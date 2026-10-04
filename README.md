# Robustness or Artifact?

**Re-examining adversarial URL perturbation results on the PhiUSIIL phishing dataset**

We reproduce a 2026 phishing URL detection study and show its results come from a dataset flaw: every safe URL in PhiUSIIL is a bare `https://www` homepage. A one-line rule scores 99.6%, matching trained AI models; real accuracy is about 86%, and the reported attack robustness hides real weaknesses. We propose fairer ways to test phishing detectors.

> **Status:** Work in progress. Reproduction and core analysis are complete; statistical validation and extension to more models are ongoing.

---

## Background

This project starts from the following paper:

> Ahamed, T., Kakon, S. C., Al Farid, F., Uddin, J., & Abdul Karim, H. B. (2026).
> *An integrated evaluation protocol for adversarial robustness, generalization, and explanation stability in URL-based phishing detection.*
> Frontiers in Computer Science, 8, 1834407. [https://doi.org/10.3389/fcomp.2026.1834407](https://doi.org/10.3389/fcomp.2026.1834407)

The paper trains phishing URL classifiers on the **PhiUSIIL** dataset, reports 99.6–99.8% accuracy, and shows that the models collapse under simple URL changes such as adding a `login.` subdomain.

**Dataset:** PhiUSIIL Phishing URL Dataset (Prasad & Chandra, 2024), UCI Machine Learning Repository, id 967.
[https://archive.ics.uci.edu/dataset/967/phiusiil+phishing+url+dataset](https://archive.ics.uci.edu/dataset/967/phiusiil+phishing+url+dataset)

---

## Key findings

| # | Finding | Evidence |
|---|---------|----------|
| 1 | The benign class is a single template | 100% of safe URLs match `https://www.domain.tld` with no path, vs 1.03% of phishing URLs |
| 2 | A trivial rule matches the published models | Regex rule: **0.9958** accuracy, vs Logistic Regression 0.9957 and XGBoost 0.9972 |
| 3 | Models mostly learned the template | Original model catches only **29.7%** of phishing URLs that look like the template |
| 4 | Real URL-only signal is much weaker | Template-neutral model: **0.8647** accuracy, 0.9159 AUROC |
| 5 | Apparent robustness is blindness to lookalikes | Under TLD-swap and typo attacks, lookalike domains of safe sites are accepted as safe 100% of the time |
| 6 | The subdomain collapse is a false-alarm cascade | Phishing recall stays at 1.00 while safe-URL accuracy drops to 0.0003; real subdomains appear in 2.88% of safe vs 54.34% of phishing URLs |
| 7 | A real evasion attack is hidden | Moving phishing domains to `.com/.net/.org/.co` lowers template-neutral phishing recall from 0.7254 to 0.6142 |
| 8 | Typosquatting goes undetected | Typo variants of legitimate domains are accepted as safe by both models |

All numbers come from our own experiments with Logistic Regression on a host-disjoint test split (n = 34,354). Confidence intervals and results for more models are in progress.

---

## Reproduction of the original paper

| Model | Our test accuracy | Paper (95% CI) |
|-------|-------------------|----------------|
| XGBoost | 0.9972 | 0.9974 [0.9968, 0.9979] |
| Logistic Regression | 0.9957 | 0.9962 [0.9956, 0.9968] |

Our subdomain-attack results (0.469 / 0.414 / 0.410 at depths 1–3) also reproduce the paper's reported collapse (0.490 / 0.436 / 0.432).

---

## Method

- **Data:** PhiUSIIL URLs only. Labels inverted so that `1 = phishing`. URLs normalised (NFKC, control-character removal, lowercased host, IDNA encoding) and deduplicated, giving 235,358 URLs.
- **Split:** host-disjoint 70 / 15 / 15 using `GroupShuffleSplit` (seed 1337): 164,901 / 36,103 / 34,354.
- **Features:** character 3–5-gram TF-IDF (`min_df = 3`).
- **Models:** Logistic Regression (L2, C = 1.0) and XGBoost (300 trees, depth 6, learning rate 0.1), matching the paper.
- **Template-neutral input ("host-only"):** keeps only the host and removes every `www` label, so safe and phishing URLs share the same form.
- **Attacks:** subdomain injection, typo, TLD swap and homoglyph at depths 1–3, implemented from the paper's description. Padding and path attacks cannot be evaluated fairly on PhiUSIIL, because no safe URL has a path.
- **Metrics:** accuracy, F1 and AUROC, plus per-class phishing recall (evasion) and benign accuracy (false alarms).

---

## Getting started

### Requirements

- Python 3.9+
- Google Colab, or a local machine with about 8 GB RAM

```bash
pip install ucimlrepo xgboost scikit-learn pandas tldextract
```

### Quick check: the template rule

```python
import pandas as pd
from ucimlrepo import fetch_ucirepo

data = fetch_ucirepo(id=967)
df = data.data.features[['URL']].copy()
df['label'] = 1 - data.data.targets['label'].values   # 1 = phishing

tmpl = df['URL'].str.match(r'^https://www\.[^/?#]+/?$')
print(pd.crosstab(df['label'], tmpl, normalize='index'))
```

The benign row (label 0) shows about 100% in the `True` column.

### Running the full analysis

Open the notebook(s) in `notebooks/` and run all cells from top to bottom. Each notebook reloads the data, so it can run on its own.

---

## Repository structure

```
.
├── notebooks/
│   ├── 01_reproduction.ipynb        # Clean XGBoost and LR baselines
│   ├── 02_template_audit.ipynb      # Template check and rule baseline
│   ├── 03_template_neutral.ipynb    # Host-only evaluation
│   └── 04_attacks_per_class.ipynb   # Mutation attacks and per-class results
├── results/                          # Saved CSV result tables
├── docs/
│   └── Research_Proposal.docx        # Full proposal and work plan
└── README.md
```

*Planned layout; folders are being populated as work progresses.*

---

## Roadmap

- [x] Reproduce the paper's clean results and subdomain collapse
- [x] Audit benign-class structure and build the rule baseline
- [x] Template-neutral evaluation and hard-phishing subset
- [x] Per-class attack analysis (Logistic Regression)
- [ ] Per-class analysis for XGBoost (and BERT if GPU time allows)
- [ ] Bootstrap confidence intervals, McNemar tests and multiple seeds
- [ ] Registered-domain (eTLD+1) disjoint split check
- [ ] Debiased benign URL set with realistic paths and subdomains
- [ ] Re-run all six attack families on the debiased set
- [ ] Paper, preprint and submission

---

## Limitations

- The host-only input discards path information, so its results are a lower bound, not the best achievable accuracy.
- The hard-phishing subset is small (n = 145).
- For TLD-swap, typo and homoglyph attacks, a "benign" label on a modified safe URL is not well defined; we read those results as acceptance of lookalike domains.
- Attack operators are our own implementation of the paper's description.
- Current results use one split and one model; validation is in progress.

---

## Acknowledgements

The benign-template pattern in PhiUSIIL was first noted in independent public code audits, including
[phish-drift](https://github.com/awesomedudeworld13/phish-drift),
[phishing-url-detection](https://github.com/ollivianguy/phishing-url-detection),
[phishguard](https://github.com/yuvraj1624/phishguard) and
[phishing-url-classifier](https://github.com/Panshul-mishra/phishing-url-classifier).
Our work independently verifies and quantifies it and examines its effect on published robustness results.

We thank the authors of the original paper and of the PhiUSIIL dataset for making their work publicly available.

---

## Citation

If you use this work, please cite the original paper and dataset:

```bibtex
@article{ahamed2026integrated,
  title   = {An integrated evaluation protocol for adversarial robustness, generalization, and explanation stability in URL-based phishing detection},
  author  = {Ahamed, Tanvir and Kakon, Shawon Chakrabarty and Al Farid, Fahmid and Uddin, Jia and Abdul Karim, Hezerul Bin},
  journal = {Frontiers in Computer Science},
  volume  = {8},
  pages   = {1834407},
  year    = {2026},
  doi     = {10.3389/fcomp.2026.1834407}
}

@article{prasad2024phiusiil,
  title   = {PhiUSIIL: A diverse security profile empowered phishing URL detection framework based on similarity index and incremental learning},
  author  = {Prasad, Arvind and Chandra, Shalini},
  journal = {Computers \& Security},
  pages   = {103545},
  year    = {2024},
  doi     = {10.1016/j.cose.2023.103545}
}
```

A citation for this project will be added once the paper is available.

---

## Authors

- Mutwassim Haider — Air University Islamabad / Creative Technologies
- Supervisor: Sir Ali Javid Peerzada
