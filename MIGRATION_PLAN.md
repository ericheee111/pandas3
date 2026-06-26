# pandas 2.x private optimization migration plan

Generated from the 2026-06-26 KPandas export and the current `pandas3`
target branch.

## Ground truth

- Source export: `../pandas_git_export_20260626_zenB_huang`.
- Export metadata: `README.txt` reports `commit_count=175`, branch `master`,
  head `03bfebc987577edc3b1407fc2382706e59dec389`.
- Export contains 84 merge commits. Excluding merges plus the baseline commits
  `Initial commit: pandas v2.3.3` and `Add README.md`, there are 89 migration
  candidate commits.
- Reconstructed pandas 2.x repo: `../pandas`, branch `kpandas-optimized`,
  `v2.3.3-90-g17f5a4d-dirty`. It has 89 optimization commits plus a local
  `fixup: correct 4 files where date-order replay produced wrong blobs`.
- Source of truth for intent and original diffs is the export. The reconstructed
  repo is only a secondary comparison point because replay and fixup commits can
  differ from the original export.
- Target repo: this checkout, branch `migration/pandas2-private-optimizations`.
  Existing uncommitted `AGENTS.md` changes are not part of this migration and
  must not be staged into migration commits.

## Existing target-side coverage

The target branch already has these migration commits after `v3.0.3`:

| Target commit | Covers | Export commits / notes |
| --- | --- | --- |
| `d978bd13ee` | `nancorr` per-column-pair no-filter fast path | `eba6d76f8c` |
| `28df2030b5` | take fast paths and object take refcount reduction | `2e964bd967`, `171fe464a6`; notes `1e811cf41e` and `33da51b84c` already in pandas3 and skips `02d2113331` as a backward step |
| `f6d861f8a6` | SwissTable implementation, config, factorizer integration, tests, ASV benchmark | SwissTable bootstrap/sync/type/factorizer commits listed in Batch 1 |
| `8e0304f403` | `maybe_convert_objects` homogeneous-block fast path and complex `zval` reuse | `844af97539`, `aae84a43d5` |
| `f39ba34d1d` | groupby loop optimizations | groupby rows in Batch 1; explicitly does not migrate `a041e1e5b3` `group_sum` due Kahan/skipna risk |
| `3fa2758641` | ASV configuration update | corresponds to the export-time uncommitted ASV config diff |
| `ee85531203` | repair for duplicated algorithm migration return type | target-only corrective commit |

Batch 1 is therefore an audit-only batch unless a later code review finds a
specific mismatch.

## Batch dependency map

| Batch | Theme | Dependencies | Target area | Risk |
| --- | --- | --- | --- | --- |
| B0 | Inventory and audit docs | none | `MIGRATION_PLAN.md`, `MIGRATION_VALIDATION.md` | documentation only |
| B1 | Existing target migration audit | B0 | `algos.pyx`, take templates, `swisstable*`, `lib.pyx`, `groupby.pyx` | avoid duplicate ports; verify commit coverage |
| B2 | Index and join hot paths | B1 audit | `index.pyx`, `index_class_helper.pxi.in`, `join.pyx`, selected `algos.pyx` | join semantics, monotonic rollback, object engine correctness |
| B3 | lib/object helpers | B1 audit | `lib.pyx`, `core/dtypes/cast.py`, `core/indexes/range.py` | object identity, NA equality, dtype inference |
| B4 | reshape/unstack/get_dummies | B3 where cast helpers overlap | `reshape.pyx`, `reshape.pyi`, `core/reshape/encoding.py` | mask shape, Fortran-order output, dummy dtype semantics |
| B5 | tslib datetime offsets | B3 where cast tests overlap | `tslibs/np_datetime.pyx`, `offsets.pyx`, `fields.pyx`, `vectorized.pyx`, `ccalendar.pyx` | overflow, timezone/NaT, business-day edge cases |
| B6 | algorithms and missing helpers | B1 audit | `algos.pyx`, `missing.pyx`, `core/algorithms.py` | duplicated signatures, stable-sort semantics, NA behavior |
| B7 | rolling/window and low-level containers | B6 only for shared algos awareness | `window/aggregations.pyx`, `skiplist.h`, `khash.h` | C macro/header portability |
| B8 | Python-layer broad optimizations | B3/B6 where shared helpers overlap | `core/apply.py`, `series.py`, internals, array algos, tests | Copy-on-Write, EA/Arrow nullable dtype behavior |
| B9 | benchmarks and ASV metadata | relevant code batches | `asv_bench/benchmarks/*`, ASV docs | do not claim ASV execution locally |

## Candidate commit inventory

The `Decision` column is the current migration disposition. `pending` means the
optimization remains to be evaluated and ported in its assigned batch.

| # | Export hash | Subject | Category | Batch | Decision | Source files |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | `26f530f608` | C 实现swisstable原型 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `asv_bench/benchmarks/swisstable.py`, `_libs/meson.build`, `_libs/swisstable*`, `core/algorithms.py`, `core/config_init.py`, tests |
| 2 | `3cbe555493` | feat(swisstable): 添加 SwissTable C++17 核心实现 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable/swisstable.hpp` |
| 3 | `063703e782` | perf(swisstable): 添加批处理优化方法和 resize | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable/swisstable.hpp` |
| 4 | `55b5cb4acb` | feat(swisstable): 添加类型别名和实例化框架 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `.gitignore`, `_libs/swisstable/swisstable_aliases.hpp`, instances headers |
| 5 | `0bd2e202c9` | feat(swisstable): 添加 pyx 文件结构和 SwissInt64Map 类 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable/swisstable.pyx` |
| 6 | `bb5778c2b4` | feat(swisstable): 添加 int64 辅助函数 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable/swisstable.pyx` |
| 7 | `e9a2ff724a` | feat(swisstable): 添加 SwissUInt64Map 类 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable/swisstable.pyx` |
| 8 | `752d2193df` | feat(swisstable): 添加 uint64 辅助函数和 SwissInt32Map 类 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable/swisstable.pyx` |
| 9 | `66103d5ee8` | feat(swisstable): 添加 SwissUInt32Map 类 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable/swisstable.pyx` |
| 10 | `0f92566a5c` | feat(swisstable): 添加 SwissFloat64Map 类 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable/swisstable.pyx` |
| 11 | `4d37bc846c` | feat(swisstable): 添加 SwissFloat32Map 类 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable/swisstable.pyx` |
| 12 | `e7ec6a4d43` | feat(swisstable): 添加 SwissInt16Map 类 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable/swisstable.pyx` |
| 13 | `39b3922be7` | feat(swisstable): 添加 SwissUInt16Map 类 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable/swisstable.pyx` |
| 14 | `cc3c7cf8f8` | feat(swisstable): 添加 SwissInt8Map 类 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable/swisstable.pyx` |
| 15 | `9028ac986d` | feat(swisstable): 添加 SwissUInt8Map 类 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable/swisstable.pyx` |
| 16 | `67c9d4e13b` | feat(swisstable): 添加 SwissComplex64Map 类 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable/swisstable.pyx` |
| 17 | `ad3b462701` | feat(swisstable): 添加 SwissComplex128Map 类 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable/swisstable.pyx` |
| 18 | `0744c0555f` | optimize BaseMultiIndexCodesEngine init | index | B2-index-join | migrated in `index/search primitives` sub-batch | `_libs/index.pyx` |
| 19 | `1e811cf41e` | optimize take_1d | algorithms | B1-existing-audit | already in pandas3 per `28df2030b5` | `_libs/algos_take_helper.pxi.in` |
| 20 | `02d2113331` | optimize take_1d | index/algorithms | B1-existing-audit | skipped as backward step per `28df2030b5`; recheck index part in B2 | `_libs/algos_take_helper.pxi.in`, `_libs/index.pyx` |
| 21 | `df0c75b9fd` | optimize BaseMultiIndexCodesEngine init | index | B2-index-join | migrated in `index/search primitives` sub-batch | `_libs/index.pyx` |
| 22 | `33da51b84c` | optimize take_1d | algorithms | B1-existing-audit | already in pandas3 per `28df2030b5` | `_libs/algos_take_helper.pxi.in` |
| 23 | `66863407cc` | roll_sum 是 rolling 窗口求和的高频路径，核心操作模式为： | low-level/window | B7-low-level-window | pending | `_libs/window/aggregations.pyx` |
| 24 | `2777612697` | kh put likely | low-level/window | B7-low-level-window | migrated in khash micro-optimization sub-batch | `_libs/include/pandas/vendored/klib/khash.h` |
| 25 | `598e709a9b` | group_cummin_max性能优化 | groupby | B1-existing-audit | audit covered by `f39ba34d1d` | `_libs/groupby.pyx` |
| 26 | `e38ed47c0e` | group_all_any 性能优化 | groupby | B1-existing-audit | audit covered by `f39ba34d1d` | `_libs/groupby.pyx` |
| 27 | `d019bc4d8e` | group_cumprod性能优化 | groupby | B1-existing-audit | audit covered by `f39ba34d1d` | `_libs/groupby.pyx` |
| 28 | `63b8520f8c` | group_shift_indexer性能优化 | groupby | B1-existing-audit | audit covered by `f39ba34d1d` | `_libs/groupby.pyx` |
| 29 | `171fe464a6` | take_xxx 消减引用计数增减 | algorithms | B1-existing-audit | audit covered by `28df2030b5` | `_libs/algos.pyx`, `_libs/algos_take_helper.pxi.in` |
| 30 | `9e640a5d2c` | add_overflowsafe: 连续场景直接使用裸指针访问替代原本的PyArray_MultiIter迭代器 | tslibs | B5-tslibs | pending | `_libs/tslibs/np_datetime.pyx` |
| 31 | `a60133b265` | array_equivalent_object: 连续场景直接使用裸指针访问替代原本的PyArray_MultiIter迭代器 | lib | B3-lib-object | pending | `_libs/lib.pyx` |
| 32 | `eaaffa04b4` | is_monotonic优化 | index | B2-index-join | pending; must reconcile with rollback `0b958ce8fd` | `_libs/algos.pyx` |
| 33 | `fefbc7b5a4` | cython construct_1d_object_array_from_listlike impl | lib | B3-lib-object | migrated in object construction batch | `_libs/lib.pyx`, `_libs/lib.pyi`, `core/dtypes/cast.py` |
| 34 | `53b373262d` | cython _searchsorted_left impl | index | B2-index-join | partially migrated: typed searchsorted hooks; DatetimeEngine call-site audit remains | `_libs/index.pyx`, `_libs/index_class_helper.pxi.in` |
| 35 | `82d3f0d7c5` | group_lastx性能优化 | groupby | B1-existing-audit | audit covered by `f39ba34d1d` | `_libs/groupby.pyx` |
| 36 | `91019df988` | skiplist优化 | low-level/window | B7-low-level-window | pending | `_libs/include/pandas/skiplist.h` |
| 37 | `eaa0ec8472` | group_mean 性能优化 | groupby | B1-existing-audit | audit covered by `f39ba34d1d` | `_libs/groupby.pyx` |
| 38 | `c60600ce36` | group_min_max性能优化 | groupby | B1-existing-audit | audit covered by `f39ba34d1d` | `_libs/groupby.pyx` |
| 39 | `829e42a602` | group_prod性能优化 | groupby | B1-existing-audit | audit covered by `f39ba34d1d` | `_libs/groupby.pyx` |
| 40 | `4c4a5096dc` | 优化 kth_smallest_c 算法性能，使用指针操作替代数组索引，减少内存访问开销 | algorithms | B6-algorithms | migrated in algos Cython batch | `_libs/algos.pyx` |
| 41 | `12c293f745` | 默认启用swisstable | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `core/config_init.py` |
| 42 | `8c1c10d14f` | swisstable 同步蓝区代码 | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | hashing/hashtable/swisstable files |
| 43 | `7d2d23951e` | fix debug build | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable.pxd` |
| 44 | `4423bd227e` | factorize: fix int na_value bug | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable_class_helper.pxi.in` |
| 45 | `72efd58130` | fix object engine performance in _get_loc_duplicates | index | B2-index-join | pending | `_libs/index.pyx` |
| 46 | `a985978612` | fix stride case in swisstable | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable.pyx`, `_libs/swisstable_class_helper.pxi.in` |
| 47 | `a3739ab1e4` | 添加携程 benchmark | benchmarks | B9-benchmarks | pending | `asv_bench/benchmarks/xiecheng.py` |
| 48 | `42b376312f` | add dataframe construction bench for xiecheng | benchmarks | B9-benchmarks | pending | `asv_bench/benchmarks/xiecheng.py` |
| 49 | `baca9d6de2` | fix xiecheng StringCategorical time_to_categorical | benchmarks | B9-benchmarks | pending | `asv_bench/benchmarks/xiecheng.py` |
| 50 | `403df2a143` | join: 优化 groupsort_indexer 和 take | join | B2-index-join | partially migrated: `groupsort_indexer` unroll only; join take helper remains | `_libs/algos.pyx`, `_libs/join.pyx` |
| 51 | `56996122c8` | add swisstable Factorizer implementation | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | hashtable/swisstable/merge files |
| 52 | `120b7a79a1` | Fix benchmark script arguments and test script build process | script | B1-existing-audit | do not migrate; reverted by `6c0d804544` | `scripts/benchmark_swiss_prefetch.py`, `scripts/test_prefetch_server.sh` |
| 53 | `6c0d804544` | Revert "Fix benchmark script arguments and test script build process" | script | B1-existing-audit | do not migrate; revert marker only | deletes the scripts from row 52 |
| 54 | `108a06f167` | swisstable get labels batch | swisstable | B1-existing-audit | audit covered by `f6d861f8a6` | `_libs/swisstable*` |
| 55 | `e951e5f721` | 优化 ints_to_pydatetime | tslibs | B5-tslibs | pending | `_libs/tslibs/vectorized.pyx` |
| 56 | `a22b61a327` | 优化 get_date_name_field | tslibs | B5-tslibs | pending | `_libs/tslibs/fields.pyx` |
| 57 | `07ccc6e28e` | faster many:many join with sort=False | join | B2-index-join | pending | `_libs/join.pyx` |
| 58 | `a041e1e5b3` | group_sum优化 | groupby | B1-existing-audit | intentionally not migrated by `f39ba34d1d` pending only if later evidence justifies | `_libs/groupby.pyx` |
| 59 | `fcdd01e0a4` | update xiecheng bench | benchmarks | B9-benchmarks | pending | `asv_bench/benchmarks/xiecheng.py` |
| 60 | `1246018d48` | 性能优化：优化 apply、astype、fillna、take 和 value_counts 核心执行路径 | python-layer | B8-python-layer | pending; split before porting | `lib.pyx/.pyi`, `core/algorithms.py`, apply/putmask/take/arrow/generic/internals/series/tests |
| 61 | `51a2b98159` | perf: Use stable sort in safe_sort for better ARM performance | algorithms | B6-algorithms | migrated in stable-sort batch | `core/algorithms.py` |
| 62 | `31a63abb20` | perf: use stable sort for all argsort calls in factorize | algorithms | B6-algorithms | migrated in stable-sort batch | `core/algorithms.py` |
| 63 | `1c4c300a36` | perf: Add sort kind in safe_sort | algorithms | B6-algorithms | migrated in stable-sort batch | `core/algorithms.py` |
| 64 | `7d7fb8bf02` | PERF: optimize row-wise apply string access and label overhead | python-layer | B8-python-layer | pending | `core/apply.py`, `core/internals/managers.py`, `core/series.py`, tests |
| 65 | `0b958ce8fd` | is_monotonic 携程 MergeJoin用例失败 退回开源版本 | join/index | B2-index-join | pending; this constrains row 32 | `_libs/algos.pyx` |
| 66 | `c064f8cd15` | eq_NA_compat 优化： 避免引用增减 | lib | B3-lib-object | pending | `_libs/lib.pyx` |
| 67 | `defb42e92e` | fix array_equivalent_object build | lib | B3-lib-object | pending | `_libs/lib.pyx` |
| 68 | `286eba1dc7` | groupby_idmin_max 优化 | groupby | B1-existing-audit | audit covered by `f39ba34d1d` if included; otherwise reassess in groupby follow-up | `_libs/groupby.pyx` |
| 69 | `52f9c792ec` | join indexer: object optimize | join | B2-index-join | pending | `_libs/join.pyx` |
| 70 | `1b285b9697` | range _concat impl in lib.pyx | index/lib | B2-index-join | migrated with pandas3 fallback and repeated-range semantics preserved | `_libs/lib.pyx`, `_libs/lib.pyi`, `core/indexes/range.py` |
| 71 | `42dc387d2c` | khash 局部缓存 keys/flags 指针 | low-level/window | B7-low-level-window | migrated in khash micro-optimization sub-batch | `_libs/include/pandas/vendored/klib/khash.h` |
| 72 | `454f5e27f1` | optimize fast_zip | lib | B3-lib-object | pending | `_libs/lib.pyx` |
| 73 | `09c956697b` | quantile 性能优化 | groupby | B1-existing-audit | audit covered by `f39ba34d1d` only if diff confirms; otherwise reassess after B1 | `_libs/groupby.pyx` |
| 74 | `9686250639` | PERF: add early-day fast path and contiguous 1-D direct loop in shift_months(day_opt=None) | tslibs | B5-tslibs | pending | `_libs/tslibs/offsets.pyx`, tests |
| 75 | `15dba60291` | PERF: dense get_dummies helper | reshape | B4-reshape | pending | `_libs/reshape.pyi`, `_libs/reshape.pyx`, `core/reshape/encoding.py`, tests |
| 76 | `8232deb8a0` | PERF: split numeric unstack mask/value writes | reshape | B4-reshape | pending | `_libs/reshape.pyx` |
| 77 | `706991b28f` | PERF: 2D no-limit pad fast path | algorithms | B6-algorithms | already covered in pandas3 current `pad_2d_inplace` | `_libs/algos.pyx` |
| 78 | `d72753640f` | PERF: flat list/tuple object-array construction fast path | lib/cast | B3-lib-object | migrated in object construction batch | `core/dtypes/cast.py`, tests |
| 79 | `3f77b3b691` | PERF: add_overflowsafe left-contiguous/right-scalar paths | tslibs | B5-tslibs | pending | `_libs/tslibs/np_datetime.pyx` |
| 80 | `0e2677777a` | PERF: checknull uses C `isnan()` | algorithms | B6-algorithms | migrated with pandas3 signature adaptation | `_libs/missing.pyx`, tests |
| 81 | `42c76d772f` | PERF: `_shift_bdays` avoids full date rebuild where possible | tslibs | B5-tslibs | pending | `_libs/tslibs/offsets.pyx`, tests |
| 82 | `82081460b3` | PERF: add_overflowsafe scalar/contiguous cached flags | tslibs | B5-tslibs | pending | `_libs/tslibs/np_datetime.pyx`, tests |
| 83 | `d1f4ed4720` | PERF: SemiMonthBegin/SemiMonthEnd narrow date rebuild | tslibs | B5-tslibs | pending | `ccalendar.pyx`, `offsets.pyx`, `core/dtypes/cast.py`, tests |
| 84 | `38f97b58aa` | PERF: add pad_inplace no-limit fast path | algorithms | B6-algorithms | already covered in pandas3 current `pad_inplace` | `_libs/algos.pyx` |
| 85 | `6fd0359851` | PERF: add pad_inplace loop unroll | algorithms | B6-algorithms | already covered in pandas3 current `pad_inplace` | `_libs/algos.pyx` |
| 86 | `844af97539` | PERF: maybe_convert_objects homogeneous-block fast path | lib | B1-existing-audit | audit covered by `8e0304f403` | `_libs/lib.pyx`, tests |
| 87 | `aae84a43d5` | PERF: avoid redundant Python-to-C complex conversion | lib | B1-existing-audit | audit covered by `8e0304f403` | `_libs/lib.pyx` |
| 88 | `eba6d76f8c` | PERF: nancorr high-validity mask fast path | algorithms | B1-existing-audit | audit covered by `d978bd13ee` | `_libs/algos.pyx` |
| 89 | `2e964bd967` | PERF: take_2d_axis1 broader contiguous fast path | algorithms | B1-existing-audit | audit covered by `28df2030b5` | `_libs/algos_take_helper.pxi.in` |

## Known export/replay differences

- Export dirty worktree only includes `asv_bench/asv.conf.json`.
- Reconstructed `../pandas` additionally has staged changes in
  `_libs/reshape.pyx`, `_libs/tslibs/offsets.pyx`, and
  `core/algorithms.py`. These staged changes are treated as replay artifacts
  unless they are proven to match export commit diffs or the local fixup commit.
- Target branch already contains migration commits. Future batches must compare
  against `v3.0.3..HEAD` before adding code to avoid duplicate implementations.

## Commit strategy

- One target commit per completed batch.
- Batch 0/1 documentation commit contains only `MIGRATION_PLAN.md` and
  `MIGRATION_VALIDATION.md`.
- Product-code batches must include adapted tests where behavior can be
  observed, and commit messages must name the representative export hashes and
  ASV limitations.
