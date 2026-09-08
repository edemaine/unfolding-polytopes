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

`summary` prints overall counts and a table of runs, including saved/planned trials, dimension and point count when recorded in run metadata, and whether each run has written its completion summary. "Unfinished" can mean running or interrupted. Counts describe **saved attempts**: retries count separately from their source trials. It reads journals for speed, falls back to trial files for missing journal entries, and counts malformed trials separately as unreadable. It does not rely on potentially stale `summary.json` totals. The report is a snapshot during a live search; unrecorded work is not counted. `--json` provides the same counts, per-run details, and warnings as JSON; `--output` saves the selected format to a file.

| Status | Meaning |
| --- | --- |
| `found` | A tree passed every interior-overlap test at the specified tolerance. |
| `no-unfolding` | The search exhausted all trees, including branches excluded by overlap pruning. This is a **numerical counterexample candidate**. |
| `inconclusive` | A node or time limit stopped the search before it found an unfolding. |

`termination` records `solution`, `exhausted`, `maxNodes`, or `timeout`. `exhaustive` is true only for `exhausted`. With `--all`, a limited run can still report `found` if it has already found a witness; its solution count is then only a lower bound. `completeTrees` counts actual leaves visited, not the trees eliminated by partial-overlap pruning.

`search` uses successive seeds and writes every trial to a fresh `results/run-*` directory (`--output` changes the parent directory). Each trial records the original points, generator parameters, hull and solver options, facet/ridge indices, status, statistics, and any unfolding. `trials.jsonl` provides one summary per trial; `summary.json` contains totals. Geometry failures are saved as errors, separate from candidates and cutoffs. Interrupting a batch preserves completed trials.

Rerun just the inconclusive examples from a run directory:

```sh
pnpm retry results/run-XXXXXX
pnpm retry results/run-XXXXXX results/run-YYYYYY
# Optional new limits or tolerance:
pnpm retry results/run-XXXXXX --timeout-ms 60000 --overlap-tolerance 1e-10
```

`retry` reads each directory's `trial-*.json` files and selects only `result.status: "inconclusive"`. It restarts each search using the exact saved points, tolerances, root, and search settings, with **all computational limits reset to Infinity** unless explicitly supplied. Solver/geometry flags override saved settings. It writes one fresh `retry-*` directory under the first input directory's parent; `--output` changes the parent directory. A single input preserves trial filenames; multiple inputs use new sequential filenames to avoid collisions. Each record has a `sourceFile` link to its original trial. Repeated directory arguments are processed once. Original runs are untouched. A retry directory can itself be retried. Selection is a snapshot of the files present when the command starts; it does not wait for ongoing source runs. If there are no inconclusive examples, it prints that fact and creates no directory.

To investigate a candidate or a cutoff, pass its saved trial JSON to `solve`. This reuses the **points**; specify the desired tolerances and budgets again:

```sh
pnpm solve results/run-XXXXXX/trial-000000-seed-0.json --tolerance 1e-11 --overlap-tolerance 1e-10 --max-nodes Infinity --timeout-ms Infinity --output recheck.json
```

The implementation uses floating-point geometry, not exact predicates or certified arithmetic. Points are translated and uniformly scaled to radius 1 before computation. The default hull tolerance is `1e-9`; overlap requires a common interior point more than `16*tolerance` from every facet boundary. Boundary contact is permitted, and overlaps thinner than this threshold can be missed. Near-degenerate hulls can have incorrect incidence or be rejected; linear systems with pivots at most `1e-13` are skipped. Both positive and negative results need independent, higher-precision validation before mathematical use. Rerunning at different tolerances is a diagnostic, not a certificate.

## Algorithm

1. **Convex hull:** enumerate every $d$-subset of the input points. Affinely independent subsets define candidate supporting hyperplanes. Retain those with all points on one side, and merge candidates by their full set of coplanar point indices. Thus nonsimplicial facets remain intact. Two facets are adjacent exactly when their intersection has affine dimension $d-2$.
2. **Tree search:** fix one root facet and branch on the first undecided ridge leaving the connected set of placed facets. Either attach its unplaced neighbor or forbid that ridge. Internal edges cannot be selected because they would create cycles. Every spanning tree determines exactly one sequence of these choices. A fixed root loses no unfoldings: changing the root changes only the global Euclidean placement.
3. **Development:** compute an orthonormal basis along the shared ridge and a perpendicular inward direction in each incident facet. Preserve the ridge pointwise and map the child's inward direction to the negative of the parent's inward direction. This gives an affine isometry into $\mathbb R^{d-1}$ without dimension-specific rotation formulas.
4. **Intersection:** express each developed facet as the intersection of halfspaces defined by its ridges. For a pair of facets, maximize $t$ subject to $n_i\cdot x+t\le b_i$ for all their unit outward normals. A positive optimum means interior intersection. Enumerate vertices of this linear program by choosing $d$ active constraints in its $d$ variables $(x,t)$. Bounding boxes and center tests provide shortcuts. This detects intersections even when neither facet contains a vertex of the other.
5. **Pruning:** if a new facet overlaps an already placed facet, discard that branch. Descendant attachments cannot change the existing placements. Also discard branches whose remaining ridges cannot connect all facets.

There is no dimension-specific upper bound. The implementation is deliberately brute force: hull construction examines $\binom nd$ subsets, intersection of two facets with $m$ combined bounding halfspaces examines up to $\binom md$ linear systems, and spanning-tree counts grow exponentially. Start with few vertices and inspect facet counts. Higher dimension can make the computation much harder even if counterexamples are easier to find.

All computational limits default to `Infinity` for both `solve` and `search`. Set `--max-hull-combinations`, `--max-nodes`, or `--timeout-ms` to impose a limit. Node and time limits apply **per polytope** to tree search, excluding hull construction. A hull cutoff is an error, never an exhausted unfolding search. Time checks are cooperative, including during intersection enumeration.

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

Tests cover hull incidence, nonsimplicial facets, dimensions 2–6, hinge continuity, isometries, boundary contact, crossing intersections, exhaustive tree counts, pruning against independent enumeration, an independent planar intersection test, seeded generators, and command-line replay and cutoffs. `pnpm check` type-checks source and tests without running them. TypeScript is pinned to the 5.x compiler API used by Civet; casing enforcement is disabled to accommodate Civet's virtual paths on Windows.
