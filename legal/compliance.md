# Compliance and Regulatory Posture — HALO-AD

## Project Classification

HALO-AD is classified by its maintainers as **simulation-and-research software** in the domain of distributed sensing, predictive tracking, geometric coverage analysis, and multi-agent coordination. The repository contains no hardware designs, no firmware capable of driving an effector, no specifications of weaponized optical or directed-energy systems, and no fire-control authority logic. The repository's outputs are HTML reports, CSV trade-study data, and visualizations.

This classification is the basis on which the maintainers have determined HALO-AD does not implicate, as published, the major dual-use export-control regimes (ITAR, EAR, the Wassenaar Arrangement, EU dual-use regulation 2021/821) or directed-energy-weapon-specific national regulations. The classification is reviewed on a per-pull-request basis. Contributions that would change the classification are out of scope.

## Dual-Use Considerations

The maintainers acknowledge that some algorithms developed in HALO-AD — multi-emitter coordination, zone handoff, multi-sensor fusion, predictive tracking — have civilian applications (sensor-network planning, telescope arrays, multi-camera sports tracking, multi-robot coordination) and potential military applications. This dual-use character is shared with most published research in optimization, sensor fusion, and computer vision. The maintainers' position is that publishing the *algorithms and the simulation harness* — without hardware, without weaponized parameters, and without any "live" mode — is consistent with the open-research norms of these fields.

The repository explicitly excludes:

- Beam-energy parameters above eye-safe educational classes.
- Coupling of simulation outputs to any external real-world system, sensor, or actuator.
- Optimization of engagement parameters for damage thresholds against any specified target (manned, unmanned, civilian, or military).
- Any code path that could be flipped from "simulation" to "live" without substantial new development that this repository does not provide.

## Export-Control Posture

The maintainers have reviewed HALO-AD's contents against the Commerce Control List (CCL) and the United States Munitions List (USML) and have determined the published content is not, in itself, controlled. This determination is the maintainers' good-faith assessment and is not legal advice. Users in any jurisdiction are responsible for their own compliance.

Specifically, the published content does not include:

- Technical data within the meaning of ITAR §120.10, because the data does not relate to defense articles enumerated on the USML.
- Technology controlled under EAR Categories 6 (sensors and lasers) or 7 (navigation), because the content describes simulation methodology rather than controlled technology parameters.
- Wassenaar dual-use list items, including Category 6 (sensors and lasers), because no system parameters at controlled thresholds are specified.

The maintainers will not accept contributions that introduce content within these categories. Pull requests including controlled parameters, hardware specifications above unregulated thresholds, or technology classified under any of the above regimes will be closed.

## Aviation and Spectrum Regulation

HALO-AD does not transmit, receive, or affect any real-world signal. No aviation-spectrum, radar-spectrum, or laser-emission regulation applies to the published content. Researchers extending the work toward physical hardware are responsible for FAA (or equivalent national civil-aviation authority), FCC (or equivalent national spectrum regulator), and laser-safety (IEC 60825 / ANSI Z136) compliance in their own jurisdictions. The maintainers do not provide guidance on physical extension.

## Research-Ethics Posture

The maintainers consider research into geometric coverage, multi-emitter coordination, and predictive tracking to be ethically aligned with established academic norms in operations research, sensor-network planning, and computer vision. The maintainers consider extension of this research to live weaponized systems to be a separate ethical and legal question that this repository does not address and is not the appropriate venue for. Researchers pursuing such extension in institutional contexts with appropriate ethical-review boards and regulatory clearance are not the audience of this repository's `legal/` directory.

## Reporting and Disclosure

If you believe content in this repository has crossed any of the lines described above — for instance, a pull request you observed, a parameter that should not have been published, or a configuration that exceeds the simulation-only posture — please open a public issue with the `compliance` label, or contact the maintainers privately if the content is sensitive enough that public disclosure would itself be harmful. The maintainers will review and, if the report is substantiated, remove the content from the repository (including from history if necessary) and publish a CHANGELOG entry describing the action.

## Versioning

This compliance posture is versioned with the repository. Changes to the posture itself require a maintainer review and a CHANGELOG entry tagged `[compliance]`. The current version of this document corresponds to the current repository tag.
