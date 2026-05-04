# Zoning Model — Cellular-RF-Planning Analogy for 3D Engagement-Volume Partitioning

**Project:** HALO-AD
**Domain:** Multi-zone partitioning, hierarchical coverage, and zone-management policy

## 1. Purpose

This document specifies the zoning model HALO-AD uses to partition large regions of interest into smaller, manageable engagement zones. The model is structured around the cellular-network-planning analogy — treating zone-boundary handoff as analogous to cell-boundary handoff in cellular telephony — but extended to three dimensions and to the heterogeneous sensor-modality regime of distributed outdoor coverage.

The cellular-RF-planning literature is mature and openly published. HALO-AD's contribution is the adaptation of the established 2D cellular framework to the 3D engagement-volume context, with explicit handling of vertical zone boundaries (altitude bands), diagonal-trajectory crossings, and multi-sensor-modality coverage.

## 2. The Hierarchy

A region of interest is partitioned hierarchically:

- **Districts.** Large geographic regions, kilometers to tens of kilometers in horizontal extent. Each district is responsible for coverage of a defined volumetric region (typically the airspace and ground surface within its horizontal boundary, up to a defined altitude ceiling).
- **Zones within districts.** Sub-regions of districts, hundreds of meters to kilometers in horizontal extent. Each zone is associated with one or more masts; coverage of the zone is the responsibility of those masts.
- **Sub-zones within zones.** Optional further partitioning, tens to hundreds of meters in horizontal extent. Used in scenarios with very high target density or with high-value point assets requiring concentrated coverage.

The hierarchy mirrors the macro-cell / micro-cell / pico-cell structure of cellular networks. The ratio between hierarchy levels is scenario-configurable; typical ratios are 1 district = 4-9 zones, 1 zone = 4-16 sub-zones.

## 3. Zone Boundaries in 3D

Cellular networks partition the 2D ground plane. HALO-AD's zone partitioning is inherently 3D:

- **Horizontal boundaries.** Project the zone's plan-view boundary upward through all relevant altitudes. Targets crossing the horizontal boundary at any altitude trigger horizontal-boundary handoff procedures.
- **Vertical (altitude) boundaries.** Optionally, zones can be partitioned by altitude — a low-altitude zone and a high-altitude zone covering the same horizontal area, with different sensor-and-mast configurations optimized for each. Targets crossing altitude boundaries trigger altitude-boundary handoff.
- **Diagonal boundaries.** A target moving diagonally crosses both a horizontal boundary and (potentially) an altitude boundary in a single transition. Handoff procedures must coordinate both dimensions simultaneously.

The simulator supports each boundary type. Researchers can study scenarios with horizontal-only, altitude-only, or full-3D partitioning.

## 4. Zone-Boundary Overlap

Cellular networks define cells with crisp boundaries (each location is in exactly one cell), but practical handoff requires *overlap* — regions where the source and destination cells both have coverage, providing time for the handoff procedure to complete before the source cell loses signal. HALO-AD adopts the same approach:

- Each zone has a *core* region (where only this zone has primary responsibility) and a *boundary* region (where this zone shares responsibility with neighbors).
- Handoff is initiated when a tracked object enters the boundary region from the core; handoff completes when the object exits the boundary region into the destination's core.
- Boundary-region width is scenario-configurable; wider boundaries provide more handoff time at the cost of more inter-zone coordination.

The simulator parameterizes boundary width and computes handoff-success rate as a function. Researchers studying particular operational constraints (e.g., bandwidth-limited inter-zone communication) can identify the appropriate boundary width.

## 5. Zone-to-Mast Mapping

Each zone is associated with one or more masts. The mapping is many-to-many in principle:

- One mast may contribute to multiple zones (if the mast is near a zone boundary).
- One zone may be served by multiple masts (the typical case for redundancy).

The mapping is determined by the coverage analysis: each (mast, zone) pair has a coverage fraction (the fraction of zone volume the mast can observe under nominal conditions). The mapping retains pairs above a configurable threshold; pairs below threshold are excluded as not contributing meaningfully to the zone.

## 6. Inter-Zone Coordination

Multiple zones must coordinate to:

- **Transfer track responsibility** at zone boundaries (handoff per `zone_handoff.md`).
- **Share situational awareness.** A high-priority track in one zone is relevant to neighboring zones that may need to prepare for it.
- **Coordinate failure response.** A mast failure in one zone may require neighbors to expand their coverage temporarily.
- **Synchronize time bases.** Inter-zone time synchronization must be sufficient for the handoff procedure's accuracy requirements.

The simulator implements inter-zone coordination as a message-passing architecture between zone controllers. Coordination overhead is characterized in terms of message rate, bandwidth requirement, and latency budget.

## 7. Zone-Level Authority

Cellular networks assign authority for transmission decisions to specific cells. HALO-AD's zoning model is analogous: each zone has authority for engagement decisions within its volume.

The authority model is policy-configurable; the maintainers do not specify a default operational policy because that decision binds regulatory, ethical, and operator-certification questions outside the repository's scope. The simulator supports policy variations:

- **Strict zone authority.** Each zone has exclusive authority within its core volume; boundary-region authority is determined by handoff state.
- **Hierarchical authority.** Zone-level decisions can be overridden by district-level decisions (e.g., for high-priority cross-zone tracks).
- **Distributed-consensus authority.** Decisions affecting multiple zones require consensus across the affected zones.

Researchers studying authority-policy alternatives can configure the simulator and compare the resulting performance across scenarios.

## 8. Dynamic Zone Reconfiguration

Real deployments may need to reconfigure zones at runtime:

- **Mast addition or removal.** A new mast comes online or an existing mast goes offline; the zone-to-mast mapping must update.
- **Zone resizing.** A zone's effective coverage shrinks (e.g., due to atmospheric conditions or sensor failures); neighboring zones may need to expand to compensate.
- **Zone splitting.** A zone with very high target density may be split into sub-zones for tractability.
- **Zone merging.** Adjacent zones with low load may be merged for efficiency.

The simulator supports each reconfiguration mechanism. Reconfiguration policies are research subjects; the maintainers provide baseline policies (e.g., simple expansion-on-failure) and researchers can study alternatives.

## 9. The Cellular-Network Analogy: Where It Fits and Where It Doesn't

The cellular-RF-planning analogy is productive but imperfect:

- **What fits.** Hierarchical structure, boundary-region overlap, handoff procedures, frequency-reuse-like analyses (zones using overlapping sensor frequencies must coordinate to avoid interference, analogous to cell-level frequency-reuse planning), authority delegation, dynamic reconfiguration.
- **What doesn't fit.** Cellular networks serve mobile users moving through cells *cooperatively* (the user's device participates in handoff). Zoning targets are uncooperative — they do not signal their position or velocity to the zone controllers. This is structurally different and requires the multi-modal sensing fusion documented elsewhere; the cellular analogy provides the *coordination architecture* but not the *sensing architecture*.
- **What's harder in the 3D context.** Cellular cells are 2D; zones are 3D, and 3D-boundary handoff is geometrically more complex. The simulator handles this but at higher computational cost than the 2D analogue.

The maintainers consider the analogy as a useful framework for understanding the coordination problem, not as a directly-applicable solution. The simulator implements the actual algorithms appropriate to the 3D-engagement-volume context.

## 10. Civilian Transfer

The zoning model transfers to several civilian application domains:

- **Cellular-network planning.** The original analog. The simulator can be configured for purely-2D cellular-network coverage problems with the appropriate parameter choices.
- **Air-traffic-management sectorization.** Air-traffic-control airspace is partitioned into sectors with sector-controller authority; sector-boundary handoff is structurally analogous to zone handoff.
- **Multi-camera sports-tracking sectorization.** Stadium tracking systems partition the field into sectors served by camera clusters; zone handoff occurs as players cross sector boundaries.
- **Wildlife-monitoring network sectorization.** Large-area wildlife-monitoring networks partition the area into sectors with sector-cluster sensor responsibility; animal-track handoff occurs as animals cross sectors.
- **Multi-robot inspection coordination.** Large-structure inspection partitions the structure into regions with region-team responsibility; inspection-task handoff occurs as the work flow crosses regions.
- **Distributed-computing region planning.** Geographically-distributed computing systems partition workload by region with region-cluster responsibility; load-balancing handoff occurs as workload shifts.

The framework is general; the parameter choices are domain-specific.

## 11. Composition with Other Theory Documents

The zoning model is one layer of HALO-AD's overall framework:

- `coverage_geometry.md` — Provides the per-zone coverage analysis. Zone-to-mast mapping uses these outputs.
- `sensor_fusion.md` — Provides the per-zone fusion architecture. Each zone implements the fusion architecture for its assigned masts.
- `steering_latency_and_field_of_view.md` — Provides the per-mast architecture. Zone-level coordination must accommodate mast architectural choices.
- `atmospheric_models.md` — Provides per-(zone, scenario) atmospheric parameters. Zone effective coverage depends on these.
- `zone_handoff.md` — Provides the inter-zone handoff procedures. Zoning model defines the boundaries; handoff procedures operate at them.
- `future_research.md` — Documents adjacent research domains (cellular-network planning theory, air-traffic management, etc.) for which the zoning model would provide upstream input.

The simulator integrates all of these; this document specifies the partitioning framework on which the integration rests.

## 12. Summary

The zoning model partitions large regions into hierarchical zones with explicit boundaries, overlap regions, and inter-zone coordination procedures. The cellular-RF-planning analogy provides the architectural framework; the 3D-engagement-volume context requires extensions to the analogy that are documented and implemented in the simulator.

The model is general across many civilian and research-application domains; the parameter choices are domain-specific.
