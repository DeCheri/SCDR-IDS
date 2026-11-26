# SCDR-IDS

Code and data accompanying the paper

> **SCDR-IDS: Scale-Calibrated Directional Flow Relations for Intrusion Detection in IoT Networks**
> Changqin Xu, Si Li, and Wu Wang. Submitted to *IEEE Internet of Things Journal*.

SCDR-IDS is a flow-level intrusion detector. It compares each NetFlow record with the outgoing traffic of its source and the incoming traffic of its destination, and it expresses endpoint activity as a share of the observed flows so that the inputs do not drift when the observation window changes in size.

## Release status

This first release covers everything needed to rebuild the data splits and to check the reported evaluation: preprocessing, split construction, the evaluation protocol, the held-out predictions of the paper, and the scripts that redraw the result figures. The model implementation and the trained checkpoints will be added once the review process is complete.

| Component | Status |
| --- | --- |
| NetFlow-v3 preprocessing and feature-group split (`preprocessing/`) | released |
| Held-out split and source-role-disjoint split (`splits/`, `results/splits/`) | released |
| Metrics, decision rules, observation-scale views (`evaluation/`) | released |
| Held-out test predictions of SCDR-IDS and its controls (`results/predictions/`) | released |
| Data and scripts for Figs. 3-8 (`results/figure_data/`, `figures/`) | released |
| Directional relational encoder, count calibration, training code | after review |
| Trained checkpoints | after review |

## Repository layout

```text
SCDR-IDS/
├── preprocessing/prepare.py          # development benchmark: 80/10/10 feature-group split, scalers, vocabularies
├── splits/heldout_sample.py          # held-out evaluation split (65/15/20 feature-group split on a row sample)
├── evaluation/
│   ├── metrics.py                    # Macro-F1, accuracy, weighted F1, macro P/R, FAR, per-class F1, flip rate
│   ├── decision_rules.py             # logit adjustment, benign-quantile alarm threshold, fixed-alarm relabeling
│   └── observation_views.py          # fixed targets and nested background-retention views (Section VII-A)
├── scripts/verify_heldout_table.py   # recomputes Table IV from the released predictions
├── figures/plot_figures.py           # redraws Figs. 3-8 from results/figure_data
└── results/
    ├── predictions/                  # held-out test predictions (3 datasets x 3 variants x 3 seeds)
    ├── splits/                       # split summaries and the CIC source-role-disjoint row partition
    └── figure_data/                  # numbers behind Figs. 3-8
```

## Installation

Python 3.11 was used for all experiments.

```bash
python -m pip install -r requirements.txt
```

## Quick check

```bash
python scripts/verify_heldout_table.py   # prints Table IV (held-out test graph)
python figures/plot_figures.py           # writes Figs. 3-8 to figures/output/
```

`verify_heldout_table.py` recomputes Macro-F1, accuracy, macro recall, and benign FAR (mean ± sample SD over seeds 17, 29, and 43) for No Direction, Uncalibrated, and SCDR-IDS on UNSW-NB15, CSE-CIC-IDS2018, and ToN-IoT.

## Data

The experiments use three NetFlow-v3 releases (Luay *et al.*, 2025): NF-UNSW-NB15-v3, NF-CSE-CIC-IDS2018-v3, and NF-ToN-IoT-v3. Download the official ZIP archives from the NetFlow dataset page of the University of Queensland (<https://staff.itee.uq.edu.au/marius/NIDS_datasets/>). The archives are not redistributed here.

### Development benchmark (Table III)

```bash
python preprocessing/prepare.py --dataset cic --source NF-CSE-CIC-IDS2018-v3.zip --root work
```

Flows with identical values of the 49 predictors (47 numerical and two protocol fields) form one group. IP addresses, timestamps, and labels are excluded from the predictors. Groups are hashed into 80% training, 10% validation, and 10% test partitions, so identical feature vectors never cross partitions. Scalers and protocol vocabularies are fitted on the training partition only, and IP addresses serve only as graph node identifiers.

### Held-out test graph (Table IV, Figs. 3, 5, 6)

```bash
python splits/heldout_sample.py --dataset cic --archive NF-CSE-CIC-IDS2018-v3.zip --out data/heldout
```

Rows are subsampled by a fixed hash of the row index (UNSW 1/4, CIC 1/12, ToN-IoT 1/20). Feature groups are then hashed into 65/15/20 training, validation, and test partitions. `results/splits/heldout_split_*.json` lists the archive checksums and the class counts of every partition, so a rebuilt split can be checked against the one used in the paper. `row_ids` in the prediction files refer to zero-based row indices in the official CSV files.

### Source-role-disjoint split on CIC (Tables IX-X, Fig. 7)

`results/splits/cic_source_role_row_partition.npz` assigns every row of NF-CSE-CIC-IDS2018-v3 to training (0), validation (1), test (2), or excluded (255; the six single-source attack classes). The nine retained classes, their source IPs, and the time ranges of all fifteen classes are listed in `class_source_coverage.csv`; `label_mapping.json` maps the original labels to the nine-class label set.

## Evaluation protocol

* **Macro-F1** averages per-class F1 over the full class list of a dataset. Classes absent from an evaluation graph are scored as zero.
* **FAR** is the fraction of benign flows that receive any attack label.
* **Threshold rule.** A flow is flagged only if 1 - P(benign) exceeds the 0.999 (or 0.99) quantile of the same score on benign flows of the threshold-calibration subset. Flagged flows receive their highest-scoring attack class. The quantile is computed on validation data only.
* **Observation-scale views.** About 20% of the held-out test flows are fixed as targets. Background flows are retained with probability p ∈ {0, 0.125, 0.25, 0.5, 0.75, 1} under two hash salts, and all graph statistics are recomputed on each view before scoring. The prediction-flip rate compares the predictions on the target flows with those on the full graph.

## Citation

```bibtex
@article{xu2026scdrids,
  title   = {{SCDR-IDS}: Scale-Calibrated Directional Flow Relations for Intrusion Detection in {IoT} Networks},
  author  = {Xu, Changqin and Li, Si and Wang, Wu},
  journal = {IEEE Internet of Things Journal},
  note    = {under review},
  year    = {2026}
}
```

## License

The code in this repository is released under the MIT License (see `LICENSE`). The NetFlow-v3 datasets are subject to the terms of their original providers.
