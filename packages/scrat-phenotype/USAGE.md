# ScRAT phenotype MCP package

ScRAT predicts a patient-level phenotype from a set of single-cell expression
profiles. This package implements the three stages described by Mao et al.
(Bioinformatics, 2024; doi:10.1093/bioinformatics/btae067): cell-type-aware
sample mixup, multi-head self-attention over unordered cells, and mean pooling
followed by a phenotype classifier. It also calculates high-attention occurrence
values (HOV) and cell-type relevance scores (R-scores).

## Scope and limitations

- Binary phenotype classification is implemented. Multi-class and continuous
  traits, baseline models, and the paper's COVID benchmark experiments are not
  reproduced.
- This is an independent implementation from the published method, not a wrapper
  around the unlicensed reference source. No pretrained model or reference data
  is included. Numerical agreement with the paper is not claimed.
- Cell-type labels are used to keep mixup within corresponding populations and
  to aggregate R-scores. The attention model itself does not consume cell-type
  labels as features.
- Input samples must have one phenotype each. Samples below `min_cells` are
  excluded from mixup/training. Prediction requires every sample to contain at
  least `crop_cells` cells.

## Input

All tools accept an `.h5ad` file with cells as observations. Required `obs`
columns default to:

- `patient_id`: patient or sample identifier;
- `Outcome`: binary phenotype (training and mixup only);
- `cell_type`: population used for mixup and interpretation.

The default feature input is `obsm["X_pca"]`. Set `embedding_key` to null to use
`X` instead; `scrat_train` then fits PCA only on training cells and stores the
transform in the model. Use normalized, comparable expression values when using
`X`. Prediction requires the same features in the same order. The lexicographically
second phenotype is the positive class.

Every output path must be new and differ from the inputs. Input files are never
modified.

## Install

Python 3.12 is recommended. Network access is needed only to install dependencies.

```bash
python -m venv .venv
. .venv/bin/activate
pip install -r src/requirements.txt
```

Start the stdio server with:

```bash
python src/scrat_mcp.py
```

An MCP client configuration looks like:

```json
{"mcpServers": {"scrat": {
  "command": "/path/to/.venv/bin/python",
  "args": ["/path/to/this-package/src/scrat_mcp.py"]}}}
```

No network access is needed after installation.

## Tools

### scrat_mixup

Generates `n_pseudo` pseudo-samples. Two source samples are mixed with a
Beta(`alpha`, `alpha`) coefficient within each shared cell population. Populations
unique to one source are scaled by their source's mixing coefficient and receive
Gaussian noise. `mixup_mode` can be `different` (the paper configuration), `same`,
or `any`.

The compressed NPZ contains the concatenated cell matrix `X`, per-cell
`sample_index` and `cell_types`, per-pseudo-sample soft `labels`, `lambdas`, and
`source_samples`, plus `label_classes`. It does not contain a trained model.

### scrat_train

Splits patients into train, validation, and held-out test sets before mixup. Only
pseudo-samples are used for gradient updates, matching the paper configuration.
Random fixed-size cell crops are used during training; validation and test use
`test_crops` crops per patient. The checkpoint with the lowest validation loss is
saved.

Paper-derived defaults include 300 pseudo-samples of about 10,000 cells, 500
cells per crop, 20 training crops, 50 evaluation crops, eight attention heads,
128-dimensional embeddings, and 100 epochs. The model artifact includes the PCA
transform when applicable, class names, feature order, and configuration. Held-out
per-sample probabilities are written to CSV; AUC is null if the test split has only
one class.

### scrat_predict

Loads a model produced by `scrat_train` and writes one row per sample with the
predicted class, mean crop probability for the positive class, and positive-class
vote fraction. Following the paper, the predicted class uses majority vote over
thresholded crop predictions; it can therefore differ from a threshold applied to
the mean probability. To interpret the prediction, provide both `cell_scores_path`
and `cell_type_scores_path`.

The cell CSV contains zero-based input cell row index, HOV, crop appearances, and
appearances-normalized HOV. The cell-type CSV contains R-scores averaged over
crops. For each crop, HOV counts how often a cell is in the top `top_k` attention
columns across every row and head; R-score sums HOV within a cell type and divides
by the number of cells of that type in the full input sample.

## Resource use

Mixup memory grows approximately as `n_pseudo x mixup_cells x n_features`; the
paper defaults at 50 features require roughly 600 MB for one float32 pseudo-sample
matrix, before overhead. Attention memory and time grow quadratically with
`crop_cells`. The full paper configuration is computationally expensive on CPU;
start with small values for a smoke test and use a supported GPU for large runs.
Runtime requires local read/write access only.
