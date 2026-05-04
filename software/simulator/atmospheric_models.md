# Atmospheric Models — Documentation for the Simulation Engine

**Project:** HALO-AD
**Component:** software/simulator/atmospheric_models
**Domain:** Documentation of atmospheric-physics parameters used by the simulator

## 1. Purpose

This document describes the atmospheric-physics parameters that HALO-AD's simulation engine uses to model how a beam (or other propagation) degrades through the atmosphere. The implementation lives in `software/simulator/atmospheric_models.py`; this document explains what the parameters mean, where the underlying physics models come from, and how they integrate into the coverage analysis.

The simulator does not contain any classified or controlled atmospheric data. All models are drawn from openly published academic and government-research literature (LOWTRAN, MODTRAN, and successors at unrestricted precision; published thermal-blooming literature; Mie and Rayleigh scattering theory).

## 2. Beam Divergence

Even an ideal beam diverges. The relationship is, in simplified form:

w(z) = w₀ + z · θ

where `w(z)` is the beam radius at distance `z`, `w₀` is the initial aperture radius, and `θ` is the divergence half-angle (radians).

The simulator parameterizes:

- `aperture_diameter` — emitter aperture, meters
- `divergence_half_angle` — radians, typically 1e-4 to 1e-3 for engineering-quality optics
- `wavelength` — meters; affects the diffraction-limited minimum divergence via λ / (π · w₀) for Gaussian beams

The simulator computes power density at any range via:

power_density(z) = total_power / (π · w(z)²)

This is the baseline; atmospheric effects below modify it.

## 3. Aerosol Scattering (Mie and Rayleigh)

Particles in the atmosphere scatter the beam. The Beer-Lambert law gives transmission through an attenuating medium:

T(z) = exp(-α · z)

where `α` is the extinction coefficient, units of inverse meters.

The extinction coefficient depends on:

- **Wavelength.** Shorter wavelengths scatter more (Rayleigh ~ λ⁻⁴ for small particles).
- **Particle size distribution.** Mie scattering for aerosols (fog, dust, smoke) where particle sizes are comparable to wavelength.
- **Particle density.** Number of scatterers per unit volume.

The simulator parameterizes atmospheric conditions through preset profiles:

- **Clear.** α ≈ 1e-4 to 1e-3 per meter at visible wavelengths.
- **Hazy.** α ≈ 1e-3 to 1e-2 per meter.
- **Light fog.** α ≈ 1e-2 to 1e-1 per meter.
- **Heavy fog or dust.** α ≈ 1e-1 to 1 per meter.
- **Smoke (operational obscurant).** Highly variable; the simulator uses configurable per-scenario values.

These profiles are wavelength-dependent. Infrared wavelengths (e.g., 1.5 μm, used for many LiDAR systems) penetrate haze and light fog substantially better than visible wavelengths; the simulator accounts for this in the per-wavelength extinction lookup.

## 4. Thermal Blooming

A beam carrying significant power density heats the atmospheric column it traverses. Heated air has lower refractive index than surrounding cool air, producing a defocusing-lens effect that worsens with continued heating. This is thermal blooming.

The blooming coefficient depends on:

- **Beam power density.** Higher power density → more heating per unit time.
- **Atmospheric absorption.** Absorption (not scattering) is what couples beam energy into atmospheric heating; the absorption coefficient is wavelength-dependent.
- **Wind.** Crosswind transports the heated air column out of the beam path, mitigating blooming. The simulator parameterizes wind speed and direction.
- **Beam dwell time.** Continuous beams accumulate blooming; pulsed beams of low duty cycle do not.

The simulator implements published thermal-blooming models at the academic-literature precision (no controlled parameter values) and produces effective beam-quality degradation as a function of these inputs. The output is an effective beam radius `w_eff(z)` larger than the geometric `w(z)`, reducing the effective power density at the target.

## 5. Turbulence

Atmospheric turbulence — small-scale refractive-index variations from temperature mixing and convection — produces beam wander, beam spread, and intensity scintillation. The standard parameterization is the Fried parameter `r₀` (the diameter of an aperture for which atmospheric turbulence becomes dominant):

- `r₀` is wavelength-dependent (longer wavelengths → larger `r₀`, less turbulence-affected).
- `r₀` varies with altitude, time of day, and weather. The simulator uses scenario-configurable values.

For long-range engagement, turbulence becomes the dominant beam-quality degradation mechanism beyond a few kilometers in typical conditions. Adaptive-optics correction (modeled in the simulator at architectural depth, not at controlled-parameter precision) can recover most turbulence-induced loss; the tradeoff is system complexity and cost.

## 6. Absorption Bands

Specific atmospheric constituents absorb at specific wavelengths:

- **Water vapor.** Strong absorption bands around 1.4 μm, 1.9 μm, 2.7 μm, and others. Wavelengths between bands ("atmospheric windows") propagate well.
- **Carbon dioxide.** Bands around 2.0 μm, 2.7 μm, 4.3 μm.
- **Other constituents.** Ozone, methane, etc., contribute at specific wavelengths.

The simulator's wavelength-selection guidance follows the standard atmospheric-windows literature: 905 nm, 1.5 μm, and 1.55 μm are common LiDAR wavelengths because they sit in atmospheric windows; longer-wavelength systems target windows around 3.5 μm or 10 μm depending on application.

## 7. Composition into Coverage Analysis

The atmospheric models compose into the coverage analysis as follows:

1. For each (emitter, target) pair, compute geometric beam parameters at target range.
2. Apply transmission `T(z)` from aerosol scattering.
3. Apply effective-beam-size `w_eff(z)` from thermal blooming and turbulence.
4. Compute effective power density at target.
5. If effective power density exceeds the application's detection threshold (for sensing) or sufficiency threshold (for any other simulation purpose), the (emitter, target) pair is "covered" in this scenario.

Atmospheric conditions are scenario-configurable, so the same emitter geometry can be tested against multiple weather conditions. This is one of the simulator's primary research outputs: how does coverage degrade across weather classes?

## 8. References

The atmospheric models are drawn from openly published academic and government-research literature:

- *Fundamentals of Aeronautical and Astronautical Engineering* and equivalent textbooks for baseline atmospheric properties.
- LOWTRAN, MODTRAN, and successor codes at their open-publication precision.
- Published Mie and Rayleigh scattering theory in atmospheric-physics textbooks.
- Published thermal-blooming literature (e.g., Smith et al., 1977; Gebhardt, 1976; subsequent decades of work).
- Published adaptive-optics literature from the astronomical-telescope community.
- Published atmospheric-windows literature for wavelength-selection guidance.

The simulator does not include parameters at thresholds restricted by export-control regimes; see `legal/export_control_posture.md` for the full posture.
