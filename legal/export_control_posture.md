# Export Control Posture — HALO-AD

## Posture Summary

HALO-AD's published content consists of:

1. Distributed-emitter coverage geometry algorithms (`software/geometry/`).
2. Multi-zone partitioning and coordination algorithms (`software/coordination/`).
3. Predictive-tracking integration with PentaTrack (`software/perception/`).
4. A simulation harness, scenario library, and visualization tools (`software/simulator/`, `software/viz/`).
5. Documentation of methodology and the maintainers' compliance posture.

The maintainers assess that none of the above is, as published, controlled under:

- The International Traffic in Arms Regulations (ITAR), 22 CFR Parts 120-130.
- The Export Administration Regulations (EAR), 15 CFR Parts 730-774.
- The Wassenaar Arrangement dual-use lists.
- EU Regulation 2021/821 (recast Dual-Use Regulation).
- UK Strategic Export Control Lists.
- Equivalent national regimes.

The basis of this assessment:

- The content is openly published, available without restriction, and consists of methodology rather than controlled technology parameters.
- No munitions, no controlled directed-energy thresholds, no controlled radar parameters, and no controlled cryptographic implementations are present.
- The "fundamental research" and "publicly available" exceptions of the U.S. regimes apply on their face, and equivalent exceptions in other regimes apply on the same basis.
- Civilian-transfer applications are concretely demonstrated (telescope-array siting, sensor-network planning, cellular-RF-planning).

This assessment is the maintainers' good-faith determination. It is not legal advice. It is reviewed on a per-pull-request basis. It is reviewable by competent counsel in any jurisdiction; the maintainers welcome correction.

## What Would Change the Posture

The following kinds of contribution would, if accepted, change the export-control posture and are therefore out of scope:

- Specification of beam-energy parameters above eye-safe educational classes (IEC 60825 Class 1).
- Specification of radar parameters at controlled thresholds.
- Inclusion of fire-control authority logic, target-classification at lethal-decision quality, or rules-of-engagement implementation.
- Adoption of any specific operational directed-energy platform's interface.
- Coupling of simulation outputs to any external real-world system, sensor, or actuator.
- Inclusion of cryptographic implementations at controlled strengths for command and control.
- Adoption of platform-specific simulators for controlled hardware.

Pull requests of these types will be closed.

## Researcher Responsibilities

Researchers extending HALO-AD toward physical hardware are entirely responsible for:

- Their own jurisdiction's export-control compliance.
- FAA / national civil-aviation-authority requirements.
- FCC / national spectrum-regulator requirements.
- Laser-safety compliance (IEC 60825, ANSI Z136).
- Their own institutional review.
- Disclosure to relevant regulators where required.

The maintainers do not provide guidance on operational extension and do not maintain a private fork of operational extensions.
