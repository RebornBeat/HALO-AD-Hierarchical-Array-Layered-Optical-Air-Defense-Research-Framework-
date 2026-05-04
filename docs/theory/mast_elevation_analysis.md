# Mast Elevation Analysis — System-Level Trade-offs for Distributed Architectures

**Project:** HALO-AD
**Domain:** Geometric Analysis, Sensor Placement Strategy, Infrastructure Engineering
**Status:** Theoretical / Simulation Framework

## 1. Purpose

This document analyzes the system-level impact of elevating directed-energy and sensor nodes onto masts or towers within the HALO-AD simulation framework. While `coverage_geometry.md` establishes the mathematical formulas for horizon calculation and beam propagation, this document focuses on the **architectural consequences** of elevation: the recovery of the lower hemisphere, the creation of "Shadow Cones," the interaction with terrain masking, and the trade-offs between "Tall Tower" vs. "Distributed Short Mast" topologies.

The goal is to provide researchers with the parameters necessary to optimize mast height ($H_{mast}$) against cost, stability, and coverage gaps.

## 2. The Baseline Problem: The "Half-Dome" Blind Spot

Standard ground-based defense systems (sensors and effectors mounted at $h \approx 2$ meters) suffer from a severe geometric constraint: the Earth's curvature acts as a physical shield for approaching low-altitude threats.

### 2.1 The Horizon Limit
The radar horizon (or optical horizon) imposes a hard limit on detection range for low-altitude targets. A threat flying at 100m altitude is visible to a ground sensor at a significantly shorter distance than a threat at 1,000m altitude.

**Critical Vulnerability:**
Low-altitude cruise missiles or terrain-following drones exploit the "Shadow Zone"—the volume of space below the horizon line. In this regime, the defense system is blind until the target is virtually overhead, drastically reducing the engagement window (reaction time).

### 2.2 The "Look-Angle" Limitation
Even if the horizon were flat, a ground-based emitter cannot safely steer its beam downward without striking the ground immediately in front of it.
*   **Minimum Elevation Angle ($\theta_{min}$):** Typically $0^\circ$ to $5^\circ$ to avoid ground clutter/reflection.
*   **Result:** A ground-based system effectively loses the lower hemisphere of its potential engagement volume. It defends a "Half-Dome."

## 3. The Mast Elevation Advantage

Elevating the sensor and emitter nodes onto a mast shifts the geometric baseline vertically.

### 3.1 Recovery of the Lower Hemisphere
Lifting the node allows the beam to be depressed below the horizontal plane ($\theta < 0^\circ$) while maintaining a safe distance from the ground.
*   **Geometric Gain:** The system can now "look down" into valleys, depressions, and the low-altitude approach corridors previously hidden by the horizon.
*   **Engagement Volume Expansion:** The coverage volume shifts from a "Half-Dome" to a "Deep Cone" or "Sphere-Cap" depending on the depression limit.

### 3.2 Extended Horizon Distance
While `coverage_geometry.md` details the formula, the systemic impact is an extension of the **Reaction Envelope**.
*   **Example:** A sensor at 2m height detects a 50m altitude target at ~30km. Raising the sensor to a 30m mast extends this detection range significantly, allowing the tracking system (PentaTrack) to initiate earlier and build a more accurate velocity/drift profile before the target enters the inner defense zone.

### 3.3 Mitigation of Terrain Masking
In non-flat terrain (hills, urban canyons), ground-based sensors are easily blocked by local obstacles.
*   **Elevation Advantage:** A mast lifts the node above local terrain clutter (buildings, trees) and potential multipath interference zones, establishing a clear Line-of-Sight (LoS) to the target.

## 4. The Shadow Cone: The Hidden Cost of Height

Mast elevation solves the horizon problem but introduces a new geometric artifact: **The Shadow Cone.**

### 4.1 Definition
The Shadow Cone is the volume of space immediately surrounding the mast base that the node *cannot* observe due to mechanical limits on depression angle.

$$R_{shadow} = \frac{H_{mast}}{\tan(\theta_{depression})}$$

Where:
*   $R_{shadow}$ = Radius of the blind spot on the ground.
*   $H_{mast}$ = Height of the mast.
*   $\theta_{depression}$ = Maximum angle the beam can tilt below horizontal.

### 4.2 The Trade-off Curve
*   **Taller Mast $\rightarrow$ Larger Shadow Cone.** As you raise the mast to see further, the blind spot immediately beneath you grows geometrically.
*   **Mechanical Limit:** Most optical systems cannot steer straight down ($90^\circ$ depression). Typical limits are $-30^\circ$ to $-60^\circ$.
    *   At $H_{mast} = 30\text{m}$ and $-30^\circ$ depression: $R_{shadow} \approx 52\text{m}$.
    *   At $H_{mast} = 100\text{m}$ and $-30^\circ$ depression: $R_{shadow} \approx 173\text{m}$.

### 4.3 System Implication
A threat that enters the Shadow Cone is invisible to that specific node. In a single-mast defense scenario, this is a fatal vulnerability. In a distributed array (HALO-AD), this mandates **Overlapping Coverage**, where neighboring masts are positioned such that their coverage volumes intersect the Shadow Cones of their peers.

## 5. Atmospheric Pathing Implications

Elevation changes the atmospheric physics of the beam path (see `atmospheric_models.md`).

### 5.1 Slant Path Through the Atmosphere
*   **Ground-Based:** The beam travels horizontally through the densest layer of the atmosphere (boundary layer), suffering maximum aerosol scattering and turbulence.
*   **Mast-Elevated:** The beam travels vertically out of the dense boundary layer before traversing horizontally at higher altitudes where air density is lower.
    *   **Benefit:** Reduced aerosol scattering and potentially less atmospheric attenuation for high-altitude targets.
    *   **Cost:** Increased path length for targets on the ground (slant range vs. horizontal range).

### 5.2 Thermal Plume Interaction
Heat rising from the ground (solar heating) creates localized turbulence layers near the surface. A beam originating from a tall mast may be able to "shoot over" the worst of the ground-level thermal turbulence, improving beam quality for distant targets.

## 6. Infrastructure and Stability Constraints

The simulation must account for the physical realities of mounting hardware on tall structures.

### 6.1 Vibration and Sway
A tall mast is subject to wind loading.
*   **Effect:** The beam pointing becomes unstable. If the mast sways by $0.1^\circ$, and the beam divergence is tight, the spot size at 5 km can wander off-target entirely.
*   **Simulation Requirement:** The simulator must model "Jitter" noise. Higher masts = Higher jitter amplitude (unless active stabilization is assumed).
*   **Beam Steering Correction:** The system must compensate for mast motion in real-time, effectively treating the mast itself as a moving platform.

### 6.2 Power and Data Infrastructure
*   **Ground-Based:** Easy access to power and high-bandwidth fiber.
*   **Mast-Elevated:** Requires vertical cable runs, increased structural load, and potentially wireless links for data backhaul. This increases the "Cost per Node" parameter in the simulation.

## 7. Architectural Comparison: Single Tall vs. Distributed Short

This is the core design decision for a HALO-AD deployment.

### 7.1 Strategy A: The "Lighthouse" (Single Tall Mast)
*   **Configuration:** One massive tower ($H > 50\text{m}$) with high-power sensors.
*   **Pros:** Excellent horizon range; simplifies infrastructure (one site).
*   **Cons:** Massive Shadow Cone; Single Point of Failure; High structural cost; High jitter complexity.
*   **Coverage:** Deep vertical coverage, but vulnerable to saturation and close-range threats in the shadow.

### 7.2 Strategy B: The "Fence" (Distributed Short Masts)
*   **Configuration:** Many smaller masts ($H \approx 10\text{-}20\text{m}$) distributed around the perimeter.
*   **Pros:** Small Shadow Cones; Redundancy (if one fails, neighbors cover); Lower structural cost per unit.
*   **Cons:** Requires complex Zone Handoff coordination (`zone_handoff.md`); More nodes to maintain.
*   **Coverage:** Excellent area saturation; overlapping fields eliminate blind spots.

### 7.3 Hybrid Optimization
HALO-AD allows for mixed topologies.
*   **Inner Ring:** Tall masts for high-altitude, long-range surveillance.
*   **Outer Ring:** Short masts to cover the low-altitude approaches and the shadow zones of the inner ring.

## 8. Simulation Parameters

To model these effects, the HALO-AD simulator exposes the following variables in the scenario configuration files:

| Parameter | Type | Description |
| :--- | :--- | :--- |
| `mast_height` | Float | Height of the node above local ground level (meters). |
| `depression_limit` | Float | Minimum elevation angle the beam can steer to (degrees, negative). |
| `sway_amplitude` | Float | Estimated maximum sway of the mast tip (degrees). |
| `sway_frequency` | Float | Dominant frequency of mast oscillation (Hz). |
| `stabilization_factor` | Float | Gain of the active stabilization system (damping factor). |
| `infrastructure_cost` | Float | Relative cost multiplier for the node. |

## 9. Conclusion

Mast elevation is not a "free" upgrade to coverage. It is a trade-off between **extending the horizon** and **creating a blind spot**, between **line-of-sight gain** and **stability loss**.

The HALO-AD simulation framework quantifies these trade-offs, allowing researchers to determine the optimal height distribution for a given threat profile. In almost all distributed array scenarios, a **Hybrid Strategy** proves superior—using elevation to defeat the horizon, but distributing nodes to eliminate the Shadow Cones that elevation creates.
