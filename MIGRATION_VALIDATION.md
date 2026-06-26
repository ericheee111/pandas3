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

## Batch 2b: RangeIndex concat helper

Status: implemented.

### Source commits covered

- `1b285b9697` / reconstructed `28b54ab`: move RangeIndex concat planning into
  a lower-level helper to reduce Python-loop overhead in the all-RangeIndex path.

### pandas3 adaptation notes

- The pandas2 helper was not copied verbatim because pandas3's `_concat`
  contains behavior added after pandas2:
  - mixed-index fallback preserves integer `Index` shallow-copy behavior.
  - repeated identical non-empty ranges use `np.tile` instead of generic
    concatenation.
  - single non-empty ranges preserve their original `step`.
- Added `concat_range_indexes` to `pandas._libs.lib` and its `.pyi` stub.
- Kept object construction and fallback behavior in `RangeIndex._concat`; the
  Cython helper returns compact planning tags only.

### Checks executed

- `python -m py_compile pandas/core/indexes/range.py`
- `git diff --check`
- Static inspection of:
  - `pandas/_libs/lib.pyx`
  - `pandas/_libs/lib.pyi`
  - `pandas/core/indexes/range.py`

### Checks not executed

- Runtime pandas import/tests: active Python cannot import pandas because NumPy
  is missing.
- Cython compile check: active Python reports `No module named cython`.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/indexes/ranges`
  - `python -m pytest pandas/tests/reshape/concat`
- Benchmark concat cases involving all `RangeIndex`, repeated identical
  `RangeIndex`, non-consecutive `RangeIndex`, and mixed index inputs.

## Batch 7a: khash put micro-optimizations

Status: implemented.

### Source commits covered

- `2777612697` / reconstructed `13e217a`: add portable likely/unlikely branch
  prediction macros and use them for the common empty-slot path in `kh_put`.
- `42dc387d2c` / reconstructed `1d031db`: cache `h->keys` and `h->flags` in
  local pointers inside the `kh_put` macro expansion.

### pandas3 adaptation notes

- Applied only inside the vendored khash macro used by pandas hashtable code.
- Kept the macro fallback portable for non-GNU/non-Clang compilers.
- Did not include skiplist allocator/prefetch changes or `roll_sum`; those are
  larger and remain separate pending B7 items.

### Checks executed

- `git diff --check`
- Static inspection of `pandas/_libs/include/pandas/vendored/klib/khash.h`

### Checks not executed

- C/Cython compile check: active Python reports `No module named cython`.
- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- Rebuild pandas3 extensions in a complete development environment.
- Run hashtable/factorize tests and any SwissTable/legacy hashtable coverage.
- Benchmark factorize/hashtable-heavy workloads.

## Batch 6a: stable sort controls for safe_sort

Status: implemented.

### Source commits covered

- `51a2b98159` / reconstructed `26d98b8`: use stable sorting in
  `safe_sort` for ARM-friendly factorize sorting.
- `31a63abb20` / reconstructed `84722ae`: extend stable sorting to the
  `safe_sort` remapping and mixed-type fallback argsort calls.
- `1c4c300a36` / reconstructed `e81e938`: add an explicit `kind` parameter
  to `safe_sort` and call it with `kind="stable"` from `factorize(sort=True)`.

### pandas3 adaptation notes

- Kept `safe_sort` default behavior at `kind="quicksort"` for external callers.
- Threaded `SortKind` through pandas3's current `safe_sort` implementation,
  including the mixed-integer fallback and the code remapping reverse argsort.
- Preserved the existing pandas3 SwissTable-aware `_get_hashtable_algo` path
  added by the earlier migration commits.

### Checks executed

- `python -m py_compile pandas/core/algorithms.py`
- `git diff --check`
- Static inspection of `pandas/core/algorithms.py` call sites touched by the
  batch.

### Checks not executed

- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/test_algorithms.py`
  - `python -m pytest pandas/tests/test_sorting.py`
- Benchmark `factorize(sort=True)`, mixed object `safe_sort`, and any
  architecture-specific ARM sorting workloads from the private ASV suite.

## Batch 6b: algos Cython scalar and fill hot paths

Status: partially implemented; partially already covered by pandas3.

### Source commits covered

- `4c4a5096dc` / reconstructed `4c72539`: replace index arithmetic in
  `kth_smallest_c` with pointer movement.
- `0e2677777a` / reconstructed `6438637`: use C `isnan()` for float scalar
  `checknull`.
- `706991b28f` / reconstructed `b84fc9f`: audited as already covered by the
  current pandas3 `pad_2d_inplace` no-limit branch.
- `38f97b58aa` / reconstructed `d287828`: audited as already covered by the
  current pandas3 `pad_inplace` no-limit branch.
- `6fd0359851` / reconstructed `af50d81`: audited as already covered by the
  current pandas3 `pad_inplace` loop-unroll implementation.

### pandas3 adaptation notes

- Kept the pandas3 `checknull(object val)` signature. The pandas2 patch's
  `inf_as_na` handling was not restored because that parameter no longer
  exists in pandas3.
- Added a narrow test for `np.float64` scalar handling that preserves pandas3
  semantics: NaN is null, ordinary finite floats and infinities are not.
- Confirmed current pandas3 already has the pad fast paths, including leading
  missing-prefix skips, no-limit branches, and 4-way `pad_inplace` unrolling.

### Checks executed

- `python -m py_compile pandas/tests/dtypes/test_missing.py`
- `git diff --check`
- Static inspection of:
  - `pandas/_libs/algos.pyx`
  - `pandas/_libs/missing.pyx`
  - `pandas/tests/dtypes/test_missing.py`

### Checks not executed

- Cython compile check: active Python reports `No module named cython`.
- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/test_algos.py`
  - `python -m pytest pandas/tests/dtypes/test_missing.py -k checknull`
- Benchmark kth-smallest/quantile-style workloads, fillna forward/backward
  workloads, and object/string isnull workloads from the private ASV suite.

## Batch 3a: object-array construction helpers

Status: implemented.

### Source commits covered

- `fefbc7b5a4` / reconstructed `bc9ee0d`: move
  `construct_1d_object_array_from_listlike` into a Cython helper for top-level
  object copying.
- `d72753640f` / reconstructed `b7cda5d`: use direct NumPy object-array
  construction for flat list/tuple inputs while preserving nested list-like
  behavior through the helper fallback.

### pandas3 adaptation notes

- Added a `pandas._libs.lib` stub declaration because pandas3 maintains
  `_libs/lib.pyi`.
- Kept pandas3's existing `is_list_like` helper and typing around
  `construct_1d_object_array_from_listlike`.
- Did not include the unrelated `Py_INCREF` call-site casts from the earlier
  pandas2 commit because pandas3's current imports and Cython state do not
  require them for this helper.

### Checks executed

- `python -m py_compile pandas/core/dtypes/cast.py`
- `python -m py_compile pandas/tests/dtypes/cast/test_construct_object_arr.py`
- `git diff --check`
- Static inspection of:
  - `pandas/_libs/lib.pyx`
  - `pandas/_libs/lib.pyi`
  - `pandas/core/dtypes/cast.py`
  - `pandas/tests/dtypes/cast/test_construct_object_arr.py`

### Checks not executed

- Cython compile check: active Python reports `No module named cython`.
- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/dtypes/cast/test_construct_object_arr.py`
- Benchmark object-array construction from flat tuples/lists, nested lists,
  generators, and ndarray inputs.

## Batch 3b: lib object pointer helpers

Status: implemented.

### Source commits covered

- `a60133b265` / reconstructed `49024a3`: add a contiguous fast path to
  `array_equivalent_object` using raw `PyObject**` array data.
- `c064f8cd15` / reconstructed `0816e09`: avoid per-element borrowed-object
  INCREF/DECREF overhead in `eq_NA_compat`.
- `defb42e92e` / reconstructed `e36f118`: apply the required
  `PyObject_RichCompareBool` pointer casts in the `array_equivalent_object`
  fast path.
- `454f5e27f1` / reconstructed `ba18686`: optimize `fast_zip` tuple writes
  by writing through `PyObject*` tuple/data pointers.

### pandas3 adaptation notes

- Kept pandas3's existing `array_equivalent_object` behavior for cases where
  only one side is an ndarray; the contiguous fast path returns `False` for
  that case just like the existing multi-iterator path.
- Consolidated the affected CPython APIs into explicit `Python.h` declarations
  so raw `PyObject*` call sites are typed consistently.
- Reused the current pandas3 `_libs/lib.pyx` layout, including the previously
  ported `construct_1d_object_array_from_listlike` helper.

### Checks executed

- `git diff --check`
- Static inspection of `pandas/_libs/lib.pyx` C API call sites.

### Checks not executed

- Cython compile check: active Python reports `No module named cython`.
- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/dtypes/test_missing.py`
  - `python -m pytest pandas/tests/indexes/test_base.py -k equivalent`
  - `python -m pytest pandas/tests/indexes/test_engines.py -k object`
- Benchmark `ObjectEngine.get_loc`, object-array equality, tuple zipping,
  and MultiIndex/grouping workloads that call `fast_zip`.

## Batch 4a: dense get_dummies helper

Status: implemented for dense get_dummies; unstack patch not directly migrated.

### Source commits covered

- `15dba60291` / reconstructed `d47726a`: add a low-level dense get_dummies
  writer that fills the Fortran-order output buffer from factorized codes.
- `8232deb8a0` / reconstructed `cdd104a`: audited but not directly migrated;
  the pandas2 patch splits `new_mask` writes from numeric value writes, while
  pandas3's current `libreshape.unstack` signature no longer receives a
  `new_mask` argument.

### pandas3 adaptation notes

- Added `_get_dummies_dense_from_codes` to `pandas._libs.reshape` and the
  pandas3 `.pyi` stub.
- Kept the small-cardinality path in Python because column-wise writes are
  efficient on the Fortran-order result matrix.
- Preserved fallback behavior for dtypes not covered by the fused Cython helper.
- Did not change `unstack`; porting row 76 would require a pandas3-specific
  redesign across the current reshape caller and mask handling.

### Checks executed

- `python -m py_compile pandas/core/reshape/encoding.py`
- `python -m py_compile pandas/tests/reshape/test_get_dummies.py`
- `git diff --check`
- Static inspection of:
  - `pandas/_libs/reshape.pyx`
  - `pandas/_libs/reshape.pyi`
  - `pandas/core/reshape/encoding.py`
  - `pandas/tests/reshape/test_get_dummies.py`

### Checks not executed

- Cython compile check: active Python reports `No module named cython`.
- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/reshape/test_get_dummies.py`
  - `python -m pytest pandas/tests/reshape/test_reshape.py -k unstack`
- Benchmark `reshape.GetDummies.time_get_dummies_1d` and dense/sparse
  get_dummies variants with wide cardinality.

## Batch 7b: roll_sum loop structure

Status: implemented.

### Source commits covered

- `66863407cc` / reconstructed `4d53c7d`: move the monotonic-window
  `roll_sum` branch out of the per-row loop, cache previous start/end bounds,
  and avoid redundant non-monotonic cleanup stores.

### pandas3 adaptation notes

- Applied to pandas3's current `pandas/_libs/window/aggregations.pyx`
  `roll_sum` implementation.
- Kept skiplist changes from `91019df988` separate because they alter memory
  layout and allocator behavior in `skiplist.h`.

### Checks executed

- `git diff --check`
- Static inspection of `pandas/_libs/window/aggregations.pyx`

### Checks not executed

- Cython compile check: active Python reports `No module named cython`.
- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/window/test_rolling.py -k sum`
  - `python -m pytest pandas/tests/window/test_expanding.py -k sum`
- Benchmark rolling sum/mean workloads with monotonic and non-monotonic bounds.

## Batch 2c: is_monotonic rollback audit

Status: not migrated by design.

### Source commits covered

- `eaaffa04b4` / reconstructed `979986f`: attempted block-based
  `is_monotonic` optimization.
- `0b958ce8fd` / reconstructed `5f74016`: reverted the optimization after a
  Xiecheng MergeJoin failure and restored upstream behavior.

### pandas3 adaptation notes

- pandas3 currently uses the upstream scalar comparison loop in
  `pandas/_libs/algos.pyx`.
- The reverted optimization was not reintroduced because the export history
  identifies a correctness failure in MergeJoin scenarios.

### Checks executed

- Static inspection of:
  - export patch sections for `eaaffa04b4` and `0b958ce8fd`
  - `pandas/_libs/algos.pyx` current `is_monotonic`
- `git diff --check`

### Checks not executed

- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/indexes/test_monotonic.py`
  - `python -m pytest pandas/tests/reshape/merge/test_merge.py -k sort`
- Reproduce the original Xiecheng MergeJoin workload before considering any
  future replacement optimization.

## Batch 2d: ObjectEngine duplicate lookup audit

Status: already covered in pandas3; no code change.

### Source commits covered

- `72efd58130` / reconstructed `b14a455`: added an
  `ObjectEngine._get_loc_duplicates` override for monotonic object indexes.

### pandas3 adaptation notes

- pandas3 already has an `ObjectEngine._get_loc_duplicates` override in
  `pandas/_libs/index.pyx`.
- The pandas3 implementation uses `_bin_search` and `_bin_search_right` rather
  than `ndarray.searchsorted`, preserving tuple-as-single-key behavior while
  keeping the monotonic duplicate fast path.
- No duplicate migration was applied.

### Checks executed

- Static inspection of:
  - export patch section for `72efd58130`
  - reconstructed pandas2 commit `b14a455`
  - pandas3 `pandas/_libs/index.pyx` current ObjectEngine implementation
- `git diff --check`

### Checks not executed

- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/indexes/base_class/test_indexing.py -k get_loc`
  - `python -m pytest pandas/tests/indexes/object/test_indexing.py -k get_loc`
- Benchmark `indexing_engines.ObjectEngineIndexing.time_get_loc` for monotonic
  duplicate object indexes.
