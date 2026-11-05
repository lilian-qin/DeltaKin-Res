# DeltaKin-Res

Code and training data for GAT-GCN paired kinase selectivity prediction. The model
predicts `delta_pIC50_median` for a ligand and an ordered kinase pair (A, B).
Its V1 model scores the two kinases with a shared ligand/kinase network; V2
adds an aligned-residue correction to a frozen V1 checkpoint.

## Contents

- `data/pairs_final_wt.csv.gz`: 138,660 training pairs, including SMILES,
  kinase domain sequences, and selectivity labels (uncompressed CSV: 115 MB).
- `data/wt_esm/esm2_t33_650M_UR50D.pt`: global ESM2 embeddings for V1.
- `data/wt_esm/esm2_t33_650M_UR50D_residue.pt.part-*`: global and residue
  ESM2 embeddings for V2, split into three files for GitHub upload.
- `data/wt_esm/split.npz`: one fixed ligand-disjoint split with 110,922
  training, 13,866 validation, and 13,866 test samples.
- `training.py`, `inference.py`, `utils.py`, `models/`, `scripts/`, and
  `configs/`: model code and data preparation tools.

The graph dataset (`data/wt_esm/processed/selectivity_all_wt.pt`, about
1.5 GB) is generated locally from the CSV. Trained checkpoints are not
included.

## Setup

Use Python 3.10+ with enough free disk space (at least 3 GB) and RAM for the
graph dataset. Install PyTorch for your CPU/CUDA platform, then install the
remaining packages:

```sh
pip install -r requirements.txt
```

From the `deltakin-res` directory, restore the data files and check them:

```sh
gzip -dk data/pairs_final_wt.csv.gz
cat data/wt_esm/esm2_t33_650M_UR50D_residue.pt.part-* > data/wt_esm/esm2_t33_650M_UR50D_residue.pt
sha256sum data/pairs_final_wt.csv data/wt_esm/esm2_t33_650M_UR50D_residue.pt
```

Expected SHA-256 values:

```text
8d75c478c3fdab41149068becc9bb80a433dac523ed86813bf00b67b31a64e6c  data/pairs_final_wt.csv
c13eb14b8492e3aa79499b64f907407241a3c265244334d373e787445cefce01  data/wt_esm/esm2_t33_650M_UR50D_residue.pt
```

Build ligand graphs once. Six rows with invalid SMILES are skipped; the saved
split indexes the resulting 138,654-sample processed dataset:

```sh
python scripts/prepare_wt_data.py
```

## Train

Train a V1 GAT-GCN baseline, then train the V2 residue correction using the
resulting V1 checkpoint:

```sh
python training.py --pair-head difference_v1 --esm-cache data/wt_esm/esm2_t33_650M_UR50D.pt --run-dir runs/v1
python training.py --pretrained-v1 runs/v1/checkpoints/model_GAT_GCN_esm_rep0.model --run-dir runs/v2
```

The only ligand graph encoder is GAT-GCN. Training falls back to CPU if CUDA
is unavailable; use `--cuda-id` to select a CUDA device. `--run-dir` must name
a directory that does not exist.
Default hyperparameters are in `configs/training.json`; use `--help` for
available overrides. A run writes checkpoints, metrics, histories, and test
predictions below its run directory.

Evaluate the V2 checkpoint on the fixed test split:

```sh
python inference.py runs/v2/checkpoints/model_GAT_GCN_esm_rep0.model
```

To regenerate ESM features instead of using the included cache, run
`python scripts/precompute_esm.py --device cuda --batch-size 4 --output <new-path>`.
This requires `fair-esm` and the ESM2 t33 650M checkpoint plus its contact
regression weights in the Torch Hub checkpoint cache (or `--model-path`).

The CSV contains source references in its `refs` column. The labels are used
as provided; they are not recomputed from the individual pIC50 columns.
