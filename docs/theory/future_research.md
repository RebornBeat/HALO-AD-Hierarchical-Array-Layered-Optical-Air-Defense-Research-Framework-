# Future Research Areas — HALO-AD

## Purpose of This Document

This document maps the research domains adjacent to and downstream of HALO-AD's published simulation framework. Its audience is researchers, students, and qualified licensees who want to understand where the published work sits in the broader field, what open research questions surround it, and where the perception/coordination/geometry layer that HALO-AD does develop would plug in to those domains.

The document is licensed MIT alongside the rest of the repository. Nothing in this document is a build specification, a parameter table, or operational guidance. Everything in this document is at the level of academic survey work — naming the domains, describing the architectures conceptually, citing the kinds of precedents that exist in the open literature, and identifying the open questions. Researchers extending into any of these domains are responsible for their own institutional, regulatory, ethical, and legal frameworks; HALO-AD's repository is not such a framework.

## How to Read the Tier Structure

Areas are organized by **regulatory and technical hardness**: lower tiers are closer to civilian-grade engineering with established academic literature and active commercial deployment; higher tiers are progressively more regulated, more speculative, or both. Tiers are not a roadmap. The maintainers do not endorse pursuit of any specific tier and do not implement any tier beyond what is in the published repository.

---

## Tier 1 — Geometric Coverage and Sensor Placement (In Scope)

This is what HALO-AD's repository implements. Distributed-emitter coverage geometry, mast-elevation lift analysis, cone-to-center vs. parallel-array vs. divergent-fan layouts, multi-zone partitioning, and zone-handoff coordination are all developed in `software/geometry/` and `software/coordination/`. The research contribution is genuine on its own: there is no widely-available open simulation tool today for A/B comparison of distributed directed-energy emitter geometries against the same threat-trajectory dataset, and the cellular-RF-planning analogy for 3D engagement-volume zone planning is, to the maintainers' knowledge, not yet generalized in published open work.

The civilian transfer of Tier 1 is concrete and substantial: the same algorithms apply to telescope-array siting, sensor-network coverage planning, distributed-camera surveillance-network planning (in non-privacy-sensitive contexts like wildlife monitoring and traffic), and multi-base-station cellular planning. The repository's `examples/` directory will, over time, include civilian-transfer examples for each of these.

The published-research questions Tier 1 addresses include: optimal mast count and placement for a given threat-trajectory distribution; sensitivity of coverage to atmospheric and environmental conditions; degradation graceful behavior when emitters fail or are occluded; and the algorithmic complexity of the multi-emitter zone-cover problem.

---

## Tier 2 — Perception and Tracking at Engagement Timescales

The perception layer of HALO-AD — multi-sensor fusion combining event-based vision, mmWave Doppler radar, and scanning LiDAR, with PentaTrack as the predictive substrate — is a research domain in its own right. The open questions include:

**Sensor-modality complementarity.** Event cameras and Doppler radar both report change, but at different physical scales (photonic events vs. RF Doppler shifts) and with different failure modes (event cameras fail in static scenes; Doppler radar fails on radial-zero-velocity motion). The fusion question is geometric, not just statistical: where in the joint state space does each modality contribute information, and how does the fusion architecture exploit complementarity rather than averaging it away?

**Latency-accuracy tradeoff.** Scanning LiDAR provides spatial resolution that event cameras lack, but at scan-cycle latency that fast crossing targets do not respect. Hybrid architectures that use event-camera detections as cues for adaptive LiDAR scanning are a published research thread; HALO-AD's simulator is suitable for studying them at the geometric level.

**Multi-mast track fusion.** When N masts each carry a perception stack, the published track-fusion literature (covariance intersection, federated Kalman filtering, multi-hypothesis tracking) provides a starting point but does not address the specific case where each mast's view of the same target has substantially different geometric properties due to elevation, range, and occlusion. The PentaTrack predictive-center field provides a natural mechanism for representing the multi-mast joint prediction, and research using HALO-AD's simulator can quantify how much information is available in the joint center distribution beyond what any single mast provides.

**Adversarial perception.** Targets that intentionally degrade perception — through low-radar-cross-section design, decoy releases, swarm tactics, or deliberate occlusion — are a published academic research subject in counter-UAS literature. HALO-AD's simulator can model these adversarial scenarios as scenario-library entries, and the perception-layer research question becomes: which fusion architectures are robust against which adversarial tactics?

This tier is in scope for HALO-AD development. The simulator and the perception layer are the published research contribution; researchers extending the perception layer with new fusion architectures, new sensor models, or new adversarial scenarios are encouraged to do so within the repository.

---

## Tier 3 — Multi-Emitter Coordination and Effector Assignment

When a defended volume contains multiple potential threats and multiple emitter masts, the assignment problem — which mast addresses which target, in what order, with what confidence requirement — is a research domain shared with multi-robot task allocation, multi-cobot coordination, multi-camera sports tracking, and multi-vehicle dispatch. The published literature includes the Hungarian algorithm and its variants for one-shot assignment, auction algorithms for distributed assignment, market-based coordination, and learned policies for sequential assignment under uncertainty.

The HALO-AD-specific open questions include:

**Multi-center engagement on a single target.** This is the same problem TALON-MESH addresses for arrays: PentaTrack produces a probability-weighted distribution of predicted future positions for each target. An emitter array, or a set of independently aimable emitter elements on a single mast, can address several of the highest-weight predicted centers simultaneously to maximize the probability that at least one element addresses the target's actual future position. The research question is the optimization formulation — weighted set-cover-with-budget over the center distribution — and its computational complexity, approximability, and online-version performance.

**Latency-budgeted coordination.** Real systems operate under hard latency budgets: detect, track, fuse, decide, command, traverse, observe-effect. The coordination layer needs to know the budget and allocate slack across stages. This is published research in real-time systems, but the specific application to multi-emitter coordination with PentaTrack-style predictive centers is open.

**Failure and degradation.** When a mast becomes unavailable (occlusion, fault, environmental degradation), the coordination layer needs to redistribute responsibility without observable degradation in coverage. Graceful-degradation policies are research subjects in distributed systems and in cellular handoff, and the analogous question for HALO-AD's coordination layer is open.

**Hierarchical zoning.** Districts, zones within districts, and sub-zones within zones form a hierarchy that mirrors the cellular-RF-planning macro/micro/pico-cell structure. The coordination layer's policy at each hierarchy level — which zone retains a track, when handoff is initiated, how priorities are merged across levels — is open research.

This tier is in scope for HALO-AD. Civilian transfer is direct: the same coordination algorithms apply to multi-cobot pick-and-place, multi-camera sports tracking, and multi-drone agricultural inspection. The repository's `examples/civilian/` directory will demonstrate these transfers.

---

## Tier 4 — Directed-Energy Effector Research (Out of Scope, Mapped for Researchers with Appropriate Frameworks)

Directed-energy effector research — the design and characterization of high-energy lasers, microwave systems, and other directed-energy mechanisms — is a published academic and industrial domain with active publication in IEEE journals, SPIE proceedings, and defense-industry conferences. The mature operational systems (Iron Beam, HELIOS, MEHEL, ODIN, Tactical High-Energy Laser, Boeing's Compact Laser Weapons System, and their international counterparts) are openly described at the architecture level in the published literature.

The published architecture, at survey-paper depth, divides into:

**Beam generation.** Solid-state lasers (fiber lasers, slab lasers, thin-disk lasers), gas lasers (chemical oxygen-iodine, deuterium fluoride — largely retired from current research), and free-electron lasers each have published characteristics. Solid-state fiber lasers are the dominant current research thread because of their wall-plug efficiency, beam quality, and scalability through coherent or spectral combination.

**Beam combination.** Coherent beam combination and spectral beam combination are the two published approaches to scaling beyond single-aperture power limits. Coherent combination requires phase locking; spectral combination uses wavelength-division multiplexing. Both are active research domains.

**Beam control.** Adaptive optics for atmospheric correction, fast-steering mirrors for fine pointing, beam-director optics for coarse pointing. Adaptive optics borrows directly from astronomical-telescope literature and has become extensively published.

**Atmospheric propagation.** Thermal blooming, turbulence, scattering, absorption, and aerosol interaction. Each has well-developed models in the published atmospheric-physics literature; the SCALAR and HELEEOS codes are examples of published propagation modeling tools (the parameters used by such tools at sensitive thresholds are not in the open literature, but the model structures are).

**Effects on targets.** Material interaction with directed energy is published in the materials-science and laser-matter interaction literature: melting, ablation, thermal stress, and structural failure of common materials at characterized power densities. The published thresholds for civilian materials (cutting steel, marking plastics, drilling) are extensive; the published thresholds for hardened or specialized targets are not.

The **HALO-AD-specific** open research question for this tier is *only* the perception/coordination/geometry layer: HALO-AD's simulator could in principle produce per-emitter aim points and per-target prioritization that would feed a real directed-energy effector's beam-control loop. The repository does not implement that handoff, does not specify the beam-control loop, and does not characterize effector parameters at any threshold beyond eye-safe educational classes. Researchers and licensees with the institutional, regulatory, and safety frameworks to pursue effector research are the audience for the published academic and industrial directed-energy literature, not this repository.

For the perception layer's own development, the eye-safe threshold is the relevant boundary: HALO-AD's perception simulator models LiDAR-class emissions at unregulated wavelengths and powers, and the coverage-geometry analyzer treats emitter geometry as parameterized without specifying emitter power. This is sufficient to study coverage, coordination, and tracking, which are the repository's contributions.

---

## Tier 5 — Adversarial and Counter-Counter Research

A defended volume's adversaries adapt. The published counter-counter-UAS and counter-counter-air-defense literature addresses:

**Decoy release.** Targets that release decoys to attract sensor attention. Modeled in HALO-AD's scenario library as multi-target scenarios where some targets are decoys with characteristic signatures.

**Swarming.** N-on-M engagement saturation. The coordination layer's open research question is how to allocate limited emitter capacity against larger N, including the question of when to engage at all versus when to let some fraction of a swarm pass.

**Maneuvering.** Targets that execute aggressive maneuvers to defeat tracking. PentaTrack's drift analysis and adaptive-drift extensions are designed to handle this; the open research question is how aggressively a target can maneuver before tracking degrades, as a function of sensor stack and fusion architecture.

**Sensor-domain attack.** Targets or third parties that emit signals designed to confuse the perception layer — chaff, RF decoys, optical dazzlers aimed at the sensor mast. The published electronic-warfare literature is the relevant precedent; HALO-AD's perception layer is a natural research substrate for studying robustness.

**Networked-attack scenarios.** Multiple coordinated attack platforms with shared situational awareness among themselves. Game-theoretic modeling and multi-agent reinforcement learning are published research domains; HALO-AD's simulator supports scenario-level scripting that can drive coordinated adversary scenarios.

This tier is in scope for HALO-AD as scenario-library and perception-layer research. The civilian transfer is direct: adversarial robustness in multi-camera sports tracking against players who deliberately occlude, in multi-cobot coordination against equipment failures and unexpected loads, and in multi-vehicle dispatch against coordinated demand spikes.

---

## Tier 6 — Hypersonic and Non-Ballistic Threat Classes

The published threat-classification literature distinguishes ballistic threats (predictable parabolic trajectories), maneuvering threats (unpredictable mid-course maneuvers), low-altitude cruise threats (terrain-following), and hypersonic threats (Mach 5+ velocities with maneuvering capability). Each class has characteristic detection, tracking, and engagement properties.

For perception research:

**Hypersonic threats** present a tracking-rate problem that exceeds standard scanning-LiDAR cycle times by orders of magnitude. Event cameras and high-PRF radar are the published candidate sensors. The PentaTrack adaptive-drift extension is conceptually appropriate; the open research question is whether the prediction tree can keep up at the relevant timescales.

**Low-altitude cruise threats** present a coverage-geometry problem that mast elevation directly addresses. The open research question is how many masts at what spacing are required for continuous track of a low-flyer crossing a defended volume.

**Maneuvering threats** present a prediction-confidence problem. PentaTrack's velocity-weighted prediction degrades gracefully under aggressive maneuvers but cannot anticipate maneuvers that have no precedent in the velocity history. The open research question is how predictive-center systems should handle maneuver-classification — recognizing that a maneuver is in progress and shifting from predictive tracking to reactive tracking.

This tier is research-only. HALO-AD's simulator can model these threat classes as scenario-library entries and the perception/coordination layers can be characterized against them. The repository does not implement, characterize, or specify any actual response system at any threat class.

---

## Tier 7 — Atmospheric and Environmental Research

Directed-energy and electro-optical sensing are both atmosphere-limited. The published research domains include:

**Thermal blooming** — beam-induced refractive-index changes in the atmospheric column that distort the beam itself.
**Turbulence** — small-scale refractive-index variations from temperature mixing.
**Aerosol scattering** — Rayleigh and Mie scattering by particulates.
**Absorption bands** — wavelength-specific absorption by water vapor, CO2, and other atmospheric constituents.

Adaptive-optics literature borrowed directly from astronomical telescopes addresses several of these. The HALO-AD-specific open question is how propagation effects interact with the *coverage* analysis: a coverage map that ignores propagation overstates the available engagement volume in degraded conditions. Coupling propagation models into the coverage analyzer is open research.

This tier is in scope for HALO-AD's simulator. Open atmospheric-physics models exist (LOWTRAN, MODTRAN, and successors at unrestricted precision) and can be integrated.

---

## Tier 8 — Networked and Command-and-Control Research

Multi-mast, multi-zone, multi-district systems are distributed systems. The published research domains include:

**Distributed consensus** under partial failures and adversarial conditions (Byzantine fault tolerance, Raft, Paxos, blockchain-style consensus where appropriate).
**Real-time messaging** under hard latency budgets (DDS, RTPS, ROS2's underlying middleware).
**Cybersecurity** of command-and-control links (authenticated and encrypted command channels, replay-attack resistance, key management at scale).
**Human-machine interface** for operator oversight, especially under high-tempo conditions.
**Authority delegation** policies — who can authorize what at which zone level under which conditions, including the published lethal-autonomy-policy literature on human-in-the-loop and human-on-the-loop designs.

The HALO-AD-specific open question is the multi-zone command-and-control architecture for the geometry/perception/coordination stack the repository develops. The repository's `coordination/` module is intended as a research substrate for this work.

This tier is in scope. Civilian transfer is broad: multi-cobot coordination cybersecurity, multi-vehicle-dispatch authority delegation, multi-camera sports-tracking real-time messaging.

---

## Tier 9 — Speculative and Long-Horizon

Programmable-matter beam directors. Metasurface adaptive optics. Quantum-radar perception. Photon-counting LiDAR at ranges currently impractical. Each is published in the academic literature at varying maturity; none is fielded.

The HALO-AD-specific question for any of these is whether the geometric coverage analyzer's assumptions still hold when the underlying physics changes. The repository's simulator is parameterized enough that researchers can substitute new sensor and emitter models as the technology matures.

This tier is documented for completeness. The maintainers do not develop it.

---

## Summary

HALO-AD develops Tiers 1, 2, 3 (in their geometric/algorithmic dimensions), 5 (as scenario-library and perception robustness), 7 (as atmospheric coupling into the coverage analyzer), and 8 (as the coordination/messaging substrate). Tiers 4, 6, and 9 are documented as adjacent research domains for which HALO-AD's published layers would provide input but which the repository does not itself implement.

All published material is MIT-licensed. Researchers and licensees with the appropriate institutional, regulatory, and safety frameworks may extend in any direction; the repository does not.
