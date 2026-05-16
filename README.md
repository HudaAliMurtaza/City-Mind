# CityMind — Urban Intelligence System

> **BS Computer Science — AI Semester Project**
> Group Project | Department of Computer Science

CityMind is an AI-driven city management simulation. A 10×10 grid city is
planned, connected, and monitored by five cooperating AI modules — all sharing
a single live city graph. Roads flood, crime shifts, ambulances reposition, and
a medical team navigates to trapped civilians in real time.

---


## Requirements

```
Python 3.11+
pygame
pygame.freetype
numpy
scikit-learn
```

Install dependencies:

```bash
pip install pygame numpy scikit-learn
```

---

## Running the Project

```bash
python ui.py
```

There is one entry point. The UI builds the city, runs all five challenge
modules in sequence, then launches the 20-step simulation dashboard.

---

## Controls

| Key / Button | Action |
|---|---|
| `SPACE` | Play / Pause simulation |
| `S` | Single step forward |
| `Z` / Rewind button | Step back one step |
| `R` / Reset button | Rebuild city from scratch |
| `1` | Toggle Road Network overlay |
| `2` | Toggle Ambulance Coverage overlay |
| `3` | Toggle Crime Heatmap overlay |
| `↑ ↓` | Scroll event log |
| `ESC` | Quit |

Speed is controlled by the slider in the bottom bar (100 ms – 2 s per step).

---

## Project Structure

```
AI Project/
├── ui.py                      ← Single entry point — dashboard UI (Pygame)
├── city_graph.py              ← Shared data model (single source of truth)
├── challenge1csp.py           ← City layout planning (CSP + Backtracking)
├── challenge2road.py          ← Road network optimisation (Kruskal + GA)
├── challenge3ambulance.py     ← Ambulance placement (Simulated Annealing)
├── challenge4emergencyroute.py← Emergency routing (A* + dynamic replanning)
└── challenge5crime.py         ← Crime prediction + police deployment (K-Means + KNN)
```

No module maintains its own copy of the city. Every read and write goes
through the shared `CityGraph` instance in `city_graph.py`.

---

## The Five Challenges

### Challenge 1 — City Layout Planning
**Algorithm: CSP with Backtracking + MRV + Forward Checking**

Places Hospitals, Schools, Industrial zones, Power Plants, and Ambulance
Depots on the grid satisfying three constraints:

- **C1 (hard):** Industrial zones cannot be adjacent to Schools or Hospitals.
- **C2 (soft):** Every Residential cell should be within 3 hops of a Hospital.
  Treated as soft because corner coverage is mathematically limited by the
  number of hospitals on a 10×10 grid.
- **C3 (hard):** Every Power Plant must have an Industrial zone within 2 hops.

MRV (Minimum Remaining Values) heuristic with degree tie-breaking selects
variables. Hospital pre-seeding near grid edges maximises C2 coverage before
backtracking begins. If backtracking fails, a constraint-aware greedy fallback
guarantees C1 = 0 violations and reports which constraint caused the conflict.

All constraint distances are centralised in the `CONSTRAINTS` dict at the top
of `challenge1csp.py` — changing one number there is all that is needed for
the live modification challenge.

---

### Challenge 2 — Road Network Optimisation
**Algorithm: Kruskal's MST → seeded Genetic Algorithm**

Finds the minimum-cost road network with a hard safety requirement: at least
two completely independent paths must exist between the Primary Hospital and
the Ambulance Depot (verified via Edmonds-Karp max-flow).

- Kruskal's MST seeds the GA population so evolution starts from a
  near-optimal baseline.
- Fitness = total road cost + heavy penalty if max-flow < 2.
- GA parameters: 60 population, 150 generations, 80% crossover, elitism 4.
- Roads not selected are marked `is_blocked = True` on the shared graph,
  immediately visible to all other modules.

---

### Challenge 3 — Ambulance Placement
**Algorithm: Simulated Annealing (multi-start)**

Places 3 ambulances to minimise the **worst-case response distance** — the
maximum distance from any populated node (all node types with
`population_density > 0`, not just Residential) to its nearest ambulance.

- Dijkstra runs from each ambulance position; objective = max over all
  populated nodes of (min distance to nearest ambulance).
- SA move: shift one ambulance to a random passable neighbour.
- 3 restarts × 3000 iterations, cooling rate 0.995.
- Re-runs automatically every simulation step when crime weights shift
  (via `rerun_placement()`), keeping placement in sync with the live graph.

---

### Challenge 4 — Emergency Routing Under Changing Conditions
**Algorithm: A\* with Dynamic Replanning + Deferred Civilian Queue**

Routes a medical team from the Primary Hospital to rescue all trapped
civilians, nearest-first by actual A\* path cost.

- Euclidean heuristic is admissible and consistent — guarantees optimal paths.
- When any road floods (`is_blocked` changes), `replan()` is called
  immediately from the current position.
- **"Reach ALL civilians" guarantee:** if A\* finds no path to the current
  target for `NO_PATH_STRIKE_LIMIT` consecutive steps, that civilian is moved
  to a `deferred` list and the team continues to others. Once the main queue
  is exhausted, deferred civilians are retried — roads may have reopened.
  The simulation is only declared done when both queues are empty.

---

### Challenge 5 — Crime Risk Prediction and Integration
**Algorithm: K-Means Clustering (unsupervised) + KNN Classifier (supervised)**

Four-step pipeline:

1. **K-Means (k=3):** clusters every node by population density and industrial
   proximity — no labels used.
2. **Synthetic dataset:** crime labels generated from domain rules
   (industrials + density → HIGH; residential near industrial → MEDIUM; etc.)
   with 10% random noise.
3. **KNN (k=5, distance-weighted):** trained on the synthetic dataset,
   predictions written back to the shared graph as `crime_risk_level`.
   Edge `effective_cost` values are recalculated automatically
   (HIGH = 1.5×, MEDIUM = 1.2×, LOW = 1.0×).
4. **Police deployment:** 10 officers allocated proportionally across HIGH and
   MEDIUM risk nodes (HIGH nodes weighted 3×), prioritising higher population
   density within each tier. Stored as `node.police_officers`.

---

## Shared Graph — Integration Design

```
CityGraph (single instance)
    │
    ├── Challenge 1 writes: node types, population densities, base road costs
    ├── Challenge 2 writes: is_blocked = True for unbuilt roads
    ├── Challenge 5 writes: crime_risk_level, edge effective_cost multipliers,
    │                       police_officers
    ├── Challenge 3 reads:  effective_cost for Dijkstra; writes ambulance positions
    └── Challenge 4 reads:  effective_cost, is_blocked for A*; block_road() on floods
```

Any change to an edge or node is immediately visible to every other module
because they all hold a reference to the same object — no copying, no syncing.

---

## Simulation Loop (20 Steps)

Each step:
1. Random road flood (probability 0.2) → `block_road()` → router replans
2. Every 5 steps: one node's crime risk escalates → edge costs update →
   ambulances reposition (SA re-runs) → router replans
3. Medical team advances one node along current A\* path
4. Event log updated; UI redraws

---

## UI Features

- **Three overlay toggles** (buttons + keyboard 1/2/3):
  - Road Network — edges coloured by cost, blocked roads shown with ✕
  - Ambulance Coverage — per-cell heatmap by Dijkstra distance to nearest ambulance
  - Crime Heatmap — amber/red tints with ▲ danger icons and police officer badges
- **Live event log** — colour-coded INFO / WARNING / CRITICAL entries with step badges
- **Step-by-step rewind** — snapshots saved before every advance, full state restore
- **Mission timeline strip** — coloured bar showing rescues, floods, and moves per step
- **Stats bar** — civilians rescued progress bar, ambulance coverage %, worst response
  distance, blocked road count
- **Mission summary banner** — displayed on grid when simulation ends
- **CSP constraints panel** — live display of active C1/C2/C3 parameters in sidebar

---

