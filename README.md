# Hot Delivery — Case Study

**Course:** RSM 8443 — Optimizing Supply Chain Management and Logistics, Winter 2026  
**Topic:** Prescriptive routing and assignment models for online food delivery platforms

This repository contains the full solution to the Hot Delivery case study, developed
as consultants for Uber Eats' data analytics group. The work progresses through four
incrementally richer models — from a single-driver distance minimiser to a real-time
greedy heuristic — and is fully reproducible from the data files and notebook included here.

---

## Repository structure

```
hot-delivery/
├── README.md
├── notebook.ipynb          ← all models, analysis, and visualizations
├── data/                   ← input CSV files (see Data files below)
├── docs/
│   └── case_description.pdf
├── report/
│   └── Hot_Delivery_Report.pdf
└── figures/                ← generated maps and trade-off plots (auto-created on run)
```

---

## Environment setup

**Requirements:** Python ≥ 3.9, PuLP ≥ 2.7, pandas ≥ 1.5, folium ≥ 0.14

Install all dependencies in one step:

```bash
pip install pulp pandas numpy matplotlib folium
```

**Optional — Gurobi:** The solver helper in Cell 5 automatically uses Gurobi when
`gurobipy` is installed and a valid licence is found, and silently falls back to CBC
(bundled with PuLP) otherwise. All results in the report were produced with CBC.
CBC solves every instance except `part4_large` within two minutes; the large instance
is handled by the heuristic in Part IV and does not require a MIP solver.

---

## Data files

All files must be placed in the `data/` folder directly under the repository root.
The notebook reads every file as `DATA / "filename.csv"` where `DATA` is resolved
at runtime — no manual path editing is needed.

| File | Used in | Description |
|------|---------|-------------|
| `distances.csv` | All parts | Pairwise distances (km) between Toronto neighbourhood centroids |
| `regions.csv` | All parts | Neighbourhood names with latitude/longitude for map rendering |
| `part1_ordersA.csv` | Part I | Instance A — 2 orders, single driver |
| `part1_ordersB.csv` | Part I | Instance B — 5 orders, single driver |
| `part2_ordersA.csv` | Part II | Instance A with food release timestamps |
| `part2_ordersB.csv` | Part II | Instance B with food release timestamps |
| `part3_small.csv` | Parts III & IV | Small instance — 5 orders |
| `part3_drivers.csv` | Part III | 3 drivers with depots and speeds |
| `part4_large.csv` | Part IV | Large instance — 15 orders |
| `part4_drivers.csv` | Part IV | 10 drivers for the large instance |

---

## Running the notebook

Open `notebook.ipynb` in Jupyter and run cells **in order from top to bottom**.
Every part depends on the shared setup cells at the top (Cells 0–5), which must be
executed first. The notebook is organised as follows:

| Cells | Part | What runs |
|-------|------|-----------|
| 0–5 | Setup | Imports, path configuration, shared data loading, constants, solver helper |
| 6–10 | Part I | Single-driver MIP solver + route visualization |
| 11–15 | Part II | Time-constrained MIP, W sweep, metric comparison, visualization |
| 16–20 | Part III | Multi-driver MIP, W × D sweep, timing audit, visualization |
| 21–25 | Part IV | Cluster-then-route heuristic, benchmark comparison, visualization |

**Expected runtimes** (CBC solver, modern laptop):

| Cell | Task | Approx. time |
|------|------|-------------|
| 8 | Part I solve (both instances) | < 5 s |
| 13 | Part II benchmark + W sweep (30 solves) | 5–10 min |
| 19 | Part III W × D sweep (up to 90 solves) | 20–45 min |
| 23 | Part IV heuristic (both instances, both repair modes) | < 1 s |

Part III Cell 19 is the most compute-intensive cell. If runtime is a concern, set
`time_limit=60` in the `solve_all_drivers` calls inside Cell 19 — the solver
will return the best incumbent found within that window.

---

## Notebook walkthrough

### Setup (Cells 0–5)

Cell 0 imports all libraries and checks minimum version requirements — a failing
assertion here means a dependency needs upgrading before proceeding.

Cell 2 resolves the `DATA` path at runtime. It works regardless of whether you
open the notebook from the repository root, a subdirectory, or via
`jupyter nbconvert --execute`. If the `data/` folder is missing it raises a
`FileNotFoundError` with the expected directory structure.

Cell 3 loads `distances.csv` and `regions.csv` once and exposes two helper
functions used throughout: `get_dist(a, b)` returns distance in km and
`get_coord(name)` returns `(latitude, longitude)` for map rendering. Both raise
a `KeyError` with an actionable message if a name is not found.

Cell 5 defines `make_solver()`, which tries Gurobi first and falls back to CBC.
It runs a smoke-test on every kernel start so a broken solver install fails
immediately rather than silently returning wrong results.

### Part I — Single-driver routing (Cells 6–10)

Cell 7 contains the model description markdown including the full constraint table
and the justification for the end-of-route treatment (C5).

Cell 8 solves the distance-minimisation MIP for instances A and B and stores
results in `_results_p1`. Cell 10 draws Folium route maps from those cached
results without re-solving. Maps are saved to
`figures/map_part1_A.html` and `figures/map_part1_B.html`.

### Part II — Time-constrained routing (Cells 11–15)

Cell 12 contains the model description markdown including the big-M derivation,
the T4a/T4b constraint pair, and the explanation of why max-wait is a strict
subset of avg-wait.

Cell 13 defines `solve_p2()` and runs three analysis blocks in sequence:

1. A benchmark solve at W = 60 for both instances (results cached in
   `_results_p2_bench`).
2. A W sweep over `[10, 15, 20, 25, 30, 35, 40, 45, 50, 60, 75, 90, 120, 150, 200]`
   for both average-wait (T4a) and max-wait (T4b) formulations, results cached in
   `_sweep_p2`.
3. An evidence table printing all feasible W values with step-transition annotations,
   and a metric comparison table at selected W values — both read from `_sweep_p2`
   with no re-solves.

> **Note:** Cell 13 does not save a trade-off plot file. To generate the
> W-vs-distance figure for the report, add the following line immediately before
> `plt.show()` at the bottom of Cell 13's plot block:
> ```python
> plt.savefig(Path("figures") / "part2_tradeoff.png", dpi=150, bbox_inches="tight")
> ```

Cell 15 defines `visualize_p2()` and renders maps for W = 60 under both metrics,
reading from the cached results. Maps are saved as
`figures/map_part2_{inst}_{metric}W{W}.html`
(e.g. `map_part2_A_avgwaitW60.html`, `map_part2_B_maxwaitW60.html`).

### Part III — Multi-driver routing and assignment (Cells 16–20)

Cell 17 contains the full model description markdown including the multi-driver
constraint table and the wait-time carryover worked example.

Cell 18 defines `solve_all_drivers()` and `_print_p3_solution()`, loads
`part3_small.csv` and `part3_drivers.csv`, and prints the reference time and
release times as a data sanity check. It ends with
`print("Solver and helpers ready.")` — no benchmark solve runs in Cell 18.

Cell 19 runs the W × D sweep — the most expensive cell in the notebook.
It loops over `D_caps = [1, 2, 3]` and
`W_values = [10, 15, 20, 25, 30, 35, 40, 45, 50, 60, 75, 90, 120, 150, 200]`,
solving both average-wait and max-wait formulations at each combination, with an
early-exit plateau check per metric. Results are cached in
`_sweep_p3[D_cap][metric]` as `(W, result_dict)` lists.

Cell 19 then produces four outputs in sequence, all reading from the cache:

1. **Sanity check** — calls `_print_p3_solution()` for D ≤ 3 at W = 60, printing
   a full per-driver arc table with arrival times and a wait-time reconciliation
   table that verifies the T4 constraint directly from the notebook output.
2. **Evidence table** — all feasible W values for every D cap and metric, with
   step-transition annotations showing where distance jumps.
3. **Summary table** — distance, avg wait, max wait, and deployed drivers at
   W = 60 across all three D caps.
4. **Trade-off plot** — one curve per D cap (D ≤ 1, D ≤ 2, D ≤ 3), saved to
   `figures/part3_tradeoff_WD.png`.

Cell 20 defines `visualize_p3()` and renders one map per D cap at W = 60,
reading from `_sweep_p3`. Maps are saved as
`figures/map_part3_small_D{D}_W60.html`
(e.g. `map_part3_small_D1_W60.html`, `map_part3_small_D2_W60.html`,
`map_part3_small_D3_W60.html`).

### Part IV — Cluster-then-route heuristic (Cells 21–25)

Cell 22 contains the heuristic model description markdown including the Phase 3b
max-wait repair rationale, the known SLA limitation of the large instance, and
the online applicability discussion.

Cell 23 defines the three-phase heuristic (`cluster_then_route()`) and runs four
evaluations:

| Variable | Instance | Repair mode | W_max |
|----------|----------|-------------|-------|
| `_res_small_avg` | small | avg-repair only | ∞ |
| `_res_small_max` | small | avg + max-repair | 1.5 × W = 180 min |
| `_res_large_avg` | large | avg-repair only | ∞ |
| `_res_large_max` | large | avg + max-repair | 180 min |

The MILP benchmark for comparison is read from `_sweep_p3[3]["avg"]` at W = 120
— no re-solve. **Cell 19 must be run before Cell 23** for this cache lookup to
succeed. A final summary table reports distance, average wait, max wait,
feasibility flags (`avg_feasible`, `max_feasible`), and wall time for all four runs.

Cell 25 renders maps for all four runs, with delivery node tooltips colour-coded
by SLA breach status (dark red = individual wait > 120 min). Maps are saved as
`figures/map_part4_{inst}_{repair_mode}.html`
(e.g. `map_part4_small_avgrepair.html`, `map_part4_large_maxrepair.html`).

---

## Reproducing specific results

**Part I optimal distances** (Table in report §Part I):
Run Cells 0–5, then Cell 8. Results print to stdout and are stored in
`_results_p1["A"]` and `_results_p1["B"]`.

**Part II trade-off evidence** (Tables in report §Part II):
Run Cells 0–5, then Cell 13 in full. The evidence table and metric comparison
print to stdout. See the note in the Cell 13 section to save the trade-off plot.

**Part III W × D summary table** (Table in report §Part III):
Run Cells 0–5, then Cells 18–19 in order. The evidence table and W = 60 summary
print at the end of Cell 19. The trade-off plot is saved to
`figures/part3_tradeoff_WD.png`.

**Part III timing audit** (Route tables in report):
The same Cell 19 run prints the full per-driver arc table with wait-formula
reconciliation for D ≤ 3 at W = 60 in the sanity check block at the top of
Cell 19's output.

**Part IV benchmark comparison** (Table in report §Part IV):
Run Cells 0–5, 18–19, then Cell 23. Cell 19 must complete first — the comparison
table in Cell 23 reads the MILP optimal from the Part III sweep cache.

---

## Inter-cell dependencies

The table below shows which cells must have been run before each analysis cell.
Running the notebook top-to-bottom satisfies all dependencies automatically.

| Cell | Depends on |
|------|-----------|
| 8 (Part I solver) | 0–5 |
| 10 (Part I viz) | 0–5, 8 |
| 13 (Part II solver + sweep) | 0–5 |
| 15 (Part II viz) | 0–5, 13 |
| 18 (Part III solver definition) | 0–5 |
| 19 (Part III W × D sweep) | 0–5, 18 |
| 20 (Part III viz) | 0–5, 18, 19 |
| 23 (Part IV heuristic) | 0–5, 18, 19 |
| 25 (Part IV viz) | 0–5, 23 |

---

## Key design decisions

**No hardcoded paths.** Every file is accessed via `DATA / "filename.csv"`.
Running the notebook from any working directory produces the same results.

**No re-solves.** Every visualization and comparison table reads from cached
result dicts produced by the solver cells. Re-running a visualization cell
without re-running the solver cell is safe and fast.

**Result dicts are self-contained.** Each solver returns a dict with all fields
needed by downstream cells — node names, index sets, arrival times, wait times,
and routes. No global state is shared between solver and visualization cells
beyond these dicts.

**Solver fallback is transparent.** `make_solver()` prints which solver was
selected and runs a smoke-test. If Gurobi is found but unlicensed, a
`RuntimeWarning` is raised so the fallback to CBC is explicit rather than silent.

---

## Results summary

| Part | Instance | Key result |
|------|----------|-----------|
| I | A | 4.71 km optimal route, 2 orders |
| I | B | 33.91 km optimal route, 5 orders |
| II | A, W = 60 | 43.32 km, avg wait 12.32 min |
| II | B, W = 60 | 55.63 km, avg wait 58.40 min |
| III | small, D ≤ 3, W = 60 | 30.18 km, 2 drivers deployed |
| IV | small, W = 120 | 41.22 km heuristic vs 30.18 km MILP (36.6% gap) |
| IV | large, W = 120 | 83.90 km, 7 of 10 drivers deployed, 1.1 ms solve time |

---

## Citation

Case study: *Hot Delivery — A Case Study in Online Food Delivery Platforms*,
RSM 8443, Rotman School of Management, University of Toronto, Winter 2026.
