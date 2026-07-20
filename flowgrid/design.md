# FlowGrid: Simulating Self-Driven Cars with Load-Aware Routing & Adaptive Signals

A design document and working implementation of a traffic simulation in which **autonomous vehicles route themselves by live network load** and **traffic signals allocate green time by live queue pressure** — and in which both strategies can be toggled against dumb baselines mid-run, so the benefit of autonomy is *measured*, not asserted.

The implementation (`flowgrid.html`, single file, zero dependencies) ships with this document and passed a headless physics test before delivery.

---

## 1. Vision and Goals

Open the page: a 3×3 grid of intersections, twelve entry/exit gates, and a stream of self-driven cars flowing through. Roads glow hotter as they load up. Each signal shows its live green axis; each car is an agent with a destination that **re-plans its route at every junction** using congestion-weighted shortest paths. Flip signals from *Adaptive (pressure)* to *Fixed cycle*, or routing from *Load-adaptive* to *Shortest path*, and watch queues form; the strategy scoreboard accumulates average trip time per configuration so the comparison is quantitative.

**Design goals**

1. **Two autonomies, separable.** Vehicle-side intelligence (routing) and infrastructure-side intelligence (signal control) are independent toggles, because their benefits are different and the sim should isolate each.
2. **Measurable, not theatrical.** Live KPIs (avg trip, avg wait, throughput/h, cars on road) plus a per-strategy scoreboard; the sim's job is to produce evidence.
3. **Deterministic core.** Seeded RNG (mulberry32) and a fixed 60 Hz timestep — a given seed and strategy replays identically, which is what makes A/B numbers meaningful.
4. **Testable without a browser.** The simulation core is DOM-free; the same file, run under Node, executes a smoke test (§5). Physics bugs get caught by CI, not by eyeballs.
5. **Honest scope.** This is a mesoscopic strategy sandbox — the right tool for studying *control policies*. It is not a microscopic traffic twin (no lane changes, no turn-conflict phases, no human-driver mix); §6 maps the path to SUMO when that fidelity is needed.

---

## 2. System Model

### 2.1 Network
A directed graph: 9 junction nodes + 12 border (source/sink) nodes, every adjacent pair joined by two one-way edges (one per direction). Each edge carries an ordered car list, a length, an axis tag (NS/EW), and a nominal capacity used by the routing cost. Border nodes are enter/exit only — the router forbids through-travel across them.

### 2.2 Vehicles (the "self-managed, self-driven" agents)
Each car = `{path, leg, pos, v, dest, wait}`. Longitudinal control is a point-follow controller: accelerate toward `v_max`, brake proportionally to the gap to the obstacle ahead — where "obstacle" is the leader on the same edge, a red stop line, or the tail of the next edge (the *don't-block-the-box* rule: a car will not enter a junction unless the receiving edge has space, which is what prevents gridlock).

**Load-adaptive routing.** Edge cost = `(len / v_max) × (1 + α·(n/capacity)²)` with α = 4 — the quadratic makes near-jammed edges effectively poisonous. Every car runs Dijkstra at spawn, and — when the *Load-adaptive* mode is on — **re-runs it at every junction it clears**, so the fleet continuously drains toward under-used corridors. With *Shortest path* mode, cars stick to their spawn-time geometric route regardless of congestion (the human-GPS-circa-2005 baseline).

### 2.3 Signals
Each junction runs two phases (NS-green / EW-green) with a 2.4 s yellow interlock.
- **Fixed cycle (baseline):** 14 s per phase, always, regardless of demand.
- **Adaptive (pressure):** after a 6 s minimum green, the controller compares waiting-queue totals on the two axes and switches when the *other* axis's pressure exceeds the served one (with +1 hysteresis to prevent flapping), and immediately when the served axis is empty but the other is not. This is a simplified **max-pressure / longest-queue-first** policy — the family of decentralized controllers known to be throughput-optimal in the literature, chosen precisely because it needs no coordination, no schedule, and no training: each intersection reacts only to its own queues.

### 2.4 Demand
Poisson-like arrivals at border gates (accumulator on a veh/h slider, 200–2400), uniform random origin→destination pairs, spawn suppressed when the entry edge tail is occupied (so overload shows up as queue spillback, not fake cars).

---

## 3. What the Simulation Demonstrates (verified numbers)

Headless run, seed 7, demand 1400 veh/h, 4 simulated minutes, identical arrivals:

| Strategy | Avg trip (s) | Avg wait (s) | Completed |
|---|---|---|---|
| Fixed signals + static routing | 38.2 | 15.0 | 81 |
| **Adaptive signals + load routing** | **31.6** | **8.4** | 80 |

Same demand served, **~44 % less time stopped** and ~17 % faster trips — and in the live UI the scoreboard lets you reproduce this interactively, including the two mixed configurations (adaptive signals alone vs. load routing alone) to see which contributes what at which demand level. The interesting emergent behavior to try: push demand past ~1800 veh/h under static routing and watch one corridor jam while parallel roads sit empty; flip to load routing and watch the jam *dissolve sideways*.

---

## 4. Interface Design

Control-room aesthetic grounded in the subject: asphalt-dark canvas as the hero (the simulation *is* the page), lane-marking amber as the accent, signal-green/red reserved strictly for their real meanings. The **signature element is the load glow** — each road segment's stroke widens and heats from cool grey toward red as occupancy rises, so network load is readable at a glance without a single number. Right rail: two strategy segments (signals / routing), demand and sim-speed sliders, four live KPI tiles, the strategy scoreboard table (best trip time highlighted green once a config has ≥ 20 samples), pause/reset. Cars render color-coded by identity and flash red when stopped, so queues are visible as red beads. Responsive down to mobile (panel drops below canvas).

---

## 5. Engineering & Test Design

**One file, two runtimes.** The sim core (network builder, Dijkstra, controller, signal logic, stepper, stats) uses no DOM APIs. Under a browser it drives the canvas UI; under Node the very same `<script>` body executes a **smoke test**: run both extreme configurations for 4 simulated minutes and assert ≥ 50 completed trips each (i.e., flow exists, no deadlock), printing the comparative stats. This test was run before delivery and passes (§3 numbers are its output). That structure means every future change to the physics is one `node simtest.js` away from verification.

Determinism guarantees: seeded RNG, fixed 1/60 s substeps accumulated from wall-clock (render rate never affects physics), and stats windows computed in sim time.

Known simplifications (deliberate): single lane per direction; turns inherit the axis-phase permission (no protected-turn conflicts); no lane changing; vehicles are homogeneous. Each is listed with its upgrade in §6.

---

## 6. Roadmap

**Phase 0 (done).** Grid network, agent cars with gap control and box-blocking prevention, Dijkstra + congestion-weighted re-planning, fixed vs. max-pressure signals, live KPIs, strategy scoreboard, headless test.
**Phase 1 — richer control.** Per-approach queues with protected left-turn phases; green-wave coordination baseline; SCOOT-style cycle adaptation as a third signal mode; anticipatory routing (cars share *intended* routes so cost reflects future load — the true "connected AV" advantage).
**Phase 2 — learning controllers.** Replace max-pressure with a reinforcement-learning signal agent (per-junction DQN, queue state → phase action) trained *inside this sim*, benchmarked on the scoreboard against max-pressure — the honest test any RL controller must pass.
**Phase 3 — scenario science.** Configurable networks (JSON), incident injection (block an edge, watch load routing reroute the fleet), demand profiles (rush-hour waves), batch runner exporting CSV for real experiments.
**Phase 4 — fidelity jump.** Port the validated policies to **SUMO** (with TraCI) for lane-level microsimulation and real city map imports (OSM); FlowGrid remains the fast policy-prototyping front end.

---

## 7. Risk Register

| Risk | Severity | Mitigation |
|---|---|---|
| Gridlock at extreme demand | Medium | Don't-block-the-box rule + entry metering; verify with high-demand smoke run |
| Oscillating adaptive signals | Medium | Minimum green + hysteresis threshold (implemented) |
| All cars rerouting to the same "empty" road (flapping) | Medium | Quadratic cost softens herding; Phase 1 anticipatory routing solves it properly |
| Results mistaken for real-world predictions | High | §1 honesty: policy sandbox, not a twin; SUMO path documented for fidelity claims |
| Physics regressions from UI edits | Low | DOM-free core + Node smoke test in one command |
