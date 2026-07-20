# CleanLoop: An AI-Hybrid Waste Management Platform for a City

A design and implementation plan for a city-scale, AI-hybrid web application that manages the full life of waste: **segregation at source** (bio, recyclable, e-waste, hazardous), **collection and removal**, **recycling / reuse / reprocessing**, **wastewater treatment oversight**, and — only as the last resort — **scientifically sited disposal far from public areas**. The system's core economic principle, exactly as you framed it: *make removal and recycling the paying, easy path so waste never piles up.*

---

## 1. Vision and Goals

A resident photographs a broken mixer-grinder in the CleanLoop app; the vision model classifies it *e-waste, category: small appliance*, offers a doorstep pickup slot, and credits their account when the certified recycler logs receipt. A ward officer's dashboard shows this morning's fill-level predictions for 400 collection points, the optimizer's truck routes, and a red flag on one sewage-treatment inlet whose sensor pattern the anomaly model says looks like an industrial discharge. The city engineer opens the GIS module and sees candidate processing-site parcels ranked by a suitability model that has already excluded everything near homes, schools, water bodies and floodplains. Every ton of waste is tracked from source to fate; "fate = landfill" is the metric everyone is paid to shrink.

**Design goals**

1. **Segregation-first.** AI makes correct segregation effortless (photo → category → instructions/pickup) and auditable (contamination detection at trucks and MRFs).
2. **Circular before terminal.** Every item is routed up the hierarchy — *reduce → reuse → recycle → recover (compost/biogas/energy) → treat → dispose* — and the platform's incentives (credits, marketplace, penalties) enforce that order.
3. **Hybrid AI, honestly hybrid.** ML where data patterns rule (vision, forecasting, anomaly detection, routing), and **transparent rule/optimization systems where law and safety rule** (siting buffers, hazardous-waste handling, treatment compliance). Regulations are constraints, never model suggestions.
4. **Disposal as a shrinking remainder.** Site identification exists in the plan (your Phase 3) but the KPI structure makes it the *least* used module.
5. **Every actor on one platform.** Citizens, collection crews, MRF operators, recyclers, treatment-plant operators, and municipal officers each get a role-specific interface over one data spine.

**Honest constraints**

- **No system "removes all waste issues."** What a system can do is make every issue *visible, owned, and trending downward*. CleanLoop therefore ships with hard KPIs per phase (§7) — landfill-diversion %, segregation-compliance %, average pickup latency, treated-water compliance % — and the honest promise is continuous measurable reduction, not elimination.
- **AI cannot fix broken ground operations.** If trucks don't exist or an MRF has no buyers, no model helps. The plan pairs every AI capability with the operational and incentive mechanism that makes it matter.
- **Data is the real Phase 0.** Ward maps, collection-point registry, vehicle fleet, facility list, historical tonnage. The roadmap starts there because every model above feeds on it.

---

## 2. System Architecture

```
┌────────────────────────── Interfaces (web-first, PWA for mobile) ───────────────────────┐
│ Citizen PWA        Crew app          Operator consoles        City dashboard            │
│ scan-my-waste,     route runsheet,   MRF intake, recycler     KPIs, GIS, forecasts,     │
│ pickup booking,    proof-of-pickup,  marketplace, STP panel   compliance, digital twin  │
│ credits, reports   contamination cam                                                    │
└──────────────┬──────────────────────────────────────────────────────────────────────────┘
               ▼
┌────────────────────────────── Core Platform (FastAPI/Django + Postgres/PostGIS) ────────┐
│ Waste Passport ledger (every batch: source → transfers → fate)                          │
│ Registry: wards, collection points, fleet, facilities, recyclers                        │
│ Incentive engine: credits, marketplace escrow, penalty workflow                         │
│ Task/dispatch service · Notification service · Open-data API                            │
└───────┬───────────────────────┬─────────────────────────────┬───────────────────────────┘
        ▼                       ▼                             ▼
┌─ AI Services ─────────┐ ┌─ Optimization ────────┐ ┌─ IoT Gateway ───────────────────────┐
│ VisionSort: waste     │ │ OR-Tools VRP routing  │ │ bin fill sensors (ultrasonic),      │
│  classifier (photo →  │ │ (time windows, truck  │ │ truck GPS, weighbridge feeds,       │
│  bio/dry/e-waste/     │ │  capacity, zones)     │ │ STP sensors (flow, pH, BOD/COD      │
│  hazard, subtype)     │ │ Facility-location     │ │  proxies, DO, turbidity) via MQTT   │
│ FillCast: per-point   │ │  models for new units │ └─────────────────────────────────────┘
│  fill/tonnage forecast│ │ GIS-MCDA siting engine│
│ FlowGuard: STP anomaly│ │  (rule buffers + ML   │
│  detection + effluent │ │  suitability scoring) │
│  soft-sensing         │ └───────────────────────┘
│ HotspotNet: dump-spot │
│  detection from citizen photos + patrol imagery                                        │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**The Waste Passport** is the spine: every collected batch gets an ID and an append-only chain of custody (who collected, weighed where, sent to which recycler/composter/plant, terminal fate). Diversion rates, recycler payouts, and fraud detection all read from this one ledger.

---

## 3. AI Model Inventory (the "hybrid" made explicit)

| Model | Type | Input → Output | Used in phase |
|---|---|---|---|
| VisionSort | CNN/ViT image classifier (fine-tuned on TrashNet/TACO + locally collected images) | photo → {bio, paper, plastic (by grade), glass, metal, e-waste subtype, hazardous, mixed} + handling instructions | 1 |
| ContamCam | Same backbone, deployed on crew phones / MRF conveyor cams | truck-tip / conveyor image → contamination % of "segregated" loads | 1 |
| FillCast | Gradient-boosted / temporal model per collection point | history + weekday + season + events → next-72 h fill & tonnage | 4 (drives routing) |
| RouteOpt | OR-Tools capacitated VRP with time windows (optimizer, not ML) | predicted demand + fleet + zones → daily routes | 4 |
| FlowGuard | Autoencoder/isolation-forest anomaly detection + soft-sensor regression | STP sensor streams → anomaly alerts; estimate hard-to-measure effluent params from cheap sensors | 2 |
| SiteRank | Rule-based exclusion (GIS buffers) + ML suitability scoring (MCDA-AHP hybrid) | parcel layers → ranked candidate sites with exclusion audit trail | 3 |
| HotspotNet | Object detection on geotagged citizen/patrol photos | image+GPS → illegal-dump hotspot map, recurrence prediction | derived |
| PriceSense | Simple forecaster on recyclable commodity prices | market feeds → dynamic credit/marketplace pricing | 1 |

The **hybrid rule**: anything touching public health, law, or land (siting buffers, hazardous handling chains, effluent standards) is deterministic and auditable; ML ranks, predicts, and flags *within* those hard constraints.

---

## 4. The Phases (yours, deepened) — each with its AI, operations, and exit KPI

### Phase 1 — Segregation, Recycling & Processing

Citizen PWA with **Scan-My-Waste** (VisionSort) → instant category + "what to do": bio to the green bin/home compost, recyclables to scheduled dry-day pickup, e-waste and hazardous to **book-a-pickup** with certified handlers only. Credits accrue per verified segregated kg (weighbridge + Waste Passport), redeemable against municipal fees or partner coupons — the "paying pathway" you specified. **Recyclables marketplace**: MRFs and aggregators list sorted bales (plastic grades, metals, e-waste lots); registered recyclers bid; escrowed payment on pickup — demand-pull so material moves instead of piling. ContamCam scores every "segregated" load; wards see their contamination league table; persistent offenders get education-first, penalty-later workflow. *Exit KPI:* ≥70 % source-segregation compliance in pilot wards; ≥40 % landfill diversion; marketplace clearing recyclables within 7 days of baling.

### Phase 2 — Wastewater & Bio-Treatment Oversight

IoT retrofit on sewage/effluent treatment plants (flow, pH, DO, turbidity, energy) streaming to FlowGuard: anomaly alerts (equipment drift, shock loads, suspected illegal industrial discharge into the network), soft-sensing of BOD/COD between lab tests, aeration-energy optimization suggestions (aeration is typically the largest cost, and ML-guided setpoints are proven savings). Compliance ledger auto-compiles regulator reports. Bio-waste loop: feed data from Phase 1 routes to composting/biogas units; the platform balances feedstock across units and tracks compost/energy output as *products* in the marketplace. *Exit KPI:* ≥95 % effluent-parameter compliance days; anomaly detection lead time > 2 h before threshold breach; 100 % of collected bio-waste reaching composting/biogas rather than dumps.

### Phase 3 — Disposal & Processing-Site Identification (last resort, publicly distant)

SiteRank runs in two locked stages. **Stage A — deterministic exclusion**, encoding your requirement directly: hard GIS buffers around residential zones, schools, hospitals, markets, water bodies, wells, floodplains, forests, airports (bird-strike), and unstable ground — excluded parcels are *unrankable*, with the exclusion reason logged for public audit. **Stage B — ML/MCDA ranking** of surviving parcels: transport cost from waste centroids, land availability, wind direction relative to settlements, groundwater depth, expansion headroom. Crucially the same engine sites *positive* infrastructure — MRFs, composting yards, e-waste units — and the platform's default recommendation is always "add processing capacity" before "add disposal capacity": the module computes, for every disposal proposal, the recycling-capacity investment that would make it unnecessary, and shows both side by side to decision-makers. Public-transparency page shows criteria and buffers for any shortlisted site. *Exit KPI:* zero new disposal capacity within buffer distances; every disposal proposal accompanied by its diversion-alternative costing.

### Phase 4 — Removal Fleet & Recycling-Unit Network

FillCast predicts every collection point's next-72 h state; RouteOpt turns predictions + fleet into daily runsheets (capacity, time windows, ward boundaries, fuel); crews get turn-by-turn runsheets with photo proof-of-pickup feeding the Passport. Dynamic dispatch: citizen bulk-pickup bookings and HotspotNet alerts inject same-day stops. Facility-location optimization proposes where the *next* MRF / e-waste unit / transfer station pays back fastest, closing the loop with Phase 3's siting engine. *Exit KPI:* ≥30 % reduction in route-km per ton; overflow complaints down 80 %; average bulk-pickup latency < 48 h.

---

## 5. Derived Use Cases (identified beyond your four — recommended additions)

1. **Citizen dump-spot reporting + HotspotNet** — geotagged photo → verified → dispatched → before/after evidence; recurrence prediction schedules preventive patrols and design fixes (lighting, bins, CCTV) at chronic spots. This is the single highest-visibility win for public trust.
2. **Legacy-dump remediation module** — inventory of existing piles, biomining/capping project tracker, so the platform manages the *stock* of past waste, not only the *flow*.
3. **Hazardous & medical waste chain-of-custody** — strictly rule-based: licensed handlers only, manifest per batch, no marketplace, regulator-visible ledger.
4. **Construction & demolition (C&D) stream** — booking + dedicated processing (crushed aggregate marketplace); C&D is usually the heaviest illegal-dumping stream and deserves its own loop.
5. **Bulk-generator compliance** — hotels, markets, apartments: mandated on-site bio-processing tracked by the platform, with FillCast-style monitoring.
6. **Repair-and-reuse directory** — before "recycle," the app offers "repair near you" for e-waste/furniture/textiles, pushing items up the hierarchy at zero municipal cost.
7. **Ward league & open data** — public dashboards per ward (segregation %, diversion %, hotspot count) — measured civic competition demonstrably moves behavior.
8. **Worker safety & dignity layer** — route fatigue limits, hazardous-exposure flags from VisionSort at booking time, and formalization of informal waste-picker collectives as first-class marketplace sellers (they are the city's existing recycling backbone; the platform should pay them better, not bypass them).
9. **Carbon/EPR accounting** — the Passport already carries the data to compute methane avoided and Extended-Producer-Responsibility credits, a real revenue line for the city.

---

## 6. Data & Integration Plan

Foundation registry (Phase 0): ward boundaries + collection points + fleet + facilities into PostGIS. Training data: start VisionSort from public datasets (TACO, TrashNet), then a 4-week local image drive (crew phones + citizen submissions, ~10–20 k labeled images) because local waste (packaging types, materials) differs from benchmark sets. IoT: ultrasonic fill sensors on a *sample* of bins first (forecasting generalizes; 100 % sensor coverage is a waste of money), GPS from existing trackers, weighbridge API. External: commodity price feeds, meteorological data (wind for siting, rain for STP inflow), land-records/GIS layers from the state portal.

**Stack:** Python (FastAPI) + PostgreSQL/PostGIS + Redis; PyTorch models served via TorchServe/ONNX; OR-Tools; MQTT (EMQX) for IoT; React/Leaflet (or MapLibre) web + PWA; role-based access; audit-log everything; open-data API for the public pages.

---

## 7. Roadmap with KPIs

**Phase 0 (6–8 weeks): registry + Passport + citizen reporting.** The un-glamorous foundation; hotspot reporting ships here for early visible wins.
**Phase 1 (3–4 months): segregation AI, credits, marketplace, ContamCam.** KPI gate as §4.
**Phase 2 (2–3 months, parallel-capable): STP IoT + FlowGuard + bio-waste loop.**
**Phase 4 before 3 (recommended sequencing): fleet intelligence** — routing pays for the platform and shrinks the "need" for disposal sites, so run it before siting studies.
**Phase 3 (2 months): SiteRank** for processing-first infrastructure; disposal only with the diversion-alternative comparison.
**Continuous:** model retraining loops (VisionSort quarterly with new local images), KPI reviews, ward expansion wave by wave — pilot 3 wards, then scale, never city-wide-big-bang.

---

## 8. Risk Register

| Risk | Severity | Mitigation |
|---|---|---|
| Citizens don't adopt segregation | High | Credits with real redemption value, dead-simple Scan-My-Waste UX, ward league, education-first enforcement |
| Marketplace has no recycler demand | High | Onboard recyclers/aggregators *before* launch; PriceSense-informed floor pricing; EPR credit pipeline as sweetener |
| Vision model wrong on local waste | Medium | Local data drive, confidence thresholds → "not sure, here's how to check", human review queue feeding retraining |
| IoT sensors vandalized/dead | Medium | Sample-based sensing + FillCast inference for the rest; ruggedized mounts |
| Siting decisions politically contested | High | Deterministic exclusions with public audit trail; processing-vs-disposal comparison mandatory; community hearings built into workflow |
| Informal sector displaced | High | §5.8: collectives as marketplace sellers with priority lanes |
| Data quality from ground ops | Medium | Photo proof-of-pickup, weighbridge cross-checks, anomaly flags on impossible tonnage |
| "AI-washing" without ops reform | High | Every phase gated on operational KPI, not model accuracy |

---

## 9. Build-with-Claude Workflow

Phase 0 is classic CRUD + GIS — one or two Claude Code sprints (registry schema, Passport ledger with append-only constraint tests, Leaflet ward map, citizen report flow). VisionSort begins as a fine-tune notebook on TACO/TrashNet with a confusion-matrix review gate before any citizen-facing deployment. RouteOpt is a clean OR-Tools module testable on synthetic wards. Keep `kpi/definitions.yaml` as the single scoreboard the dashboard, the phase gates, and the city's public page all read from — the whole system's honesty lives in that file.
