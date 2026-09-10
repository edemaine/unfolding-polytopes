# Unfolding Polytopes

A famous open problem (dating back to Dürer 1525) is whether every convex polyhedron in 3D can be unfolded into a single non-overlapping polygon in the plane by cutting along its edges. See e.g. *Geometric Folding Algorithms* [the book](https://gfalop.org) or [the class](https://courses.csail.mit.edu/6.849/).

What about higher dimensions? Can every convex polytope in $d$ dimensions be unfolded into a single non-overlapping polyhedron in $d-1$ dimensions by cutting along its ridges ($(d-2)$-dimensional faces)? Stated another way, does every $d$-dimensional convex polytope have a spanning tree of its facets ($(d-1)$-dimensional faces) that, when developed into $d-1$ dimensions, do not overlap? This is the question explored by this project.

## Running

The implementation is Civet, currently using TypeScript syntax with the `civet esCompat` directive. It runs directly; no build step is required. Use Node.js 22+ and pnpm.

```sh
pnpm install
pnpm test
pnpm check

# Solve a tesseract (its eight cubical facets unfold into 3D).
pnpm solve --family cube --dimension 4 --output tesseract.json

# Generate a reproducible point set and solve its convex hull.
pnpm generate --dimension 4 --points 8 --seed 42 --output points.json
pnpm solve points.json --max-nodes 1000000 --timeout-ms 60000 --output unfolding.json

# Search 100 random convex hulls in 5D, starting at seed 0.
pnpm run search --dimension 5 --points 9 --trials 100 --seed 0

# Search thin Gaussian polytopes.
pnpm run search --dimension 4 --points 8 --distribution gaussian --scales 1,1,1,0.05

# Equivalent direct invocation, including help.
pnpm exec civet src/cli.civet --help
pnpm exec civet src/cli.civet solve points.json
```

Input JSON is either an array of points `[[x0, x1, ...], ...]` or an object with a `points` field. Supply coordinates in the polytope's intrinsic dimension $d\ge2$: lower-dimensional inputs in a larger ambient space are rejected. Interior, duplicate, and redundant boundary points are allowed. No facet triangulation or precomputed incidence structure is required.

`solve` stops at the first nonoverlapping tree by default. `--all` counts every successful tree, and `--no-prune` delays overlap testing until a complete tree is built. For example, `pnpm solve --family cube --dimension 3 --all --no-prune` visits all 384 labeled spanning trees. The output includes the first successful tree, developed facet coordinates, tolerances, and search statistics.

## Results and counterexample searches

```sh
pnpm summary                         # All runs under results, recursively
pnpm summary results/run-XXXXXX      # One run (or any results subdirectory)
pnpm summary results --json --output report.json
```

`summary` sorts runs numerically by dimension, then point count, with unknown values last and directory names breaking ties. It adds a subtotal when multiple runs share these parameters and overall totals at the bottom. The table includes saved/planned trials, dimension and point count when recorded in run metadata, and worker state. Counts describe **saved attempts**: retries count separately from their source trials. It reads journals for speed, falls back to trial files for missing journal entries, and counts malformed trials separately as unreadable. It does not rely on potentially stale `summary.json` totals. The report is a snapshot during a live search; unrecorded work is not counted. `--json` provides sorted `runs`, `groups` with subtotals, overall `totals`, and warnings; `--output` saves the selected format to a file.

New runs save `worker: {pid, hostname, startedAt}` in `run.json` before processing trials. Summary reports `finished` when `summary.json` exists. Otherwise, it probes the saved PID on the same host: `running (PID ...)` if the process exists, `interrupted` if it has exited, or `unknown` for older records, another host, invalid metadata, or a failed permission check. JSON reports expose `state` and `pid` for each run. This is a best-effort process-existence check: it cannot detect a hung worker or distinguish a reused PID from the original worker. The saved timestamp is informational, not a verified process identity.

| Status | Meaning |
| --- | --- |
| `found` | A tree passed every interior-overlap test at the specified tolerance. |
| `no-unfolding` | The search exhausted all trees, including branches excluded by overlap pruning. This is a **numerical counterexample candidate**. |
| `inconclusive` | A node or time limit stopped the search before it found an unfolding. |

`termination` records `solution`, `exhausted`, `maxNodes`, or `timeout`. `exhaustive` is true only for `exhausted`. With `--all`, a limited run can still report `found` if it has already found a witness; its solution count is then only a lower bound. `completeTrees` counts actual leaves visited, not the trees eliminated by partial-overlap pruning.

`search` uses successive seeds and writes every trial to a fresh `results/run-*` directory (`--output` changes the parent directory). Without `--seed`, it scans that parent recursively for matching random generation parameters (dimension, point count, distribution, and axis scales) and starts after the largest **planned** seed range. Using planned ranges prevents overlap with jobs still running; interrupted ranges can be completed with `retry`. The first matching run starts at 0. An explicit `--seed` permits intentional reuse; `generate` and `solve` still default to 0. The allocated range is printed at startup and saved in `run.json` before trials begin. Searches using the same output parent coordinate allocation through a short-lived directory lock. Automatic allocation refuses to wrap the PRNG's 32-bit seed range.

Each trial records the original points, generator parameters, hull and solver options, facet/ridge indices, status, statistics, and any unfolding. `trials.jsonl` provides one summary per trial; `summary.json` contains totals. Geometry failures are saved as errors, separate from candidates and cutoffs. Interrupting a batch preserves completed trials.

Every trial's console JSON includes its absolute `outputFile` path and the worker's `pid`, including failed trials. The final totals JSON names `summary.json`. The startup message also prints the PID, so a trial that has not finished yet can be associated with its output directory. Resume that directory with `pnpm retry`. On Windows, `Get-Process -Id PID` in PowerShell also checks whether that process still exists.

Resume unfinished runs and rerun inconclusive examples:

```sh
pnpm retry results/run-XXXXXX
pnpm retry results/run-XXXXXX results/run-YYYYYY
# Optional new limits or tolerance:
pnpm retry results/run-XXXXXX --timeout-ms 60000 --overlap-tolerance 1e-10
```

`retry` compares each run's planned trials with its saved `trial-*.json` filenames and uses the compact `trials.jsonl` journal to select inconclusive examples. It reads full trial JSON only for selected examples or files missing from the journal, avoiding large coordinate-file reads for indexed completed trials. Missing or truncated journals fall back to those trial files; stale aggregate `summary.json` totals are not used for selection. For each input directory it first fills missing seeds (including gaps), then reruns records with `result.status: "inconclusive"`. Found unfoldings, candidates, and recorded errors are already attempted and are skipped. Missing random trials are regenerated from the saved generator settings; existing trials reuse their exact points. Tolerances, root, and search settings are preserved, with **all computational limits reset to Infinity** unless explicitly supplied. Solver/geometry flags override saved settings.

It writes one fresh `retry-*` directory under the first input directory's parent; `--output` changes the parent directory. A single input preserves trial filenames; multiple inputs use new sequential filenames to avoid collisions. Each record has a `sourceFile` path to its original trial (which may be absent for a missing seed). Repeated directory arguments are processed once. Original runs are untouched. The new run saves a task manifest, so if it is interrupted, pass **that retry directory** to `retry` to continue its remaining tasks. Older retry runs are also supported using their saved source-file lists. To avoid duplicating work, stop a source job before resuming it; selection is a snapshot, not a live handoff. If there is no missing or inconclusive work, no directory is created. Malformed trial JSON encountered during these reads is reported as an error; indexed completed files are not inspected.

To investigate a candidate or a cutoff, pass its saved trial JSON to `solve`. This reuses the **points**; specify the desired tolerances and budgets again:

```sh
pnpm solve results/run-XXXXXX/trial-000000-seed-0.json --tolerance 1e-11 --overlap-tolerance 1e-10 --max-nodes Infinity --timeout-ms Infinity --output recheck.json
```

The implementation uses floating-point geometry, not exact predicates or certified arithmetic. Points are translated and uniformly scaled to radius 1 before computation. The default hull tolerance is `1e-9`; overlap requires a common interior point more than `16*tolerance` from every facet boundary. Boundary contact is permitted, and overlaps thinner than this threshold can be missed. Near-degenerate hulls can have incorrect incidence or be rejected; linear systems with pivots at most `1e-13` are skipped. Both positive and negative results need independent, higher-precision validation before mathematical use. Rerunning at different tolerances is a diagnostic, not a certificate.

## Algorithm

1. **Convex hull:** incrementally insert points into a triangulated boundary, then merge coplanar pieces into true facets. Two facets are adjacent exactly when their intersection has affine dimension $d-2$. See [Hull construction](#hull-construction).
2. **Tree search:** fix one root facet and branch on an undecided ridge leaving the connected set of placed facets. Either attach its unplaced neighbor or forbid that ridge. Internal edges cannot be selected because they would create cycles. Every spanning tree determines exactly one sequence of these choices, regardless of branch order. A fixed root loses no unfoldings: changing the root changes only the global Euclidean placement.
3. **Development:** compute an orthonormal basis along the shared ridge and a perpendicular inward direction in each incident facet. Preserve the ridge pointwise and map the child's inward direction to the negative of the parent's inward direction. This gives an affine isometry into $\mathbb R^{d-1}$ without dimension-specific rotation formulas.
4. **Intersection:** test whether each newly placed facet overlaps the interior of an already placed facet in $\mathbb R^{d-1}$. Skip its direct hinge parent: convexity places their interiors on opposite sides of the hinge. Originally adjacent facets whose shared ridge is cut still need testing. The default solves a linear program for the common interior margin. Boundary contact is allowed; see [Intersection algorithms](#intersection-algorithms) for the available methods and their tolerance checks.
5. **Pruning and propagation:** discard overlapping attachments and disconnected branches. Contract the placed facets into one graph vertex, retaining parallel edges. A bridge leaving that vertex must belong to every completion, so attach it without an exclusion branch. Forward checking tests every frontier attachment against the placed facets and forbids overlapping attachments throughout the current branch. Existing placements cannot change in descendants, so these exclusions remain valid. Recheck connectivity and bridges after these exclusions.

The search caches candidate placements and the prefix of placed facets they have passed. Descendants check only newly placed facets; sibling branches retain their own checked prefixes. Cache entries belong to the active recursion stack and disappear on backtracking, so memory does not grow with the number of previously searched trees.

There is no dimension-specific upper bound. Hull construction can produce exponentially many faces, and spanning-tree counts grow exponentially. The default intersection method uses the simplex algorithm; its enumeration fallback examines up to $\binom md$ linear systems for $m$ combined bounding halfspaces. Start with few vertices and inspect facet counts. Higher dimension can make the computation much harder even if counterexamples are easier to find.

All computational limits default to `Infinity` for both `solve` and `search`. Set `--max-hull-combinations`, `--max-nodes`, or `--timeout-ms` to impose a limit. Node and time limits apply **per polytope** to tree search, excluding hull construction. A hull cutoff is an error, never an exhausted unfolding search. Time checks are cooperative, including during intersection enumeration.

### Search options

| Option | Behavior |
| --- | --- |
| `--forward-check auto` (default) | Start checking all frontier attachments after visiting more than twice as many nodes as facets. Easy first descents avoid this extra work. |
| `--forward-check on` / `off` | Always check the frontier / check only the chosen attachment. |
| `--no-force-bridges` | Disable forced bridge attachments; connectivity pruning remains enabled. |
| `--branch-order constrained` (default) | Prefer an unplaced facet with the fewest remaining incident ridges. Break ties by ridge ID. |
| `--branch-order index` | Choose the first eligible ridge by ID. |
| `--branch-order bottleneck` | Prefer the ridge whose removal gives the longest shortest alternative route to its unplaced facet. Break ties by degree, then ridge ID. This requires more graph searches. |

Every order tries attachment before exclusion. `--no-prune` also disables forward checking and checks all pairs at complete trees, including hinge pairs. Bridge propagation still preserves exhaustive tree counts. The API equivalents are `forwardCheck: 'auto' | true | false`, `forceBridges: boolean`, and `branchOrder`. Runs save these settings, and retries preserve them unless overridden.

Results include the effective `searchOptions` and `searchStats`: placement constructions and cache reuses, previously checked pairs reused, direct-parent checks skipped, forward-check exclusions, forced attachments, and graph checks. Search-node and prune counts can change with these options; compare exhaustive solution counts when checking completeness.

## Hull construction

**`incremental` is the default** in the CLI and API. Hull construction runs once per polytope, before unfolding search.

| Method | Algorithm |
| --- | --- |
| `incremental` (default) | Maintain a triangulated boundary while inserting points; merge coplanar pieces into true facets afterward. |
| `enumerate` | Test every $\binom nd$ subset of input points for a supporting hyperplane. Also serves as the reference and numerical fallback. |

The incremental method uses [beneath-beyond construction](https://qhull.org/html/qh-eg.htm). It chooses an affinely independent initial simplex, then inserts the remaining points in input order. For each outside point, it removes the visible boundary simplices and cones their horizon to the new point. A fixed interior point determines outward orientation. Interior, duplicate, and boundary points require no insertion unless they extend the hull.

The triangulation is only an intermediate representation: coplanar pieces merge using their full sets of input point indices, preserving nonsimplicial facets and redundant boundary points. Incremental construction reuses the simplex planes and maps simplex adjacency to true facet adjacency, discarding internal boundaries and deduplicating facet pairs. It constructs ridges only for these pairs; enumeration checks all facet pairs. Both methods preserve the same facet and ridge ordering.

Detected numerical inconsistencies in incremental construction trigger enumeration; the result records the actual `hullMethod` and optional `hullFallback` reason. Both methods use floating-point tolerances and can fail on near-degenerate inputs.

Select a method with `--hull-method incremental|enumerate` on `solve`, `search`, or `retry`, or `convexHull(points, {method: 'incremental'})`. Saved hull options record the requested method. Retries preserve it unless overridden; older records without a method use the current default.

`--max-hull-combinations` limits candidate-plane work, with `Infinity` as the default. Enumeration charges one unit per input subset. Incremental construction charges one per created boundary simplex and one per final simplex checked for merging. A fallback uses only the remaining budget. The total is recorded as `hullCombinations`; it does not count visibility scans or ridge construction.

## Intersection algorithms

The intersection test asks whether two developed facets overlap in their **interiors** in $\mathbb R^{d-1}$. Shared boundaries are allowed. **`dual-lp` is the default** in the CLI and API.

| Method | Representation | Algorithm |
| --- | --- | --- |
| `dual-lp` (default) | Halfspaces | Maximize the common interior margin using the custom two-phase simplex solver. |
| `enumerate` | Halfspaces | Solve the same LP by enumerating sets of active constraints. Also serves as the reference and numerical fallback. |
| `primal-lp` | Vertices | Find a common point expressed as strictly positive convex combinations of both vertex sets. |
| `gjk-lp` | Vertices | Try GJK separation first, then resolve remaining pairs with the vertex LP. |

Select a method with `--overlap-method` on `solve`, `search`, or `retry`, or with the API's `overlapMethod` option:

```sh
pnpm solve points.json --overlap-method enumerate
pnpm retry results/run-XXXXXX --overlap-method dual-lp
```

```ts
const result = solve(polytope, {overlapMethod: 'dual-lp'});
```

Saved trials and job manifests record the method. Retries preserve a saved method unless overridden; records without one use the default. Results include the chosen `overlapMethod` and `overlapStats` counters for calls, accepted witnesses, separation bounds, and enumeration fallbacks.

All four methods first try bounding-box and facet-center shortcuts. They retain both vertex and halfspace data, and can detect intersections even when neither facet contains a vertex of the other. Numerical failures or inconclusive internal iterations use the enumeration fallback; only node or time limits can produce an `inconclusive` search status. These are floating-point decisions, not exact certificates.

The LP methods share a custom two-phase simplex solver with reusable tableau buffers and Bland's pivot rule. The halfspace method fills the tableau directly. Buffer reuse also supports nested calls and interrupted searches; returned solutions own their data.

### Halfspace methods: `dual-lp` and `enumerate`

Each facet is represented by inequalities $n_i\cdot x\le b_i$, with unit outward normals. Combining the inequalities of both facets gives the LP

$$
\max t \quad\text{subject to}\quad n_i\cdot x+t\le b_i\quad\text{for every }i.
$$

A positive optimum means interior overlap. The implementation requires a margin greater than `--overlap-tolerance`, which defaults to 16 times the hull tolerance in normalized coordinates.

`dual-lp` solves this LP with the custom two-phase simplex solver, splitting the unrestricted coordinates and margin into positive and negative parts. It checks an overlap witness against the original inequalities. To rule out overlap, it checks an upper bound on the margin from nonnegative LP dual multipliers, accounting for residual normal error using the bounding box. An ambiguous result falls back to `enumerate`. The method name refers to the halfspace representation.

`enumerate` chooses every set of $d$ active constraints in the $d$ variables $(x,t)$, solves the resulting linear system, and checks the candidate point against every inequality. It stops when it finds a point with sufficient margin. This is simple but expensive: up to $\binom md$ systems for $m$ combined inequalities.

### Vertex methods: `primal-lp` and `gjk-lp`

For matrices $V$ and $W$ whose columns are the two facets' vertices, `primal-lp` maximizes $\epsilon$ subject to

$$
V\alpha=W\beta,\qquad \sum_i\alpha_i=\sum_j\beta_j=1,\qquad
\alpha_i\ge\epsilon,\quad\beta_j\ge\epsilon.
$$

Strictly positive weights characterize interior intersection for these full-dimensional facets, including nonsimplicial facets. However, the weight margin is not a geometric distance: the common point must also pass the halfspace margin check above. The LP's dual multipliers can yield a separating direction, checked directly against the vertices. If necessary, GJK searches for another separation bound before falling back to enumeration.

`gjk-lp` reverses that order: GJK first searches for a separating direction using support queries on the Minkowski difference $A-B$. Its nearest-simplex calculation works in arbitrary dimension by enumerating simplex faces. Zero distance alone does not distinguish touching from interior overlap, so unresolved pairs go to the same vertex LP and geometric checks.


## Generators and API

Import from `src/index.civet`:

```ts
"civet esCompat";
import {convexHull, randomPoints, solve, developTree} from './src/index.civet';

const points = randomPoints(5, 9, 42, 'sphere');
const polytope = convexHull(points);
const result = solve(polytope, {maxNodes: 1000000, timeoutMs: 60000});
console.log(result.status, result.stats);
if (result.tree) {
  const facets = developTree(polytope, result.tree);
  console.log(facets.map(f => f.coordinates));
}
```

| Function | Point set |
| --- | --- |
| `randomPoints(d, n, seed, distribution)` | Uniform sphere, uniform ball, standard Gaussian, or uniform box; default sphere. |
| `simplex(d)` | Regular simplex with $d+1$ vertices. |
| `hypercube(d)` | $\{-1,1\}^d$. |
| `crossPolytope(d)` | $\{\pm e_i\}$. |
| `cyclic(d, n)` | Equally spaced parameters on the moment curve $(t,t^2,\ldots,t^d)$, $t\in[-1,1]$. |
| `hypersimplex(d, weight)` | Fixed-weight binary vectors in $d+1$ coordinates, projected isometrically into $d$ dimensions. |
| `product(a, b)` | Cartesian product of two point sets. |
| `prism(points, height)` | Product with an interval; default height 2. |
| `pyramid(points, height)` | Base in an extra coordinate and an apex above its point average; default height 1. |

The CLI supports the first six families, with `cube` and `cross` as family names. `--scales` applies axis factors before the convex hull. Random generation uses Mulberry32 with the seed reduced modulo $2^{32}$; the full point set is saved so replay does not depend on the PRNG.

Facet and ridge IDs are zero-based array indices. Facet vertex indices reference the original input array; facet vertices are **not cyclically ordered**. A returned `tree` is a list of uncut ridge IDs. Cut ridges may give multiple developed copies of the same input vertex, so each placement stores its own `vertices` and `coordinates` arrays. All API and saved placement coordinates use normalized units; multiply them by `polytope.scale` to restore original lengths. Placement maps act on `(inputPoint - polytope.center) / polytope.scale`.

## Timing distributions

```sh
pnpm timings                         # Group all saved attempts by generation parameters
pnpm timings results/retry-XXXXXX     # One run or any results subdirectory
pnpm timings --by-run                 # Keep individual runs separate
pnpm timings --include-timeouts       # Show timed-out attempts too
pnpm timings --nodes                  # Node-count distributions instead of times
pnpm timings --json --output timings.json
```

The table reports **N, minimum, median, P90, P99, P99.9, maximum**, and cumulative counts **over one second, minute, hour, and day**. A two-hour search contributes to the first three tail columns. Thresholds are strict: exactly one minute counts as over a second, but not over a minute. P99.9 is the **99.9th percentile**. Percentiles use nearest rank, so high percentiles can equal the maximum in small groups. JSON durations are in milliseconds; the table selects readable units.

Groups distinguish dimension, point count, generator family, distribution, axis scales, and hypersimplex weight when applicable. Seeds are pooled. Groups sort by dimension and point count; overall totals by outcome appear at the bottom with repeated headers. Retry parameters come from saved task settings or the source run's metadata when available; missing parameters display as `?`. Retries count as separate attempts. Use `--by-run` to compare individual jobs, especially when they used different algorithms, tolerances, or machines.

Times measure **tree search only**, excluding hull construction and file writes. Successful searches and exhausted candidates are separate outcomes. Timed-out attempts are omitted by default from both group rows and overall timing totals; `--include-timeouts` shows them. Node cutoffs remain visible. Cutoffs have separate rows, including limited runs that already found a witness: their recorded times do not measure an exhaustive search. Active trials have no saved timing yet. There is no combined percentile mixing cutoffs with completed searches.

`--nodes` reports the same group and overall percentiles for **visited search nodes**, with cumulative counts over **1,000, 1 million, and 1 billion nodes**. Counts remain exact integers in the table, with comma separators. JSON identifies `metric: "nodes"` and uses fields such as `minNodes`, `p999Nodes`, and `maxNodes`; time reports identify `metric: "time"` and retain their millisecond fields. Timeout filtering and `--by-run` apply to either metric. Node counts can be present even when elapsed times are missing.

The command shares `summary`'s reader: it parses compact journals, keeps the last row for each trial filename, ignores entries without a saved trial file, and reads individual trial JSON only for missing or invalid journal entries. Missing/invalid measurements and unreadable records are counted separately. Existing files with valid journal rows are not opened. The report is a snapshot while jobs are running.

## Tests and benchmarks

The checks below test **the implementation of hull construction, facet placement, overlap detection, and tree search**. They do not prove the unfolding conjecture or certify a numerical counterexample.

`pnpm test` checks:

- **Hull and development geometry:** agreement between hull methods on facet/ridge incidence and bounded searches, numerical fallback budgets, known facet/ridge counts, nonsimplicial facets, hinge continuity, and preservation of distances in dimensions 2–6.
- **Overlap decisions:** containment, crossing intersections, boundary contact, and thin overlaps; agreement with an independent planar separating-axis test and with the enumeration method.
- **Search completeness and pruning:** known spanning-tree counts and agreement between pruned searches, searches that test only complete trees, and independent small-case enumeration.
- **Saved jobs:** reproducible generators, CLI options, limits, summaries, and retry/resume behavior.

`pnpm check` type-checks the source, tests, and benchmark scripts without running them. TypeScript is pinned to the 5.x compiler API used by Civet; casing enforcement is disabled to accommodate Civet's virtual paths on Windows.

The benchmarks measure speed while checking that changing hull or intersection methods preserves results:

| Command | Correctness checks and timing workload | Report |
| --- | --- | --- |
| `pnpm bench:hull` | Compare hull incidence, supporting planes, and searches with an 80-node cap; time hull construction alone. | [Hull construction](bench/README.md#hull-construction) |
| `pnpm bench:compare path/to/reference/src` | Compare exact hull data and search results against another revision; time hulls and completed searches separately. | [Comparing revisions](bench/README.md#comparing-revisions) |
| `pnpm bench` | Compare overlap decisions against `enumerate` on sampled facet pairs, then compare search counts and witness trees with a 120-node cap. | [Fixed examples](bench/README.md) |
| `pnpm bench:sampled` | Sample parameters and seeds from saved runs; compare pair decisions and searches with a 40-node cap. Also check rotated boxes against analytic thresholds for separation, contact, and thin overlap. | [Sampled examples](bench/README.md#sampled-examples) |
| `pnpm bench:completed` | Rerun saved successful examples to completion, comparing search counts and witness trees. Time full searches and profile search alone and hull construction plus search. | [Completed searches and CPU profiles](bench/README.md#completed-searches-and-profiling) |
| `pnpm bench:search` | Compare forward checking, forced bridges, and branch orders on completed fixtures and difficult 4D examples. Check every distinct witness with intersection enumeration. | [Search heuristics](bench/README.md#search-heuristics) |

The benchmark case lists contain exact coordinates and numerical settings, so the suites run without local trial files. Generated measurements and reports go under the ignored `bench/output/` directory. `pnpm bench:select-sampled` updates the committed sampled case list from local runs; see [benchmark instructions](bench/README.md) for selecting inputs and generating reports.

Agreement between methods is a regression check, not an independent proof: they share geometry code and may use the same fallback. The analytic and planar tests provide separate checks on overlap decisions. Benchmark timings depend on the selected examples and machine load; see each report for its workload and measurement details.
