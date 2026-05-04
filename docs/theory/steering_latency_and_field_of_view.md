# Steering Latency and Field of View — The Tradeoff That Drives Distributed-Array Architectures

**Project:** HALO-AD
**Domain:** Beam-steering physics and array-architecture tradeoffs

## 1. The Core Tradeoff

Any system that points a beam — sensor, communication, or otherwise — faces a fundamental tradeoff between three properties that cannot be jointly maximized:

- **Field of view** (FoV): the angular range the system can address at all.
- **Steering latency**: how quickly the system can move its beam from one direction to another within its FoV.
- **Beam quality at range**: the focus, divergence, and pointing precision of the beam at typical engagement distances.

Architectures choose two of the three; the third is sacrificed or worked around through other means. This document characterizes the architectural families HALO-AD's simulator supports and the geometry that drives the choice between them.

## 2. Mechanical Steering

The traditional architecture: a beam emitter is mounted on a rotating platform (gimbal, turret) that physically points the emitter at the target. Examples include radar dishes, traditional searchlights, and gimbal-mounted optical systems.

### 2.1 Characteristics

- **FoV.** 360° azimuth, often nearly hemispherical in elevation. The mechanical mount can in principle point anywhere.
- **Steering latency.** Limited by motor torque, mirror or aperture mass, and the angular distance to slew. For meaningful aperture sizes, slew rates are tens to hundreds of degrees per second; full-180° slews take fractions of a second to several seconds.
- **Beam quality.** Excellent. The aperture can be physically large because nothing about the architecture imposes a small-aperture constraint; large apertures support tight beam focus and good range.

### 2.2 Failure Modes

The slew time bounds the system's ability to track fast crossing targets or to engage multiple targets distributed across the FoV. A target appearing on bearing 0° while the system is pointed at bearing 180° has up to a full slew time before the system can observe it; during that interval the target can traverse a substantial fraction of the engagement volume.

### 2.3 Where It Wins

Mechanical steering wins when the threat distribution is concentrated (one target at a time, addressed sequentially) and the threat velocity is low relative to slew rate. For ground-based wide-area surveillance with mostly slow targets, mechanical steering's full FoV and excellent beam quality dominate.

## 3. Solid-State Steering

Solid-state architectures steer the beam without moving mass. The two dominant families are:

- **Optical phased arrays (OPA).** A two-dimensional grid of emitters with electronically controlled phase relationships. The relative phasing produces constructive interference in a chosen direction; changing the phasing changes the direction. Borrowed conceptually from radio-frequency phased-array radar (which has been mature since the 1960s) and miniaturized into integrated photonics.
- **MEMS mirror arrays.** Micro-electromechanical mirrors that tilt under electrical control. Each mirror is small and low-mass, so steering is fast even though it remains technically "mechanical."

### 3.1 Characteristics

- **FoV.** Limited. Practical OPAs and MEMS arrays achieve cones of ±30° to ±45°, or roughly 90° hemispherical coverage at the upper end of current engineering. Full 360° is not achievable in a single solid-state aperture.
- **Steering latency.** Microseconds to milliseconds. No moving mass means the steering rate is bounded by control electronics, not mechanical inertia.
- **Beam quality.** Variable. OPA aperture sizes are limited by integrated-photonics manufacturing constraints; MEMS apertures by mirror geometry. Beam quality at range is consequently more constrained than for mechanical steering, though the published research is rapidly advancing this front.

### 3.2 Failure Modes

The limited FoV is the fundamental issue. A solid-state aperture cannot see behind itself. A threat appearing outside the cone is invisible to that aperture, regardless of how fast the aperture can steer within its cone.

### 3.3 Where It Wins

Solid-state steering wins when the threat distribution is broad (multiple targets simultaneously) within a constrained FoV, the threats are fast (latency dominates), or the application requires sustained tracking through fast maneuvers.

## 4. The Hybrid Distributed Array

Neither mechanical nor solid-state alone solves the joint problem of full-FoV coverage with low steering latency. The hybrid architecture distributes multiple solid-state apertures across separate platforms, each covering a portion of the total FoV, with their union covering the full required volume.

### 4.1 Characteristics

- **FoV.** Full 360° (or whatever the deployment geometry requires) achieved by union of multiple narrow-FoV apertures.
- **Steering latency.** Low — within each aperture's cone, the latency is the solid-state-aperture latency.
- **Beam quality.** As good as the solid-state-aperture beam quality; potentially better if multiple apertures coherently combine.

### 4.2 The Costs

The hybrid architecture buys its joint properties at the cost of:

- **Hardware count.** N apertures rather than 1.
- **Synchronization.** N apertures must share a coordinate frame, time base, and (for coherent combination) phase reference.
- **Coordination logic.** N apertures must agree on which addresses which target, with handoff as targets cross aperture boundaries.
- **Power and infrastructure.** N apertures require N power feeds and N data links.

The complexity is not free. The simulator quantifies the tradeoffs against the threat-trajectory-distribution scenarios in the scenario library.

### 4.3 Mast Elevation as a Force Multiplier

The hybrid distributed array compounds with mast elevation: each aperture mounted at mast height covers a larger ground-projected volume than the same aperture mounted at ground level, because the geometry of the aperture's cone intersects more of the engagement volume from the elevated position. The two architectural choices — distributed solid-state arrays and mast elevation — are independent and combine multiplicatively.

The simulator parameterizes both jointly. A research user studying coverage of a particular region against a particular threat-trajectory distribution can vary aperture count, aperture FoV, mast height, and mast count separately and observe the coverage outcome.

## 5. Where the Simulator Quantifies the Tradeoff

HALO-AD's simulator quantifies steering-latency-vs-FoV tradeoffs through three primary outputs:

- **Time-to-first-track.** For each threat trajectory in the scenario, how long after the threat enters the engagement volume does the array produce its first track? Mechanical-steering scenarios produce longer time-to-first-track for crossings; distributed-solid-state scenarios produce shorter time-to-first-track at the cost of more hardware.
- **Track continuity.** Does the array maintain track through the entire engagement volume, or does the track drop during slew or handoff? Mechanical-steering scenarios produce dropouts during slew; distributed-solid-state scenarios produce dropouts during inter-aperture handoff (which is generally faster but not zero).
- **Coverage density.** What fraction of the engagement volume is observable from at least N apertures? Distributed-solid-state architectures can deliver high coverage density without sacrificing latency; mechanical-steering architectures cannot.

These outputs are CSV-and-visualization deliverables suitable for academic comparison.

## 6. Civilian Transfer

The same architectural family appears in many civilian domains:

- **Multi-camera sports tracking.** Mechanical PTZ cameras (high FoV, slow), wide-FoV fixed cameras (no steering, fixed coverage), or distributed multi-PTZ arrays (the analogue of the hybrid distributed approach).
- **Wildlife monitoring.** Distributed sensor networks with the same coverage-vs-latency tradeoffs.
- **Cellular base-station planning.** Phased-array antennas with similar FoV constraints; multi-base-station coverage with similar handoff problems.
- **Telescope arrays.** The same multi-aperture coverage problem solved at much longer baselines and slower timescales.
- **Multi-robot inspection.** Distributed inspection of large structures; same coverage and handoff problems at the multi-robot level.

The simulator's algorithms and outputs apply to each. The `examples/civilian/` directory in the repository will demonstrate at least one transfer per architectural family.
