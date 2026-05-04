# Research Ethics — HALO-AD

## Position

The maintainers consider HALO-AD's published research scope — distributed-emitter coverage geometry, multi-zone partitioning, predictive tracking integration, and multi-mast coordination — to be ethically aligned with established academic norms in operations research, sensor-network planning, computer vision, and distributed systems. The published work does not engage with the most ethically contested elements of directed-energy research (effector design at damage thresholds, fire-control authority logic, operational deployment) and is not the appropriate venue for them.

The maintainers further consider that the underlying academic research community has produced substantial published work on the algorithmic problems HALO-AD addresses, and that the research-norm expectation is publication, peer review, and reproducibility — which is what HALO-AD provides. Restricting publication of the algorithms and simulation methodology does not, in the maintainers' view, advance any ethical objective; it merely transfers the work from open science to closed institutional research, with no reduction in accessibility to actors with the resources to develop them privately. Open publication, by contrast, makes the work available to defensive-research communities, civilian-transfer applications, students, and researchers in under-resourced institutions.

## Lethal-Autonomy Position

The maintainers do not develop, do not accept contributions for, and do not endorse the development of *autonomous* lethal-decision systems. The repository's coordination layer is a simulation harness; it does not close any engagement loop, does not classify targets at lethal-decision quality, and does not integrate with any specific operational platform. The repository's posture aligns with the published statements of multiple AI-research organizations and a substantial fraction of the AI-research community on the question of lethal autonomous weapons systems.

The line is the closing of the loop. The repository sits, by deliberate design, well behind that line, and stays there.

## Civilian Transfer

The maintainers consider the civilian-transfer applications of HALO-AD's algorithms (telescope-array siting, sensor-network planning, distributed-camera coverage in non-privacy-sensitive contexts, cellular-RF-planning) to materially advance the project's ethical posture by demonstrating that the published work's utility does not depend on the directed-energy framing. The maintainers will resist project drift toward purely-defense framing and will actively curate civilian-transfer examples as a substantive part of the repository.

## Disagreement and Forking

If a researcher or contributor disagrees with this ethics posture and wishes to pursue work that crosses the lines this document draws, the appropriate path is to fork under a new name, publish that fork's own ethics posture, and pursue the regulatory and institutional approvals that operational work requires. The maintainers will not litigate ethical disagreements in the repository's issue tracker.

## Reporting

Researchers, journalists, and the public who believe this posture has been violated in practice — for instance, a pull request was merged that crossed the line, or a downstream project misrepresents its relationship to this one — are invited to open a public issue with the `ethics` label or to contact the maintainers privately.

---

---

# `halo-ad/legal/export_control_posture.md`

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
