# Steering Latency and Field of View — The Tradeoff That Drives Distributed-Array Architectures

**Project:** HALO-AD
**Domain:** Beam-steering physics and array-architecture tradeoffs

## 1. The Core Tradeoff

Any system that points a beam—whether for sensing, communication, or directed-energy applications—faces a fundamental tradeoff between three properties that cannot be jointly maximized:

- **Field of View (FoV):** The angular range the system can address at any given moment or within a finite reconfiguration time.
- **Steering Latency:** The time delay between the decision to point at a new coordinate and the moment the beam is stable and usable at that coordinate.
- **Beam Quality at Range:** The focus, divergence, and pointing precision (jitter) of the beam at typical engagement distances, which determines effective power density and resolution.

Architectures typically optimize for two of these three properties, requiring the third to be sacrificed or mitigated through system-level complexity. This document characterizes the architectural families HALO-AD's simulator supports and the geometry that drives the choice between them.

## 2. Mechanical Steering

The traditional architecture: a beam emitter is mounted on a rotating platform (gimbal, turret, or turntable) that physically points the emitter at the target. Examples include surveillance radar dishes, traditional searchlights, and gimbal-mounted optical systems.

### 2.1 Characteristics

- **Field of View.** Effectively continuous 360° azimuth and near-hemispherical in elevation. The mechanical mount can, in principle, point anywhere within its mechanical limits.
- **Steering Latency.** Bounded by physics: motor torque, the moment of inertia of the moving mass (mirror, aperture, housing), and the angular distance the system must slew.
    - **Slew Rate ($\omega$):** The maximum angular velocity. For heavy apertures, this is often tens of degrees per second.
    - **Settling Time ($t_{settle}$):** After the slew, the system vibrates. It takes finite time for the gimbal to stabilize so the beam is precise enough for use. This often exceeds the slew time itself.
- **Beam Quality.** Excellent. The aperture can be physically large because the architecture does not impose a size constraint related to steering speed. Large apertures support tight beam focus ($\theta \propto 1/D$) and excellent range performance.

### 2.2 Failure Modes

The primary failure mode is **temporal latency**.
*   **The "Crossing Target" Problem:** A target appearing at bearing $0^\circ$ while the system is pointed at $180^\circ$ requires a full reversal. At a slew rate of $60^\circ/s$, this takes 3 seconds. In that time, a threat moving at Mach 1 has traveled $\sim 1$ km.
*   **Sequential Engagement Limit:** The system cannot engage multiple dispersed targets simultaneously. It must slew-engage-slew, creating a queue of threats that may overwhelm the system in saturation attacks.

### 2.3 Where It Wins

Mechanical steering wins when the threat distribution is sparse (one or few targets at a time, addressed sequentially) and threat velocities are low relative to the slew rate. It is also preferred for applications requiring massive apertures (long-range astronomy or high-power directed energy) where solid-state alternatives cannot yet handle the power density.

## 3. Solid-State Steering

Solid-state architectures steer the beam without bulk mechanical movement. The two dominant families are:

- **Optical Phased Arrays (OPA):** A two-dimensional grid of emitters with electronically controlled phase relationships. By adjusting the relative phase of each emitter, constructive interference is steered to a chosen direction. This is the optical analogue of phased-array radar, miniaturized into integrated photonics.
- **MEMS Mirror Arrays:** Micro-electromechanical mirrors that tilt under electrical control. While technically "mechanical," the masses are microscopic, allowing for kHz steering rates.

### 3.1 Characteristics

- **Field of View.** Limited by diffraction physics and emitter spacing.
    - **The Diffraction Limit:** To steer light without generating "grating lobes" (unwanted secondary beams), emitters must be spaced at sub-wavelength distances. This manufacturing constraint limits the effective steering cone to typically $\pm 30^\circ$ to $\pm 45^\circ$ from boresight.
    - **Result:** A single solid-state aperture cannot see "behind" itself. It has a Field of Regard (FoR) limited to a forward cone.
- **Steering Latency.** Microseconds to milliseconds. The steering rate is bounded by control electronics and capacitance, not mechanical inertia. The beam can "jump" instantaneously between points within its FoV.
- **Beam Quality.** Variable.
    - **OPAs:** Beam quality depends on the fill factor and phase precision. Gaps between emitters cause power loss and sidelobes.
    - **MEMS:** Beam quality depends on the mirror flatness and tilt angle limits.

### 3.2 Failure Modes

The primary failure mode is **geometric coverage**.
*   **The "Blind Spot" Problem:** A solid-state sensor facing North cannot detect a threat approaching from the South. If the threat maneuver puts it outside the $\pm 45^\circ$ cone, the track is lost immediately.
*   **Power Scaling:** OPA apertures are currently limited in the total optical power they can handle compared to large reflective mirrors used in mechanical systems.

### 3.3 Where It Wins

Solid-state steering wins when the threat distribution is dense (multiple simultaneous threats), the threats are fast (latency dominates), or the application requires "random access" scanning (jumping between points of interest without traversing the intervening space).

## 4. The Hybrid Distributed Array

Neither mechanical nor solid-state architectures alone solve the joint problem of full-sky coverage (Mechanical strength) with low latency (Solid-State strength). The Hybrid Distributed Array is the architectural solution HALO-AD simulates.

This architecture distributes multiple solid-state apertures across separate platforms or mounting points, each covering a specific sector. Their union covers the full $360^\circ$ volume.

### 4.1 Characteristics

- **Field of View.** Full 360° achieved by the union of $N$ narrow-FoV apertures.
    - Example: 6 apertures, each with $60^\circ$ FoV, arranged hexagonally, provide seamless $360^\circ$ azimuth coverage.
- **Steering Latency.** Low. Within any aperture's sector, the latency is the solid-state latency (microseconds). Handoff between sectors is managed by software coordination.
- **Beam Quality.** High. Each aperture can be high-quality. Furthermore, adjacent apertures can theoretically coherently combine their beams to increase effective aperture size (and thus range/resolution) on a single target, though this requires precise phase-locking.

### 4.2 The Costs

The hybrid architecture buys its joint properties at the cost of **Complexity**:
- **Hardware Count:** $N$ apertures, $N$ power supplies, $N$ cooling loops.
- **Synchronization:** All apertures must share a precise coordinate frame and time base.
- **Coordination Logic:** The system must manage "Sector Handoff." When a target moves from Sector A (Aperture 1) to Sector B (Aperture 2), the track state must be transferred flawlessly. This is the core coordination problem simulated in HALO-AD.

### 4.3 Mast Elevation as a Force Multiplier

The hybrid distributed array compounds with mast elevation. Each aperture mounted at mast height covers a larger ground-projected volume than the same aperture mounted at ground level because the geometry of the aperture's cone intersects more of the engagement volume from the elevated position.

Furthermore, elevation mitigates the solid-state "blind spot" problem:
*   **Ground Level:** The rear blind spot is a critical vulnerability (threats can approach from behind).
*   **Mast Level:** The "rear" blind spot is mitigated by geometry. A threat "behind" the mast is often visible to a neighboring mast. The shadow cone directly beneath the mast is the only true blind spot, which is small relative to the total coverage volume.

The simulator parameterizes both jointly. A research user can vary aperture count, aperture FoV, mast height, and mast count separately.

## 5. Where the Simulator Quantifies the Tradeoff

HALO-AD's simulator quantifies steering-latency-vs-FoV tradeoffs through three primary outputs:

- **Time-to-First-Track ($t_{ft}$):** Measures the latency budget from threat entry to established track.
    - *Mechanical Scenario:* High variance. Depends heavily on where the gimbal was pointing when the threat appeared.
    - *Hybrid Scenario:* Low variance. The nearest aperture is always "looking" in that sector.
- **Track Continuity ($C_{track}$):** The probability that a track is maintained without dropout during maneuvers.
    - *Mechanical:* Dropouts occur during high-speed slews if the target moves faster than the gimbal can track.
    - *Hybrid:* Dropouts occur only during "Sector Handoff" failures (software/coordination latency).
- **Coverage Density ($\rho_{cov}$):** The fraction of the volume observable by at least $N$ apertures.
    - *Hybrid architectures* allow for high overlap (redundancy) in critical sectors (e.g., the "front" of a moving vehicle or the center of a district) without sacrificing latency.

These outputs are CSV-and-visualization deliverables suitable for academic comparison.

## 6. Civilian Transfer

The architectural families modeled here have direct analogues in civilian domains:

- **Autonomous Vehicles:**
    - *Mechanical:* Spinning LiDAR (high resolution, scan latency $\sim 100$ms).
    - *Solid-State:* Flash LiDAR or MEMS LiDAR (lower resolution, instant update, limited FoV).
    - *Hybrid:* Vehicles using multiple solid-state LiDARs (e.g., one front, one rear) to achieve $360^\circ$ coverage with low latency.
- **Multi-Camera Sports Tracking:**
    - *Mechanical:* PTZ cameras for following players (high zoom, slow pan).
    - *Solid-State:* Fixed wide-angle cameras (low zoom, instant capture).
    - *Hybrid:* Arrays of fixed cameras stitched together to cover a stadium.
- **Cellular Networks:** Phased-array base stations (Solid-State) with sectorized coverage patterns, handoff protocols, and frequency reuse constraints.
- **Telescope Arrays:** Distributed apertures using interferometry to合成 a large aperture, sharing the coordination challenges (phase referencing) of the hybrid architecture.

The simulator's algorithms and outputs apply to each. The `examples/civilian/` directory in the repository will demonstrate at least one transfer per architectural family.
