# Validation

## How this was verified

- Environment: Windows 11 x64, Python 3.12.7, CPU-only execution. A fresh virtual
  environment was built solely from the eight distributions in
  `src/requirements.txt`. Every direct dependency also resolved with `--no-deps`
  on PyPI, the Tsinghua mirror, and the Aliyun mirror.
- Test input: a synthetic H5AD with 1,200 cells, 16 genes, 20 patients (60 cells
  each), two phenotype classes, and two cell types. Class-dependent signal was
  inserted in disjoint gene/cell-type blocks. This input tests execution, not
  biological validity.
- `scrat_mixup` — generated two different-phenotype pseudo-samples (72 cells in
  eight PCA dimensions); checked array shapes, soft-label bounds, source-class
  contrast, and readable NPZ output at a path without an `.npz` suffix. An
  existing output and a missing `obs` column were both rejected.
- `scrat_train` — ran two epochs using six pseudo-samples, 24-cell crops, and a
  12/4/4 patient train/validation/test split. Checked the model artifact, stored
  PCA shape, four held-out predictions, and best-epoch selection. The synthetic
  held-out accuracy and AUC were both 1.0; these numbers are smoke-test results,
  not a benchmark.
- `scrat_predict` — predicted all 20 synthetic patients with three 24-cell crops;
  checked probability and vote-fraction bounds, 1,200 per-cell HOV rows, 40
  per-cell-type R-score rows, nonnegative HOV, and finite R-scores.
- MCP stdio: started `src/scrat_mcp.py`, initialized an MCP client, and confirmed
  exactly `scrat_mixup`, `scrat_train`, and `scrat_predict` were registered. All
  three tools were then called successfully through the protocol on small inputs;
  a nonexistent H5AD returned an MCP error result as expected.
- Repository checks: `python tools/catalog.py validate` passed for all four
  packages (with the expected pending-approval warning for this new package);
  `python -m unittest discover -s tests -v` ran 78 tests successfully with one
  Windows symlink-privilege skip.

## What remains unverified

- Agreement with the paper's COVID benchmarks, figures, tables, or the reference
  implementation's numerical outputs. No real dataset or pretrained model was
  available in this validation.
- Biological or clinical validity, performance on large datasets, sparse matrices,
  GPU execution, and Linux/macOS runtime behavior.
- Multi-class and continuous traits, which are outside this package's scope.
- Redistribution review by maintainers. The package contains new implementation
  code only; the unlicensed upstream source is not vendored.
