# Atmospheric Models — Documentation for the Simulation Engine

**Project:** HALO-AD
**Component:** `software/simulator/atmospheric_models`
**Domain:** Atmospheric Physics & Propagation Modeling

## 1. Purpose

This document describes the atmospheric-physics parameters that HALO-AD's simulation engine uses to model how a beam (or other propagation) degrades through the atmosphere. The implementation lives in `software/simulator/atmospheric_models.py`; this document explains what the parameters mean, where the underlying physics models come from, and how they integrate into the coverage analysis.

The simulator does not contain any classified or controlled atmospheric data. All models are drawn from openly published academic and government-research literature (LOWTRAN, MODTRAN, and successors at unrestricted precision; published thermal-blooming literature; Mie and Rayleigh scattering theory).

## 2. Beam Propagation and Divergence

Even an ideal beam diverges due to diffraction. Real beams diverge further due to optical imperfections.

### 2.1 Geometric Divergence

The relationship for beam radius $w(z)$ at distance $z$ from the aperture is governed by the divergence half-angle $\theta$:

$$w(z) = w_0 + z \cdot \theta$$

Where:
*   $w(z)$: Beam radius (1/e² radius) at distance $z$.
*   $w_0$: Initial aperture radius.
*   $\theta$: Divergence half-angle (radians).

### 2.2 Beam Quality Factor ($M^2$)

Real optical systems are not perfectly Gaussian. The simulator uses the Beam Quality Factor ($M^2$) to model real-world imperfections. An ideal Gaussian beam has $M^2 = 1.0$. High-quality fiber lasers typically range from $1.1$ to $1.5$. The effective divergence is:

$$\theta_{eff} = M^2 \cdot \frac{\lambda}{\pi \cdot w_0}$$

The simulator parameterizes:
*   `aperture_diameter`: Emitter aperture, meters.
*   `wavelength`: Meters (e.g., $1.55 \times 10^{-6}$ for typical LiDAR).
*   `m_squared`: Dimensionless beam quality factor (default 1.0–2.0).

### 2.3 Power Density Calculation

The simulator computes power density ($I$) at any range via:

$$I(z) = \frac{P_{total}}{\pi \cdot w(z)^2}$$

This serves as the baseline energy on target before atmospheric effects are applied.

## 3. Atmospheric Transmission (Scattering and Absorption)

Particles in the atmosphere scatter and absorb the beam. The Beer-Lambert law gives transmission $T(z)$ through an attenuating medium:

$$T(z) = \exp(-\alpha \cdot z)$$

Where $\alpha$ is the extinction coefficient (inverse meters).

### 3.1 Extinction Mechanisms

The extinction coefficient is the sum of scattering and absorption:

$$\alpha = \alpha_{scattering} + \alpha_{absorption}$$

*   **Rayleigh Scattering:** Dominant for small particles (molecules). Scales as $\lambda^{-4}$. Significant for UV/Visible; negligible for Infrared.
*   **Mie Scattering:** Dominant for aerosols (fog, dust, smoke) comparable to wavelength. Wavelength dependence is weaker.
*   **Absorption:** Wavelength-specific coupling of energy into atmospheric gases (Water Vapor, CO₂).

### 3.2 Double-Pass Attenuation (LiDAR Mode)

For active sensing (LiDAR), the signal traverses the path twice (Emitter $\rightarrow$ Target $\rightarrow$ Receiver). The total transmission is squared:

$$T_{total} = T(z)^2 = \exp(-2 \alpha z)$$

This is critical for calculating **Receiver Sensitivity** and **Maximum Detection Range** in the simulation.

### 3.3 Weather Presets

The simulator parameterizes atmospheric conditions through preset profiles derived from published visibility data:

| Profile | Visibility (km) | Extinction $\alpha$ (1/m) at 1.5 µm | Notes |
| :--- | :--- | :--- | :--- |
| **Clear** | > 23 | $1 \times 10^{-5}$ | Standard atmosphere, excellent propagation. |
| **Hazy** | 5 – 15 | $5 \times 10^{-5}$ | Moderate aerosol loading. |
| **Light Fog** | 1 – 2 | $2 \times 10^{-4}$ | Significant attenuation; IR advantages visible. |
| **Heavy Fog** | < 0.5 | $> 1 \times 10^{-3}$ | Optical systems effectively disabled; Radar dominance. |
| **Smoke** | Variable | Configurable | Scenario-defined obscurants. |

Infrared wavelengths (e.g., 1.5 µm, 1.55 µm) generally exhibit lower extinction in hazy conditions compared to visible wavelengths (e.g., 532 nm), a distinction the simulator accounts for in its per-wavelength lookup tables.

## 4. Thermal Blooming

A beam carrying significant power density heats the atmospheric column it traverses. Heated air has a lower refractive index ($n$) than surrounding cool air, producing a defocusing-lens effect that worsens with continued heating. This is **Thermal Blooming**.

### 4.1 Governing Factors

The blooming effect is governed by the thermal nonlinearity of air. The effective blooming distance $R_B$ (distance at which blooming significantly degrades the beam) is inversely proportional to power and absorption.

Key dependencies:
*   **Power Density ($I$):** Higher intensity $\rightarrow$ faster heating.
*   **Absorption Coefficient ($\alpha_a$):** Only absorbed energy heats the air.
*   **Wind Speed ($v$):** Crosswind transports the heated air out of the beam path, mitigating blooming.
*   **Beam Dwell Time ($t$):** Continuous Wave (CW) beams accumulate blooming; pulsed beams with low duty cycle generally do not.

### 4.2 Simulation Model

The simulator implements published thermal-blooming models (e.g., **Smith & Gebhardt**) to produce an effective beam radius $w_{bloom}(z)$ that is larger than the geometric radius $w(z)$.

$$w_{eff}(z) = \sqrt{w(z)^2 + w_{bloom}(z)^2}$$

The output is a reduction in effective power density at the target. This model is essential for simulating high-power directed-energy geometries, as it introduces a non-linear constraint: **increasing power indefinitely eventually reduces effective intensity on target due to self-defocusing.**

## 5. Atmospheric Turbulence

Atmospheric turbulence—small-scale refractive-index variations from temperature mixing and convection—produces beam wander, beam spread, and intensity scintillation (fading).

### 5.1 Refractive Index Structure Constant ($C_n^2$)

The strength of optical turbulence is characterized by the refractive index structure constant, $C_n^2$.
*   **Units:** $m^{-2/3}$
*   **Range:**
    *   Strong turbulence: $C_n^2 > 10^{-13}$
    *   Moderate turbulence: $C_n^2 \approx 10^{-14} - 10^{-15}$
    *   Weak turbulence: $C_n^2 < 10^{-16}$

The simulator allows users to define $C_n^2$ profiles, often modeled as varying with altitude (e.g., Hufnagel-Valley model).

### 5.2 Fried Parameter ($r_0$)

The standard parameterization for long-range imaging/tracking is the Fried parameter ($r_0$), defining the aperture diameter over which atmospheric phase distortions become severe.

$$r_0 \approx \left[ 0.423 \cdot k^2 \cdot C_n^2 \cdot z \right]^{-3/5}$$

Where $k = 2\pi / \lambda$.
*   If Aperture $D > r_0$: The beam spreads beyond the diffraction limit, limiting resolution.
*   **Wavelength Dependence:** $r_0$ scales as $\lambda^{6/5}$. Longer wavelengths are less affected by turbulence (e.g., a 10 µm beam is far more robust to turbulence than a 1 µm beam).

### 5.3 Adaptive Optics (AO)

The simulator models **Adaptive Optics** correction at an architectural level.
*   **AO Off:** Beam spread is limited by $r_0$ rather than aperture $D$.
*   **AO On:** Corrects phase distortions, effectively increasing $r_0$ (often modeled as restoring the beam to near-diffraction-limited performance).
*   **Trade-off:** AO systems add complexity, cost, and latency (computational loop time) to the simulation.

## 6. Absorption Bands and Atmospheric Windows

Specific atmospheric constituents absorb strongly at specific wavelengths. Selecting an operating wavelength requires choosing an "Atmospheric Window" where absorption is minimized.

### 6.1 Key Absorbers
*   **Water Vapor (H₂O):** Strong bands around 1.4 µm, 1.9 µm, 2.7 µm, and 6.3 µm.
*   **Carbon Dioxide (CO₂):** Strong bands at 2.0 µm, 2.7 µm, 4.3 µm.
*   **Ozone (O₃):** UV absorption, some IR bands.

### 6.2 Common Simulation Windows
The simulator defaults to standard "eye-safe" and LiDAR-optimal windows:
*   **905 nm:** Near-IR. Good for short-range LiDAR; heavily attenuated by fog.
*   **1.5 µm – 1.55 µm:** "Eye-safe" band. Excellent balance of low absorption and good detector availability. Standard for long-range terrestrial LiDAR.
*   **3.5 µm – 4.0 µm:** Mid-IR window. Good transmission through moderate haze.
*   **10.6 µm:** Far-IR (CO₂ laser). Poor resolution (diffraction limit) but robust in steam/fog.

## 7. Composition into Coverage Analysis

The atmospheric models compose into the `coverage_geometry` engine via a strict calculation pipeline:

1.  **Geometric Calculation:** Compute geometric beam radius $w(z)$ based on aperture and divergence.
2.  **Atmospheric Transmission:** Calculate $T(z)$ using Beer-Lambert law with scenario-defined $\alpha$.
3.  **Beam Quality Degradation:**
    *   Apply **Turbulence** effects (modify effective spot size based on $r_0$).
    *   Apply **Thermal Blooming** effects (increase spot size based on power and wind).
4.  **Effective Intensity:** Compute effective power density $I_{eff}$ on target.
5.  **Threshold Check:**
    *   **For Sensing:** Compare return signal strength (Factor of $T^2$) against detector sensitivity.
    *   **For Effect:** Compare $I_{eff}$ against target damage/fluency thresholds (if simulating offensive geometry; abstracted for this research scope).

**Primary Research Output:** This composition allows researchers to observe how a system optimized for "Clear" weather fails in "Fog," or how high-power systems suffer from "Thermal Blooming" at long ranges, forcing trade-offs between power, wavelength, and aperture size.

## 8. References

The atmospheric models are drawn from openly published academic and government-research literature:

*   *Fundamentals of Aeronautical and Astronautical Engineering* and equivalent textbooks for baseline atmospheric properties.
*   **LOWTRAN / MODTRAN:** Legacy and current US Air Force atmospheric transmission codes (unrestricted versions).
*   **Mie Theory:** Standard calculation for aerosol scattering (Bohren & Huffman, "Absorption and Scattering of Light by Small Particles").
*   **Thermal Blooming:** Smith, D.C. (1977). "High-power laser propagation." *Applied Optics*; Gebhardt, F.G. (1976). "High power laser propagation."
*   **Turbulence:** Andrews, L.C., & Phillips, R.L. (2005). *Laser Beam Propagation through Random Media*. (Spie Press).
*   **Adaptive Optics:** Hardy, J.W. (1998). *Adaptive Optics for Astronomical Telescopes*. Oxford University Press.

The simulator does not include parameters at thresholds restricted by export-control regimes; see `legal/export_control_posture.md` for the full posture.
