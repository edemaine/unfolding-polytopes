# Benchmarks

Run these commands from the repository root after `pnpm install`. All suites work from a fresh checkout: their inputs are either generated from fixed seeds or stored as exact coordinates and numerical settings.

The scripts and `*-cases.json` fixtures belong in version control. All generated measurements, Markdown reports, logs, CPU profiles, and failure dumps go under **`bench/output/`**, which is ignored by Git. Running a benchmark does not overwrite this README or the committed fixtures.

## Fixed examples

```sh
pnpm bench
```

Compares the four intersection methods on facet pairs and identical searches capped at 120 nodes. It asserts agreement with `enumerate` on overlap decisions, search counts, and witness trees. Three rounds rotate method order. Hull construction is excluded from timings.

Writes `output/results.json` and `output/overlap.md`.

## Sampled examples

```sh
pnpm bench:sampled
pnpm bench:sampled-report
```

Uses the 43 portable cases in [sampled-cases.json](sampled-cases.json), covering 15 parameter sets. The selection includes slow and inconclusive examples from earlier runs; their coordinates and settings are embedded, so those runs are not required.

Compares facet-pair decisions and identical searches capped at 40 nodes, twice per method. Rotated-box checks independently test separation, boundary contact, and thin overlap at several tolerances. A limited search is a timing workload, not a claim that an example has no unfolding.

Writes `output/sampled-results.json`, `output/sampled.md`, and any overlap disagreements under `output/sampled-failures/`.

To select different examples from your local `results/` directory:

```sh
pnpm bench:select-sampled
pnpm bench:sampled
pnpm bench:sampled-report
```

The selector updates `sampled-cases.json` directly with self-contained inputs. Commit that change when updating the suite. Useful failure cases should become regression tests.

## Completed searches and profiling

```sh
pnpm bench:completed
pnpm bench:completed-report
```

Uses the ten successful examples in [completed-cases.json](completed-cases.json). Each is searched to completion three times with `enumerate` and `dual-lp`, without node or time cutoffs. Search counts and witness trees must match. Hull construction is timed separately.

Two CPU profiles measure repeated full searches with precomputed hulls and repeated hull construction plus search. Startup and input-file reads are outside the profiles. The profiler runs for at least six seconds per workload.

Writes `output/completed-results.json`, `output/completed.md`, and `output/completed-{search,pipeline}.cpuprofile`. Open the profiles in Chrome DevTools to inspect call trees.

## Hull construction

```sh
pnpm bench:hull
```

Compares incremental hull construction with enumeration on all 43 sampled and ten completed fixtures, plus cubes and cross polytopes in dimensions 4 and 5. Three repetitions alternate method order. Hull timings exclude input loading and correctness checks. Each case must have identical facet/ridge incidence, matching supporting planes, and identical search counts and witness trees with an 80-node cap. Records any enumeration fallbacks separately.

Writes `output/hull-results.json` with all timings and candidate-plane work counts.

## Comparing revisions

```sh
pnpm bench:compare path/to/reference/src
```

Loads a reference source tree alongside the current code. Both use incremental hulls and `dual-lp`. Compares exact hull data, search counts, witness trees, and overlap counters on the sampled and completed fixtures, plus 4D/5D cubes and cross polytopes. Completed fixtures run without cutoffs; other searches have an 80-node cap. Hulls and completed searches are timed separately over five repetitions after warmup, alternating version order.

Writes `output/optimization-results.json`. The reference must support the current hull and solver APIs. For the optimization comparison below, prepare the reference with these Bash commands:

```sh
mkdir -p bench/output/reference
git archive eb10196 src | tar -x -C bench/output/reference
pnpm bench:compare bench/output/reference/src
```

## Results

Measurements from September 10, 2026, using Node v24.11.0. Search times include placement, pruning, and intersection tests, but exclude hull construction and startup. Each table compares methods on the same inputs within one benchmark run. Times are in **milliseconds** unless stated otherwise; short timings are sensitive to machine load.

### Fixed examples

Median search time over three repetitions, with a 120-node cap. Searches stop earlier when an unfolding is found.

| d | Points | Seed / family | `enumerate` | `primal-lp` | `gjk-lp` | `dual-lp` (default) |
|---|---|---|---:|---:|---:|---:|
| 3 | 8 | 1 | 0.68 | 0.45 | 0.51 | 0.49 |
| 4 | 10 | 42 | 11.71 | 3.13 | 4.40 | 2.55 |
| 5 | 10 | 42 | 149.27 | 13.04 | 27.77 | 8.71 |
| 6 | 12 | 196 | 6,774.89 | 172.78 | 522.18 | 116.83 |
| 4 | 16 | cube | 0.20 | 0.21 | 0.21 | 0.20 |

### Sampled examples

Median search time across the selected seeds and two repetitions per method, with a 40-node cap. The 43 cases cover all 15 parameter sets below.

| d | Points | Cases | `enumerate` | `primal-lp` | `gjk-lp` | `dual-lp` (default) |
|---|---|---:|---:|---:|---:|---:|
| 2 | 3 | 1 | 0.30 | 0.05 | 0.04 | 0.03 |
| 4 | 5 | 3 | 0.66 | 0.28 | 0.36 | 0.20 |
| 4 | 6 | 3 | 1.49 | 0.37 | 0.60 | 0.27 |
| 4 | 7 | 3 | 2.75 | 0.68 | 1.26 | 0.56 |
| 4 | 8 | 3 | 4.84 | 1.14 | 2.17 | 1.29 |
| 4 | 9 | 3 | 9.96 | 3.28 | 4.68 | 1.94 |
| 4 | 10 | 3 | 10.79 | 2.64 | 4.25 | 2.30 |
| 4 | 11 | 3 | 18.61 | 3.68 | 5.19 | 2.83 |
| 5 | 6 | 3 | 4.11 | 0.52 | 0.92 | 0.41 |
| 5 | 7 | 3 | 24.07 | 2.34 | 3.69 | 1.50 |
| 5 | 9 | 3 | 139.98 | 11.07 | 20.16 | 7.81 |
| 5 | 20 | 3 | 218.20 | 19.02 | 37.98 | 14.97 |
| 6 | 12 | 3 | 1,020.77 | 23.36 | 70.81 | 17.61 |
| 6 | 15 | 3 | 950.59 | 24.19 | 74.72 | 15.65 |
| 6 | 20 | 3 | 1,398.50 | 37.81 | 119.78 | 32.27 |

Total time across the sampled suite, including both repetitions, in **seconds**:

| Method | Facet-pair tests | Searches | Search enumeration fallbacks |
|---|---:|---:|---:|
| `enumerate` | 7.11 | 22.66 | — |
| `primal-lp` | 0.34 | 0.81 | 0 |
| `gjk-lp` | 0.66 | 2.11 | 0 |
| `dual-lp` | 0.21 | 0.69 | 0 |

### Completed searches

Full searches without cutoffs, comparing enumeration with the default backend. Search times are medians of three repetitions. Hull construction is a separate single measurement.

| d | Points | Seed | Search nodes | `enumerate` | `dual-lp` | Speedup | Hull construction |
|---|---|---|---:|---:|---:|---:|---:|
| 4 | 11 | 10131 | 1,129 | 311.41 | 58.01 | 5.4× | 6.45 |
| 4 | 11 | 14600 | 398 | 194.38 | 41.77 | 4.7× | 4.69 |
| 5 | 9 | 70353 | 24 | 57.22 | 3.71 | 15.4× | 2.72 |
| 5 | 9 | 71440 | 372 | 1,396.51 | 89.91 | 15.5× | 2.68 |
| 5 | 20 | 384 | 164 | 1,488.80 | 91.21 | 16.3× | 237.59 |
| 5 | 20 | 334 | 205 | 2,238.37 | 131.61 | 17.0× | 232.05 |
| 6 | 12 | 304 | 74 | 4,763.00 | 54.58 | 87.3× | 19.73 |
| 6 | 12 | 236 | 107 | 5,277.18 | 88.55 | 59.6× | 21.82 |
| 6 | 15 | 13 | 171 | 12,471.86 | 190.27 | 65.5× | 109.40 |
| 6 | 15 | 0 | 195 | 15,252.57 | 257.05 | 59.3× | 102.45 |
| **Sum of search medians** | | | | **43,451.32** | **1,006.66** | **43.2×** | |

### Correctness checks

| Suite | Checks | Result |
|---|---|---|
| Fixed examples | Facet-pair decisions, search counts, and witness trees across all four methods | All agree with enumeration |
| Sampled examples | 54,318 pair comparisons and 258 bounded-search comparisons | 0 disagreements |
| Analytic boxes (included in sampled comparisons) | 360 checks in unfolded dimensions 1–5: separation, exact contact, and intersections below/above the tolerance threshold | All agree with the analytic answer |
| Completed searches | 60 full searches; compare node/prune/solution counts and witness trees | All agree |

Counts include repetitions. Agreement between implementations checks regressions; the analytic boxes provide a separate geometric check. Shared code and fallback behavior mean agreement alone is not a certificate of exact correctness.

### CPU profiles of `dual-lp`

Share of CPU samples while repeatedly solving the ten completed examples. Each profile runs for at least six seconds after setup. Search-only measurements reuse constructed hulls; the combined profile rebuilds them. Phase shares include time in called functions.

| Phase | Search only | Hull construction + search |
|---|---:|---:|
| Hull construction | — | 49.6% |
| Overlap checking | 86.0% | 41.8% |
| Other tree search | 7.9% | 3.3% |
| Facet placement | 3.5% | 2.3% |
| Connectivity pruning | 1.6% | 0.7% |
| Garbage collection | 0.8% | 1.5% |
| Other / runtime | 0.3% | 0.7% |

Overlap checking remains the main search cost. Hull construction becomes comparable to search once the faster intersection backend is used. Garbage-collection samples are shown separately because the profile does not identify which phase allocated the collected objects.

### Hull construction times

Hull-only measurements, using the median of three repetitions for each case, then summing within each row. The 57 cases all matched geometry and bounded searches; none used the enumeration fallback. Duplicate cases shared by the two fixture lists are included twice. The earlier pipeline profiles above used enumeration for hull construction.

| d | Points | Family | Cases | `enumerate` | `incremental` (default) | Speedup |
| --- | --- | --- | ---: | ---: | ---: | ---: |
| 2 | 3 | random | 1 | 0.09 | 0.16 | 0.54x |
| 4 | 5 | random | 3 | 0.37 | 0.80 | 0.46x |
| 4 | 6 | random | 3 | 1.38 | 1.81 | 0.76x |
| 4 | 7 | random | 3 | 1.75 | 2.21 | 0.79x |
| 4 | 8 | cross | 1 | 1.35 | 1.42 | 0.95x |
| 4 | 8 | random | 3 | 2.80 | 2.41 | 1.16x |
| 4 | 9 | random | 3 | 4.02 | 3.23 | 1.24x |
| 4 | 10 | random | 3 | 6.08 | 3.89 | 1.56x |
| 4 | 11 | random | 5 | 19.20 | 9.89 | 1.94x |
| 4 | 16 | cube | 1 | 33.08 | 3.39 | 9.77x |
| 5 | 6 | random | 3 | 0.75 | 1.66 | 0.45x |
| 5 | 7 | random | 3 | 1.47 | 2.33 | 0.63x |
| 5 | 9 | random | 5 | 14.86 | 12.04 | 1.23x |
| 5 | 10 | cross | 1 | 6.06 | 4.29 | 1.41x |
| 5 | 20 | random | 5 | 1533.17 | 125.59 | 12.21x |
| 5 | 32 | cube | 1 | 5687.98 | 30.80 | 184.67x |
| 6 | 12 | random | 5 | 191.81 | 89.91 | 2.13x |
| 6 | 15 | random | 5 | 969.48 | 190.02 | 5.10x |
| 6 | 20 | random | 3 | 4305.07 | 271.41 | 15.86x |
| **Total** | | | **57** | **12780.77** | **757.27** | **16.88x** |

These measurements used the initial incremental implementation. It benefits larger point sets most; its bookkeeping can cost more on tiny hulls. The following comparison measures the additional optimizations.

### LP buffers, hull linear algebra, and ridge adjacency

Comparison with revision `eb10196`, which already uses incremental hulls and `dual-lp`. The optimized version reuses LP tableau buffers, fills halfspace tableaux directly, streamlines pivots and orthogonalization, retains simplex planes, and derives ridge candidates from simplex adjacency. Times are sums of per-case medians over five repetitions, in milliseconds.

| Workload | Cases | Before | After | Speedup |
| --- | ---: | ---: | ---: | ---: |
| Hull construction, all fixtures and added families | 57 | 793.67 | 227.78 | 3.48x |
| Hull construction, completed fixtures | 10 | 198.46 | 58.50 | 3.39x |
| Full search, completed fixtures | 10 | 1067.02 | 601.78 | 1.77x |
| Hull + search, sum of the preceding two rows | 10 | 1265.48 | 660.28 | 1.92x |

All 57 hulls matched the reference exactly, including facet/ridge IDs and numerical geometry. Search counts, witness trees, and overlap counters also matched. The separate sampled suite passed 54,318 overlap comparisons and 258 search comparisons across all four intersection methods, including analytic contact and thin-overlap cases. These checks establish regression agreement, not exact geometric certification.
