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
