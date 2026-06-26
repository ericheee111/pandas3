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

## Batch 2e: join indexer object fast paths

Status: implemented for object join indexers; many-to-many sort=False audited as
already covered.

### Source commits covered

- `07ccc6e28e` / reconstructed `5aa4d2d`: avoid post-hoc reordering for
  many-to-many grouped joins when `sort=False`.
- `52f9c792ec` / reconstructed `0a0060b`: add object ndarray specializations
  for monotonic join indexers.

### pandas3 adaptation notes

- pandas3 already had the `sort=False` grouped join structure in
  `inner_join` and `left_outer_join`, including direct output generation in
  original left order.
- Ported the object-specific `left_join_indexer_unique`,
  `left_join_indexer`, `inner_join_indexer`, and `outer_join_indexer` helper
  paths to pandas3 `pandas/_libs/join.pyx`.
- The fast paths are limited to the object fused-type specialization and
  C-contiguous input arrays. Other dtypes and non-contiguous arrays continue to
  use pandas3's existing generic code.
- Added low-level libjoin tests for object dtype unique, left, inner, and outer
  monotonic join indexers.

### Checks executed

- Static inspection of:
  - export patch sections for `07ccc6e28e` and `52f9c792ec`
  - reconstructed pandas2 commits `5aa4d2d` and `0a0060b`
  - pandas3 `pandas/_libs/join.pyx` current grouped and indexer paths
- `python -m py_compile pandas3/pandas/tests/libs/test_join.py`
- `git diff --check`

### Checks not executed

- Cython compile check: active Python reports `No module named cython`.
- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/libs/test_join.py`
  - `python -m pytest pandas/tests/reshape/merge/test_join.py -k sort`
  - `python -m pytest pandas/tests/reshape/merge/test_merge.py -k sort`
- Benchmark monotonic object index join workloads and many-to-many
  `sort=False` joins with high duplicate counts.

## Batch 7c: skiplist header optimization

Status: implemented.

### Source commits covered

- `91019df988` / reconstructed `caf7a1d`: optimize skiplist allocation and
  traversal behavior.

### pandas3 adaptation notes

- Ported the single-allocation `node_t` layout to
  `pandas/_libs/include/pandas/skiplist.h`, reducing per-node allocator calls
  from separate node/next/width allocations to one contiguous allocation.
- Added portable `SL_LIKELY`, `SL_UNLIKELY`, and `SL_PREFETCH_R` macros.
- Adapted traversal changes in `skiplist_get`, `skiplist_min_rank`,
  `skiplist_insert`, and `skiplist_remove`, including cached node values,
  conditional prefetching, and unrolled width updates.
- Kept pandas3's current `NAN` and `log2` usage instead of reintroducing the
  pandas2 export's `PANDAS_NAN` and `Log2` helpers.

### Checks executed

- Static inspection of:
  - export patch section for `91019df988`
  - reconstructed pandas2 commit `caf7a1d`
  - pandas3 `pandas/_libs/include/pandas/skiplist.h`
- Confirmed pandas3 Meson uses `c_std=c17` and `cpp_std=c++17`.
- `git diff --check`

### Checks not executed

- Standalone C syntax check: no `gcc`, `cc`, or `clang` binary is available in
  the active shell environment.
- Cython compile check: active Python reports `No module named cython`.
- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/window/test_rolling.py -k "quantile or median"`
  - `python -m pytest pandas/tests/window/test_expanding.py -k "quantile or median"`
  - `python -m pytest pandas/tests/series/methods/test_rank.py`
- Benchmark rolling quantile/median and ranking workloads that exercise the
  skiplist insert/remove/get paths.

## Batch 9a: Xiecheng ASV benchmark suite

Status: implemented.

### Source commits covered

- `a3739ab1e4` / reconstructed `0d05357`: add initial Xiecheng benchmark file.
- `42b376312f` / reconstructed `f2dc10f`: add DataFrame construction benchmark.
- `baca9d6de2` / reconstructed `4d3e2e5`: fix categorical benchmark mutation.
- `fcdd01e0a4` / reconstructed `d12115c`: update data generation to reuse
  precomputed arrays for construction benchmarks.

### pandas3 adaptation notes

- Added `asv_bench/benchmarks/xiecheng.py` using the final exported benchmark
  shape instead of replaying intermediate states.
- Converted non-ASCII section comments to concise ASCII structure and renamed
  the duplicate `time_groupby_agg_count` benchmark to
  `time_groupby_agg_nunique`.
- Preserved the later fixes: object arrays for string-like columns,
  precomputed constructor inputs, and writing categorical conversions into new
  columns.

### Checks executed

- Static inspection of:
  - export patch sections for `a3739ab1e4`, `42b376312f`, `baca9d6de2`, and
    `fcdd01e0a4`
  - reconstructed pandas2 final benchmark file at `d12115c`
- `python -m py_compile pandas3/asv_bench/benchmarks/xiecheng.py`
- `git diff --check`

### Checks not executed

- ASV: not run in this environment.
- Runtime benchmark import/execution: active Python cannot import pandas because
  NumPy is missing.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m asv run -b xiecheng --quick`
  - targeted full ASV runs for `xiecheng.GroupByAgg`,
    `xiecheng.MergeJoin`, `xiecheng.StringCategorical`, and
    `xiecheng.Constructor`.

## Batch 5a: add_overflowsafe contiguous fast paths

Status: implemented.

### Source commits covered

- `9e640a5d2c` / reconstructed `da350b3`: use raw pointers for contiguous
  add_overflowsafe operands instead of `PyArray_MultiIter`.
- `3f77b3b691` / reconstructed `92c98b7`: add a left-contiguous/right-scalar
  fast path and tighten same-shape checks.
- `82081460b3` / reconstructed `c3c0728`: cache contiguity flags and add
  scalar/same-shape tests.

### pandas3 adaptation notes

- Replaced pandas3 `add_overflowsafe` loop in
  `pandas/_libs/tslibs/np_datetime.pyx` with three paths:
  left C-contiguous plus scalar right, both operands C-contiguous and same
  shape, and the existing `PyArray_MultiIter` fallback.
- Preserved pandas3 overflow behavior by keeping the outer `try`/`except`
  around all paths under `@cython.overflowcheck(True)`.
- Added tests for scalar, same-shape with NaT, non-contiguous fallback, and
  overflow raising.

### Checks executed

- Static inspection of:
  - export patch sections for `9e640a5d2c`, `3f77b3b691`, and `82081460b3`
  - reconstructed pandas2 final implementation at `c3c0728`
  - pandas3 current `add_overflowsafe`
- `python -m py_compile pandas3/pandas/tests/tslibs/test_np_datetime.py`
- `git diff --check`

### Checks not executed

- Cython compile check: active Python reports `No module named cython`.
- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/tslibs/test_np_datetime.py -k AddOverflowSafe`
  - `python -m pytest pandas/tests/arrays/timedeltas -k add`
  - `python -m pytest pandas/tests/arithmetic -k "datetime or timedelta"`
- Benchmark datetime/timedelta array addition for scalar, contiguous same-shape,
  and broadcast/non-contiguous cases.

## Batch 5b: datetime object construction and date-name audit

Status: partially implemented; one source commit already covered.

### Source commits covered

- `e951e5f721` / reconstructed `1b17cae`: optimize `ints_to_pydatetime` by
  using CPython datetime C constructors.
- `a22b61a327` / reconstructed `d6455bd`: avoid repeated `.capitalize()` calls
  inside `get_date_name_field` loops.

### pandas3 adaptation notes

- Ported `ints_to_pydatetime` constructor changes in
  `pandas/_libs/tslibs/vectorized.pyx`, using `datetime_new`, `date_new`, and
  `time_new` after `import_datetime()`.
- Removed the now-unused `date`, `datetime`, and `time` cimports.
- pandas3 `pandas/_libs/tslibs/fields.pyx` already had the
  `get_date_name_field` optimization: localized day/month names are capitalized
  once before the loop and reused.

### Checks executed

- Static inspection of:
  - export patch sections for `e951e5f721` and `a22b61a327`
  - reconstructed pandas2 commits `1b17cae` and `d6455bd`
  - pandas3 current `vectorized.pyx` and `fields.pyx`
- `git diff --check`

### Checks not executed

- Cython compile check: active Python reports `No module named cython`.
- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/series/accessors/test_dt_accessor.py -k "date or time"`
  - `python -m pytest pandas/tests/indexes/datetimes/test_misc.py -k "day_name or month_name"`
- Benchmark `timeseries.DatetimeAccessor.time_dt_accessor_date` and
  `timeseries.DatetimeAccessor.time_dt_accessor_time`.

## Batch 5c: shift_months and BusinessDay shift fast paths

Status: implemented.

### Source commits covered

- `9686250639` / reconstructed `cd5c272`: add an early-day fast path and
  contiguous 1-D direct loop for `shift_months(day_opt=None)`.
- `42c76d772f` / reconstructed `1ab19b1`: avoid full date decomposition in
  `BusinessDay._shift_bdays` when whole-day epoch arithmetic is sufficient.

### pandas3 adaptation notes

- Added a direct pointer loop for contiguous 1-D `shift_months` inputs when
  `day_opt is None`; non-contiguous, higher-dimensional, and day-opt paths keep
  the existing `PyArray_MultiIter` behavior.
- Avoided `get_days_in_month` for days `<= 28` in the month-shift fast path.
- Reworked `_shift_bdays` to compute weekday from epoch days and preserve the
  intraday remainder without `pandas_datetime_to_datetimestruct`.
- Kept SemiMonth-specific changes from `d1f4ed4720` separate because they also
  touch `get_days_in_month` and conflict with the already-migrated object-array
  construction helper.

### Checks executed

- Static inspection of:
  - export patch sections for `9686250639` and `42c76d772f`
  - reconstructed pandas2 commits `cd5c272` and `1ab19b1`
  - pandas3 current `pandas/_libs/tslibs/offsets.pyx`
- `python -m py_compile pandas3/pandas/tests/arithmetic/test_datetime64.py
  pandas3/pandas/tests/tseries/offsets/test_business_day.py`
- `git diff --check`

### Checks not executed

- Cython compile check: active Python reports `No module named cython`.
- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/arithmetic/test_datetime64.py -k shift_months`
  - `python -m pytest pandas/tests/tseries/offsets/test_business_day.py`
  - `python -m pytest pandas/tests/tseries/offsets/test_offsets.py -k "DateOffset or BusinessDay"`
- Benchmark month-shift arithmetic and BusinessDay array addition workloads.

## Batch 5d: SemiMonth narrow date rebuild

Status: partially implemented.

### Source commits covered

- `d1f4ed4720` / reconstructed `f074c1e`: narrow
  `SemiMonthBegin`/`SemiMonthEnd` date rebuild logic and speed up month length
  checks.

### pandas3 adaptation notes

- Ported the `get_days_in_month` arithmetic shortcut in
  `pandas/_libs/tslibs/ccalendar.pyx`.
- Added `fast_days_in_month` in `pandas/_libs/tslibs/offsets.pyx` for
  SemiMonth internal loops.
- Split `SemiMonthOffset._apply_array` into Begin and End paths so Begin avoids
  unnecessary month-end checks and End only clamps to month-end when needed.
- Added an array-vs-scalar SemiMonth test covering Begin/End and positive/negative
  offsets.
- Did not migrate the export commit's `pandas/core/dtypes/cast.py` rollback or
  deleted tests because those conflict with the already-migrated
  `construct_1d_object_array_from_listlike` helper and would remove coverage.

### Checks executed

- Static inspection of:
  - export patch section for `d1f4ed4720`
  - reconstructed pandas2 commit `f074c1e`
  - pandas3 current `ccalendar.pyx`, `offsets.pyx`, and object-array helper
    migration state
- `python -m py_compile pandas3/pandas/tests/tseries/offsets/test_month.py`
- `git diff --check`

### Checks not executed

- Cython compile check: active Python reports `No module named cython`.
- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/tseries/offsets/test_month.py -k SemiMonth`
  - `python -m pytest pandas/tests/tseries/offsets/test_common.py -k SemiMonth`
  - `python -m pytest pandas/tests/dtypes/cast/test_construct_object_arr.py`
- Benchmark SemiMonthBegin/SemiMonthEnd array addition workloads.

## Batch 8a: row-wise apply label cache

Status: implemented.

### Source commits covered

- `7d7fb8bf02` / reconstructed `c8634f7`: optimize row-wise
  `DataFrame.apply(axis=1)` string label access and reused-Series reference
  handling.
- `1246018d48` / reconstructed `391ce67`: partially covered only for the
  row-wise apply cache infrastructure that row `7d7fb8bf02` depends on.

### pandas3 adaptation notes

- Added a `Series`-side row-apply label-to-position cache and consulted it
  before `apply_if_callable` for non-callable labels.
- Wired `FrameColumnApply.series_generator` to attach the cache to the reused
  row `Series`, to update the cached row values after `set_values`, and to
  reset refs only after an applied function returned a `Series`.
- Kept duplicate-column semantics by enabling the cache only when
  `self.columns.is_unique`; duplicate labels continue through normal Series
  lookup.
- Removed the per-row `BlockPlacement` rebuild from
  `SingleBlockManager.set_values`, matching the export intent while preserving
  the existing manager locations for the reused row object.
- Did not migrate the rest of `1246018d48` in this batch. Its astype, fillna,
  take, value_counts, Arrow, putmask, and generic-manager changes remain broad
  and require separate pandas3 semantic review.

### Checks executed

- Static inspection of:
  - export patch sections for `1246018d48` and `7d7fb8bf02`
  - reconstructed pandas2 commits `391ce67` and `c8634f7`
  - pandas3 current `core/apply.py`, `core/series.py`, and
    `core/internals/managers.py`
- `python -m py_compile pandas/core/apply.py pandas/core/series.py
  pandas/core/internals/managers.py pandas/tests/apply/test_frame_apply.py`
- `git diff --check`

### Checks not executed

- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- Cython compile check: active Python reports `No module named cython`.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/apply/test_frame_apply.py -k "apply_axis1_label_lookup or apply_axis1_string_label_lookup or apply_axis1_callable_key or duplicate_columns_keep_series_lookup"`
  - `python -m pytest pandas/tests/apply/test_frame_apply.py -k "axis1 and apply"`
- Benchmark `frame_methods.Apply.time_frame_apply_axis_1_*` and add a focused
  benchmark for repeated string-label row access if the existing ASV coverage is
  insufficient.

## Batch 8b: object-int64 value_counts dense sorted path

Status: implemented.

### Source commits covered

- `1246018d48` / reconstructed `391ce67`: partially covered for the
  object-dtype integer `value_counts(dropna=False)` dense sorted fast path.

### pandas3 adaptation notes

- Added `lib.value_counts_object_int64_dense_sorted` with a dense counting path
  for object arrays containing only int64-range integer scalars and a compact
  value range.
- Integrated the helper into `value_counts_internal` only for large object
  arrays when `sort=True`, `ascending=False`, `normalize=False`, and
  `dropna=False`; all other modes fall back to pandas3's existing hashtable
  implementation.
- Preserved pandas3 stable sort semantics by treating the helper result as
  already sorted by descending count with first-occurrence tie order.
- Adapted the exported Cython implementation to compute the min/max range size
  through Python integers before casting to `Py_ssize_t`, avoiding signed int64
  overflow for extreme sparse ranges.
- Did not migrate the same commit's nullable-integer dense path, astype/fillna,
  Arrow fill-null, putmask, or take changes in this batch.

### Checks executed

- Static inspection of:
  - export patch section for `1246018d48`
  - reconstructed pandas2 commit `391ce67`
  - pandas3 current `pandas/_libs/lib.pyx`, `pandas/_libs/lib.pyi`, and
    `pandas/core/algorithms.py`
- `python -m py_compile pandas/core/algorithms.py
  pandas/tests/base/test_value_counts.py`
- `git diff --check`

### Checks not executed

- Cython compile check: active Python reports `No module named cython`.
- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/base/test_value_counts.py -k "object_integer_large_sorted"`
  - `python -m pytest pandas/tests/base/test_value_counts.py -k value_counts`
  - `python -m pytest pandas/tests/series/methods/test_value_counts.py`
- Benchmark object-dtype integer `value_counts(dropna=False)` workloads,
  especially compact-range large arrays and sparse-range fallback cases.

## Batch 8c: putmask scalar and bool take fast paths

Status: implemented.

### Source commits covered

- `1246018d48` / reconstructed `391ce67`: partially covered for
  `putmask_without_repeat` scalar assignment and 2-D bool-to-object
  `take_nd` with NaN fill.

### pandas3 adaptation notes

- Added the scalar `putmask_without_repeat` fast path before listlike shape
  checks, preserving existing listlike exact-length behavior.
- Adapted the exported bool-object take helper to pandas3's current
  `_take_nd_ndarray` `mask_info` and `allow_fill` flow.
- Kept the fast path restricted to 2-D bool arrays promoted to object output
  with floating NaN fill; all other take modes continue through existing
  pandas3 dispatch tables.

### Checks executed

- Static inspection of:
  - export patch section for `1246018d48`
  - reconstructed pandas2 commit `391ce67`
  - pandas3 current `core/array_algos/putmask.py` and
    `core/array_algos/take.py`
- `python -m py_compile pandas/core/array_algos/putmask.py
  pandas/core/array_algos/take.py pandas/tests/array_algos/test_putmask.py
  pandas/tests/test_take.py`
- `git diff --check`

### Checks not executed

- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/array_algos/test_putmask.py`
  - `python -m pytest pandas/tests/test_take.py -k "2d_bool or fill_nonna"`
  - `python -m pytest pandas/tests/internals/test_internals.py -k take_nd`
- Benchmark bool DataFrame reindex/take paths and scalar `putmask_without_repeat`
  assignment workloads.

## Batch 8d: Arrow integer fillna fast path

Status: implemented.

### Source commits covered

- `1246018d48` / reconstructed `391ce67`: partially covered for
  single-chunk integer ArrowExtensionArray scalar `fillna`.

### pandas3 adaptation notes

- Added `_fill_null_with_single_chunk_integer_scalar` inside the
  `HAS_PYARROW` block and reused pandas3's existing
  `pyarrow_array_to_numpy_and_mask` helper.
- Routed `ArrowExtensionArray.fillna` through this helper before the general
  `_safe_fill_null` kernel path.
- Kept the fast path restricted to valid scalar fills, single chunk arrays, and
  integer pyarrow types; multi-chunk, non-integer, invalid scalar, and kernel
  fallback cases retain the current pandas3 implementation.

### Checks executed

- Static inspection of:
  - export patch section for `1246018d48`
  - reconstructed pandas2 commit `391ce67`
  - pandas3 current `core/arrays/arrow/array.py` and Arrow utility helpers
- `python -m py_compile pandas/core/arrays/arrow/array.py
  pandas/tests/extension/test_arrow.py`
- `git diff --check`

### Checks not executed

- Runtime pandas tests: active Python cannot import pandas because NumPy is
  missing.
- PyArrow runtime tests: not run for the same pandas import dependency issue.
- ASV: not run in this environment.

### Follow-up validation

- After installing/building the pandas3 development environment, run:
  - `python -m pytest pandas/tests/extension/test_arrow.py -k fillna_single_chunk_integer_scalar`
  - `python -m pytest pandas/tests/extension/test_arrow.py -k fillna`
  - `python -m pytest pandas/tests/arrays/masked/test_arrow_compat.py`
- Benchmark Arrow integer scalar fillna with single chunk arrays containing
  nulls, and include multi-chunk fallback coverage.
