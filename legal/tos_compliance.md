# GitHub and Platform Terms-of-Service Compliance — HALO-AD

The maintainers' assessment of HALO-AD's compliance with the Terms of Service of relevant platforms.

## GitHub Acceptable Use Policy

The GitHub Acceptable Use Policy restricts content that, in the platform's judgment, constitutes weapons development or facilitates illegal activity. The maintainers' assessment is that HALO-AD's published content — simulation algorithms, geometric analysis, predictive tracking integration, and methodology documentation — does not constitute weapons development under any reasonable reading of the policy, for the reasons documented in `compliance.md` and `export_control_posture.md`:

- The repository contains no hardware design, no firmware, and no operational integration code.
- No directed-energy hardware specifications are present at any threshold above unregulated educational classes.
- The algorithms have demonstrated civilian applications and the civilian examples are part of the published repository.
- The documentation explicitly disclaims operational use, names the regulatory landscape, and identifies categories of contribution that would be rejected.

The maintainers will respond promptly and in good faith to any GitHub Trust & Safety inquiry. If GitHub's policy interpretation differs from the maintainers' own, the maintainers will revise the repository to comply, including by removing content if necessary.

## Platform Mirroring

The same posture extends to GitLab, Codeberg, SourceHut, and other Git hosting platforms. The maintainers will not host HALO-AD on any platform whose policy the content cannot satisfy.

## Package Registries

If HALO-AD's components are published to PyPI, npm, or other package registries, the maintainers will ensure metadata accurately reflects the simulation-and-research classification.

## Pull-Request and Issue Discipline

PRs and issues are restricted to the simulation-and-research scope. Operational-deployment discussion, jurisdictional-circumvention discussion, integration-with-real-platform discussion, and similar threads will be locked and labeled. This is not a content-moderation preference; it is a project-scope requirement, and it is what allows the repository to remain on hosting platforms in good standing.
