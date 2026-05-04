# Internal Simulation Protocol Specification — HALO-AD

## Scope

This document specifies the internal event protocol used among HALO-AD's simulation components. It is not a hardware-communication protocol, not a command-and-control protocol, and not a network protocol. It exists solely to coordinate components of the simulation harness running in a single process or across processes on a research workstation.

The protocol's events are simulation events. No event in this protocol corresponds to a real-world physical effect.

## Event Types

The simulation harness emits and consumes events of the following abstract categories:

**Scenario events.** Scenario load, scenario start, scenario end, scenario fault. Carry scenario metadata (region geometry, threat-trajectory dataset reference, mast configuration reference).

**Threat events.** Simulated threat appearance, position update, classification update, disappearance. Carry simulated threat state (position, velocity, type, simulated radar cross-section, simulated optical signature).

**Sensor events.** Simulated sensor detection, track update, track loss. Carry simulated sensor outputs in the same form a real sensor would produce: bounding boxes, range-Doppler returns, event-camera spike trains.

**Coverage events.** Coverage map update, dead-zone identification, observability change. Carry per-voxel coverage statistics.

**Coordination events.** Zone-handoff request, zone-handoff confirmation, mast-assignment update. Carry coordination decisions.

**Logging events.** Performance metric, latency measurement, statistical update. Carry evaluation data for trade-study reporting.

## Encoding

Events are encoded as Python dataclasses internally and as JSON for cross-process communication. A JSON Schema is published in `software/protocol/schema.json` for tooling support. No binary or wire-efficient encoding is provided; the protocol is for research-workstation use, not for high-throughput operational deployment.

## Authority and Decision Boundaries

The protocol does not include any "engage" event, "fire" event, or "authorize" event. Coordination decisions are simulation decisions about which simulated mast tracks which simulated threat. There is no event that would, if observed by an external system, cause a real-world effect.

This is by deliberate design. The protocol's event vocabulary is bounded so that no extension to operational use is possible without redesigning the protocol entirely — which would constitute a separate project outside this repository's scope.

## Versioning

The protocol is versioned with the repository. Changes require a maintainer review and a CHANGELOG entry tagged `[protocol]`.
