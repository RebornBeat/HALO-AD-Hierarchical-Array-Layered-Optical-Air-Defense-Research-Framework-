# Zone Handoff — Documentation for the Coordination Engine

**Project:** HALO-AD
**Component:** software/coordination/zone_handoff
**Domain:** Documentation of multi-zone coordination logic

## 1. Purpose

This document describes how HALO-AD's coordination engine handles tracks and engagement responsibility across zone boundaries. The implementation lives in `software/coordination/zone_handoff.py`; this document explains the architecture, the failure modes, and the simulation outputs that quantify handoff performance.

The coordination engine operates entirely within the simulation. It does not communicate with any real-world hardware, does not issue any operational commands, and is not connected to any external system. Its purpose is to study the algorithmic and architectural questions of multi-zone coordination.

## 2. The Multi-Zone Architecture

A defended region in HALO-AD's simulator is partitioned into:

- **Districts.** Large geographic regions (cities, sectors, regions of a country).
- **Zones within districts.** Smaller engagement-volume partitions associated with one or more masts.
- **Sub-zones within zones.** Optional finer partitioning for high-density coverage areas.

Each zone is associated with one or more masts. Zones overlap at their boundaries to enable continuous-track handoff between mast clusters.

A track that crosses a zone boundary must be handed off from the source zone's mast cluster to the destination zone's mast cluster without observable degradation. This is the core problem zone handoff addresses, structurally analogous to cellular-network handoff.

## 3. Handoff Triggers

The coordination engine evaluates handoff opportunities continuously:

- **Geographic trigger.** A track has crossed (or is predicted to cross) a zone boundary. Predictive triggering uses the track's velocity and the zone-boundary geometry.
- **Coverage trigger.** The destination zone's masts have better observability than the source zone's masts (better range, less occlusion, better atmospheric path).
- **Load trigger.** The source zone is at coordination capacity (multiple high-priority tracks); the destination zone has spare capacity.
- **Failure trigger.** A mast in the source zone has gone offline; the destination zone takes responsibility.

Triggers are not exclusive — multiple can fire simultaneously. The coordination policy weights them.

## 4. Handoff Procedure

When a handoff is triggered:

1. **Pre-handoff observation.** The destination zone's masts begin observing the track, building local track state in parallel with the source zone.
2. **Track state merge.** The destination zone receives the source zone's track state (PentaTrack predictive-center field, drift history, object classification, anomaly state) and merges it with its own observations.
3. **Authority transfer.** The destination zone takes responsibility for the track. The source zone releases the track.
4. **Confirmation.** Both zones confirm the transfer; the simulator logs the handoff event for analysis.

The procedure is conservative — overlapping observation reduces the risk of track loss during handoff at the cost of duplicate observation effort during the overlap window. The overlap window is policy-configurable.

## 5. Failure Modes

Handoff can fail in several ways:

- **Slow handoff.** The handoff procedure takes longer than the track's transit time across the boundary. The track is briefly observed by neither zone and the prediction confidence drops.
- **State-merge inconsistency.** The destination zone's local observations conflict with the source zone's track state. The merge produces a track with degraded confidence; the simulator flags this for analysis.
- **Authority confusion.** Both zones believe they have authority simultaneously, or neither does. Detected and recovered automatically; simulator logs.
- **Cascade failures.** A handoff failure during a multi-track scenario can produce coordination overload as both zones try to recover, increasing the failure probability for subsequent handoffs.

The simulator characterizes each failure mode and produces statistics across scenarios.

## 6. Hierarchical Coordination

For country-scale or large-region scenarios, the coordination engine supports hierarchical zoning:

- **District level** (top of hierarchy). Long-horizon resource allocation across districts; slow-tempo handoffs of tracks between districts.
- **Zone level.** Intra-district coordination; medium-tempo handoffs.
- **Sub-zone level** (when present). High-density coordination within a zone; fast-tempo handoffs.

The hierarchy mirrors cellular-network macro/micro/pico-cell structure. The simulator's coordination engine implements all three levels.

## 7. Failure-Resilient Coordination

When a mast or zone fails, the coordination engine redistributes responsibility:

- **Mast failure.** The mast's tracks are handed off to neighboring masts within the same zone. Coverage degrades within the zone but is not lost.
- **Zone failure** (multiple masts simultaneously). The zone's tracks are handed off to neighboring zones. Coverage in the failed zone may be lost; neighboring zones expand their effective coverage as resources permit.
- **District failure** (multiple zones simultaneously). The district's tracks are handed off to neighboring districts. Cross-district handoff is slower than intra-district handoff; the simulator characterizes the resulting performance degradation.

## 8. Simulation Outputs

Zone-handoff performance is quantified through:

- **Handoff latency distribution.** Time from handoff trigger to authority transfer, across scenarios.
- **Handoff success rate.** Fraction of triggered handoffs that complete without track loss.
- **Track-continuity metric.** Average track length across zone boundaries, normalized to the no-boundary case.
- **Coordination overhead.** Bandwidth and computation cost of inter-zone communication, characterized as a function of track count and zone-boundary-crossing rate.

These outputs are CSV deliverables suitable for academic comparison.

## 9. Civilian Transfer

The zone-handoff architecture transfers to:

- **Cellular network handoff.** The same problem with different physical-layer details.
- **Multi-camera sports tracking.** Tracking players across camera fields of view.
- **Multi-robot coordination.** Tasks crossing robot territories.
- **Multi-vehicle dispatch.** Dispatch responsibility crossing dispatcher zones.
- **Wildlife tracking.** Animal movements crossing camera-trap or sensor-network zones.

The algorithms in `software/coordination/zone_handoff.py` are written to be generic over the underlying observation modality; the civilian-transfer applications use the same code with different sensor configurations.
