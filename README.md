# HALO-AD — Hierarchical Array Layered Optical Air-Defense (Research Framework)

**A simulation and research framework for distributed, mast-elevated, multi-zone directed-energy coverage geometry, event-driven tracking, and multi-emitter coordination.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](#license)
[![Status: Research](https://img.shields.io/badge/Status-Research%20Only-red.svg)](#scope-and-disclaimers)
[![Simulation-First](https://img.shields.io/badge/Mode-Simulation--First-orange.svg)](#scope-and-disclaimers)

---

## Scope and Disclaimers

**HALO-AD is a research and education project.** It is *not* a product, a deployable weapon system, or a controlled-technology export package. It does not include, design, or specify any actual high-energy laser, beam-forming optic capable of weaponized output, lethal effector, fire-control link, or targeting authority logic for use against persons, aircraft, or property.

What this repository **does** contain:

- A **simulation engine** for studying the *geometric coverage* of distributed directed-energy emitters arranged on masts, towers, and arrays.
- A **tracking and coordination architecture** built on PentaTrack predictive centers, event-based sensors, and mmWave radar fusion.
- A **zoning and handoff model** for partitioning a study area into districts and analyzing engagement-volume coverage as a topological / optimization problem.
- A **sensor-fusion stack** for combining LiDAR, mmWave radar, and neuromorphic event cameras into low-latency perception pipelines.
- A **planning notation** for coverage-overlap trade studies similar to cellular RF planning, sensor placement optimization, and astronomical telescope arrays.

What this repository **does not** contain and will never contain:

- High-energy laser hardware design, optics specifications above eye-safe / educational class, beam-combining schemes for damage thresholds, or kill-chain authority logic.
- Any code that issues a "fire" command to a real effector.
- Any munitions, ordnance, propellant, or lethal-mechanism design.
- Any export-controlled (ITAR / EAR / Wassenaar) technical data, dual-use specifications, or controlled-software cryptography.

All "engagement" terminology in this repository refers to **simulated geometric coverage events** in a virtual environment — analogous to a radar simulator or air-traffic visualization. Anyone wishing to extend this work toward physical hardware must independently obtain the appropriate legal, regulatory, FAA / national-aviation, and export-compliance approvals. The maintainers do not endorse, support, or enable such extension.

---

## Why This Project Exists

Modern published counter-UAS and ground-based air-defense literature consistently identifies the same set of unsolved or under-explored problems:

1. **Coverage geometry is half-solved.** Ground-mounted directed-energy emitters lose roughly the lower hemisphere of their potential engagement volume because the ground itself blocks elevation. Naval and mobile systems partially compensate by elevation, but most published academic geometry treats emitter placement as a 2D problem.
2. **Mast elevation changes the math entirely.** Lifting an emitter even a few meters off the deck restores coverage *under* the horizontal plane — meaning low-altitude crossing trajectories that would otherwise pass through dead zones become observable and trackable.
3. **Cone-to-center vs. parallel-array vs. divergent-fan layouts** each have different sweet spots in coverage density. There is no widely available open simulation tool that lets a researcher A/B these configurations against the same threat-trajectory dataset.
4. **Tracking latency dominates the engagement loop.** Conventional scanning LiDAR is too slow for fast crossing targets; event cameras and Doppler radar respond in microseconds but at lower spatial resolution. The fusion problem is geometric, not just statistical.
5. **Zoning** — partitioning a region into districts where multiple emitters can hand off a track — is essentially the same problem as cellular tower planning, but the published work does not generalize cleanly to 3D engagement volumes.

HALO-AD is a unified open simulation platform for studying these geometry, coordination, and tracking questions in a reproducible way.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      HALO-AD Research Stack                     │
├─────────────────────────────────────────────────────────────────┤
│  Scenario Layer                                                 │
│  ─────────────                                                  │
│  Region geometry • Threat trajectories • Wind / atmosphere      │
│  Zone definitions • Mast placements • Emitter configurations    │
├─────────────────────────────────────────────────────────────────┤
│  Coverage Geometry Engine                                       │
│  ─────────────────────────                                      │
│  Cone-to-center model • Parallel-array model • Divergent-fan    │
│  Mast-elevation lift analysis • Dead-zone visualization         │
│  Coverage density heatmaps • Multi-emitter overlap optimization │
├─────────────────────────────────────────────────────────────────┤
│  Perception Layer                                               │
│  ────────────────                                               │
│  Event-based vision sim • mmWave radar sim • LiDAR sim          │
│  Sensor fusion • PentaTrack predictive centers (multi-track)    │
├─────────────────────────────────────────────────────────────────┤
│  Coordination Layer                                             │
│  ──────────────────                                             │
│  Zone handoff • Emitter assignment optimization                 │
│  Latency budgeting • Failure / occlusion fallback simulation    │
├─────────────────────────────────────────────────────────────────┤
│  Output                                                         │
│  ──────                                                         │
│  Coverage reports • Topology renders • Latency histograms       │
│  Trade-study CSVs • Visualization overlays                      │
└─────────────────────────────────────────────────────────────────┘
```

No layer of this stack issues a real-world effect. Every "emit," "engage," or "track" event is a logged simulation event.

---

## Geometry Models Studied

HALO-AD's core research question: *for a given threat-trajectory distribution and a given budget of emitters, what placement geometry maximizes observability and minimizes latency?*

The framework lets you A/B four canonical geometries on the same scenario:

### A. Ground-Static (baseline)

Emitters at zero elevation, fixed positions. Approximates a ground-mounted directed-energy installation. Used as the worst-case reference: roughly half the potential engagement volume is lost to the ground plane, and any low-flyer that crosses the horizontal can hide in the lower hemisphere.

### B. Mast-Elevated

Same emitters lifted to mast height (parameterized per emitter). Recovers the lower-hemisphere coverage and shifts the dead-zone geometry from "below the device" to "directly under the mast." This is the simplest geometric upgrade and the one with the largest published-but-underexplored payoff.

### C. Cone-to-Center

Multiple mast-elevated emitters, each angled inward toward a shared focal volume. At the focal point, all emitter axes intersect — meaning a target inside that small focal volume is observable from every emitter simultaneously. Outside the focal volume, coverage becomes sparse. Ideal for protecting a single high-value asset; poor for area coverage.

### D. Parallel-Array Field

Mast-elevated emitters with parallel optical axes pointing outward across a field. Maximum area coverage with low-density observation; opposite trade from cone-to-center.

### E. Divergent-Fan (Hybrid)

Cone-to-center for the inner zone, parallel-array for the outer zone, with a tunable transition angle. The general case both pure geometries are special cases of.

The simulator computes **observability density** (how many emitters can see a given voxel), **latency-to-track** (how long after threat entry until at least N emitters have a fix), and **coverage continuity** (no track loss as the target crosses zone boundaries) for each layout.

---

## Theory Documents

The simulation framework is supported by a set of theory documents that characterize the physics and geometry the simulator models. These are intended for researchers, students, and qualified licensees who want to understand the assumptions behind the simulator's outputs.

- **`docs/theory/coverage_geometry.md`** — Earth-curvature, beam-divergence, atmospheric-attenuation, and steering-latency physics that govern engagement-volume coverage.
- **`docs/theory/steering_latency_and_field_of_view.md`** — The mechanical-vs-solid-state-vs-hybrid-array tradeoff in beam steering, and how it drives the choice of distributed-array architectures over single-emitter systems.
- **`docs/theory/sensor_fusion.md`** — Event-based vision, mmWave Doppler radar, and scanning LiDAR fusion at engagement timescales (shared with the AEGIS-MESH treatment, adapted for outdoor long-range coverage).
- **`docs/theory/zoning_model.md`** — The cellular-RF-planning analogy for 3D engagement-volume zone planning.
- **`docs/theory/future_research.md`** — The full research-domain map across all tiers, from in-scope simulation work to adjacent regulated domains documented for researchers with appropriate frameworks.

The simulator's atmospheric and coordination components are documented at the implementation level in:

- **`software/simulator/atmospheric_models.md`** — Thermal blooming, aerosol scattering, beam-divergence parameters used by the simulation.
- **`software/coordination/zone_handoff.md`** — District handoff logic for multi-zone scenarios.

---

## Zoning Model

Inspired by cellular planning, the simulator partitions a region into **districts** (large areas) and **zones within districts** (smaller engagement volumes). Each zone is associated with one or more masts; every voxel inside a zone is observable from at least N masts (configurable). Zones overlap at their boundaries to enable continuous-track handoff between mast clusters as a target moves.

The zoning solver is treated as a weighted geometric set-cover problem:

- Inputs: region bounds, mast budget, target observability `N`, threat-trajectory distribution.
- Output: mast placements, per-mast orientations, and zone polygons that maximize observability while minimizing dead-zone fraction.

This is the same family of optimization used in cellular RF planning, sensor-network coverage, and astronomical telescope-array siting.

---

## Tracking Stack (Perception)

HALO-AD integrates [PentaTrack](https://github.com/RebornBeat/PentaTrack) (the predictive-center bounding-box framework) as its trajectory predictor and adds three simulated sensor types:

- **Event-based vision** (microsecond-latency motion edges; ideal for fast crossing targets).
- **mmWave Doppler radar** (continuous velocity readout; robust to fog, smoke, rain).
- **Scanning LiDAR** (high spatial resolution but scan-cycle limited).

Each emitter mast in the simulator is treated as a sensing node, *not* an effector node. The fusion pipeline produces a multi-mast track on every threat trajectory and feeds the tracks into the coverage analyzer. PentaTrack's velocity-weighted prediction tree, drift analysis, and object-type awareness all participate.

---

## Repository Layout

This project follows the Full Stack Hardware Documentation Framework, but with **most hardware folders intentionally empty / placeholder**, since this is simulation-first research.

```
halo-ad/
├── docs/
│   ├── guides/
│   ├── theory/
│   │   ├── coverage_geometry.md
│   │   ├── steering_latency_and_field_of_view.md
│   │   ├── zoning_model.md
│   │   ├── mast_elevation_analysis.md
│   │   ├── sensor_fusion.md
│   │   └── future_research.md
│   ├── api/
│   └── assets/
├── hardware/                  # PLACEHOLDER — no physical emitter design
│   └── README.md              # Explicit "out of scope" notice
├── firmware/                  # PLACEHOLDER
│   └── README.md
├── software/
│   ├── simulator/
│   │   └── atmospheric_models.md
│   ├── geometry/              # Coverage geometry analysis
│   ├── perception/            # Sensor fusion + PentaTrack integration
│   ├── coordination/
│   │   └── zone_handoff.md
│   ├── viz/                   # 3D coverage visualization
│   ├── cli/                   # Scenario runner
│   ├── sdk/                   # Python API for custom scenarios
│   └── protocol/
│       └── protocol_spec.md   # Internal sim-event protocol only
├── mechanical/                # PLACEHOLDER
│   └── README.md
├── production/                # NOT APPLICABLE
├── legal/
│   ├── compliance.md          # Export-control / dual-use posture
│   ├── research_ethics.md
│   ├── export_control_posture.md
│   └── tos_compliance.md
├── media/
├── README.md
├── LICENSE
└── CHANGELOG.md
```

The `hardware/`, `firmware/`, `mechanical/`, and `production/` directories each contain a single `README.md` stating: *"This directory is intentionally empty. HALO-AD is a simulation and research framework. Physical hardware is out of scope and will not be designed, specified, or distributed in this repository."*

---

## Quick Start

```bash
git clone https://github.com/<user>/halo-ad.git
cd halo-ad/software
pip install -e .

# Run the baseline ground-static coverage study
halo-ad run scenarios/baseline_ground.yaml

# Compare against mast-elevated and cone-to-center
halo-ad run scenarios/mast_elevated.yaml
halo-ad run scenarios/cone_to_center.yaml

# Generate a coverage trade-study report
halo-ad report --inputs runs/ --output report.html
```

All scenarios are virtual. Outputs are HTML reports, 3D coverage renders, and CSV trade-study data.

---

## Roadmap

- **Phase 1 (current):** coverage geometry simulator + zoning optimizer.
- **Phase 2:** full PentaTrack integration with multi-mast track fusion.
- **Phase 3:** event-camera + mmWave + LiDAR fused-perception pipeline (still simulated).
- **Phase 4:** scenario library — published threat-trajectory datasets for reproducible research.

There is no Phase N for hardware. By design.

---

## License

MIT for code. Documentation under CC BY 4.0. See `LICENSE` and `legal/compliance.md`.
