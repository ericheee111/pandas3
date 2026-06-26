# pandas 2.x private optimization migration validation log

This file records what has been checked during the migration. Full ASV is not
available in this environment, so ASV results must not be claimed unless a
future run records exact commands and output.

## Batch 0/1: inventory and existing migration audit

Status: in progress; documentation-only changes.

### Commands executed

- `git status --short --branch` in `../pandas`.
- `git log --oneline --decorate --graph -80` in `../pandas`.
- `git status --short --branch` in this repo.
- `git log --oneline --decorate --graph -80` in this repo.
- Parsed `../pandas_git_export_20260626_zenB_huang/README.txt`.
- Counted export commits from `all_commits_name_status.txt`:
  175 total commits, 84 merge commits, 89 non-merge private optimization
  candidates after excluding the v2.3.3 baseline and README commit.
- Inspected existing target commits:
  `d978bd13ee`, `28df2030b5`, `f6d861f8a6`, `8e0304f403`,
  `f39ba34d1d`, `ee85531203`.

### Findings

- `../pandas` has uncommitted state:
  - unstaged `asv_bench/asv.conf.json`, matching the export dirty worktree
    theme.
  - staged `_libs/reshape.pyx`, `_libs/tslibs/offsets.pyx`, and
    `core/algorithms.py`, which are not part of the export dirty worktree and
    require later export/fixup comparison before use as evidence.
- This repo has an unrelated uncommitted `AGENTS.md` change. It is intentionally
  excluded from migration commits.
- Existing target commits already cover the first audit set:
  - `d978bd13ee` covers `eba6d76f8c`.
  - `28df2030b5` covers `2e964bd967` and `171fe464a6`; it documents
    `1e811cf41e`/`33da51b84c` as already present and skips `02d2113331`.
  - `f6d861f8a6` covers SwissTable implementation and integration.
  - `8e0304f403` covers `844af97539` and `aae84a43d5`.
  - `f39ba34d1d` covers the main groupby loop optimizations and intentionally
    does not migrate `a041e1e5b3` `group_sum`.
  - `ee85531203` is target-only corrective work for duplicated algorithm
    migration.

### Checks executed for this batch

- Documentation diff review.
- `git diff --check` before commit.

### Checks not executed

- Full ASV: unavailable/unreliable in this environment.
- Full pandas test suite: not run for documentation-only changes.
- Cython rebuild: not run for documentation-only changes.

### Future batch validation requirements

- For Cython/C/C++ batches, verify `.pyx`, `.pxd`, `.pxi`, header, and
  `meson.build` declarations before tests.
- For Python-layer batches, run the smallest relevant pytest files named in the
  source export plus adjacent pandas3 tests.
- Record ASV benchmark names, even when ASV is not executed.

## Pending ASV benchmark mapping

- `stat_ops.Correlation.*` for nancorr.
- `frame_methods.Reindex.time_reindex_axis1*` and relevant take benchmarks for
  take fast paths.
- SwissTable benchmark coverage in `asv_bench/benchmarks/swisstable.py`.
- GroupBy aggregation benchmarks for mean/prod/min/max/cum*/last/quantile.
- Join benchmarks for object join indexer, groupsort/take, and many-to-many
  `sort=False`.
- `series_methods.Fillna.*` for `pad_inplace`.
- datetime offset/vectorized benchmarks for `shift_months`,
  `_shift_bdays`, `ints_to_pydatetime`, and `get_date_name_field`.
- reshape benchmarks for unstack and `get_dummies`.
- apply/astype/fillna/take/value_counts benchmarks for Batch 8.

## Batch 2a: index/search primitives and groupsort unroll

Status: implemented as a narrow sub-batch.

### Source commits covered

- `0744c0555f` / reconstructed `8d1d415`: `BaseMultiIndexCodesEngine`
  initialization uses one `int64` labels array and a `uint64` view, and computes
  `level_has_nans` vectorized from the shifted codes.
- `df0c75b9fd` / reconstructed `0b060de`: duplicate export of the same
  `BaseMultiIndexCodesEngine` optimization.
- `53b373262d` / reconstructed `29f4090`: typed Cython binary-search hooks from
  `index_class_helper.pxi.in`; migrated for non-complex unmasked engines.
- `403df2a143` / reconstructed `928c75c`: only the `groupsort_indexer` count and
  output loop unroll was migrated in this sub-batch. The join take helper part is
  still pending because it changes join control flow and needs targeted tests.

### pandas3 adaptation notes

- Kept pandas3's existing `ObjectEngine._get_loc_duplicates` implementation,
  which already uses `_bin_search` / `_bin_search_right` to preserve tuple-key
  semantics. Therefore reconstructed `b14a455` was not mechanically copied.
- Added `_searchsorted_right` beside `_searchsorted_left` and routed duplicate
  monotonic lookup through these hooks.
- Added numeric Cython type imports required by generated typed search methods.
- Left the larger many-to-many `sort=False` join rewrite and object join indexer
  specializations pending.

### Checks executed

- `git diff --check`
- Static inspection of edited declarations in:
  - `pandas/_libs/index.pyx`
  - `pandas/_libs/index_class_helper.pxi.in`
  - `pandas/_libs/algos.pyx`
- `python -m cython --version` to check compile-tool availability.

### Checks not executed

- Cython compile check: active Python reports `No module named cython`.
- Full pandas tests: not run because the active environment cannot rebuild the
  edited Cython extensions.
- ASV: not run in this environment.

### Follow-up validation

- Build pandas3 extensions with the project-supported Meson/Cython environment.
- Run focused index/join tests after rebuild:
  - `python -m pytest pandas/tests/indexes/multi`
  - `python -m pytest pandas/tests/indexes/test_engines.py`
  - `python -m pytest pandas/tests/reshape/merge`
- Run ASV for `groupsort_indexer` consumers and MultiIndex engine construction.
