# Sensor Fusion — Multi-Modal Long-Range Outdoor Perception

**Project:** HALO-AD
**Domain:** Multi-modal sensor fusion at engagement timescales for distributed outdoor coverage
**Status:** Theoretical / Simulation Framework

## 1. Purpose

This document characterizes HALO-AD's approach to multi-modal sensor fusion in the long-range outdoor coverage context. It defines the architecture for combining disparate sensing modalities—Event-Based Vision, mmWave Doppler Radar, Scanning LiDAR, and Acoustic Echo profiling—into a unified perception layer.

Unlike indoor sensing (AEGIS-MESH), HALO-AD operates in a regime defined by **extreme range**, **atmospheric interference**, and **high-velocity targets**. The fusion architecture must solve the "Latency vs. Resolution" trade-off: fast sensors (Radar/Event) provide immediate detection but low detail, while slow sensors (LiDAR) provide high detail but introduce scan-cycle lag.

This document specifies what each modality contributes, how the fusion is structured, how it composes with PentaTrack's predictive-tracking layer, and how it handles the unique constraints of mast-mounted distributed arrays.

## 2. The Sensing Modalities

HALO-AD’s perception layer is built on four primary sensing modalities. Each modality occupies a specific niche in the "Latency vs. Resolution" matrix.

### 2.1 Event-Based Vision (Neuromorphic)

Event-based image sensors (e.g., Prophesee, iniVation) emit asynchronous events when individual pixels detect illumination changes. They do not record "frames"; they record "change."

*   **Strengths:**
    *   **Microsecond Latency:** Detection is essentially instantaneous for moving objects. This is critical for crossing targets that traverse the field of view in milliseconds.
    *   **High Dynamic Range (HDR):** Typically 120dB+, allowing operation in high-contrast environments (e.g., looking from shadow into bright sky).
    *   **Data Efficiency:** Sparse output (only moving objects generate data) reduces processing load.
*   **Weaknesses:**
    *   **Requires Motion:** Static targets are invisible.
    *   **Atmospheric Sensitivity:** Light is scattered by fog, smoke, and dust.
*   **Role in HALO-AD:**
    *   **Primary Trigger:** The "First Photon" detection layer. It alerts the system that a fast object has entered the volume, initiating the track.

### 2.2 mmWave Doppler Radar

Frequency-Modulated Continuous Wave (FMCW) radar operating at 24, 60, 77, or 94 GHz. It emits radio waves and analyzes the frequency shift (Doppler effect) of returns.

*   **Strengths:**
    *   **Direct Velocity Readout:** Doppler shift provides radial velocity instantly, without frame-to-frame differentiation.
    *   **All-Weather Robustness:** RF penetrates fog, rain, smoke, and dust with minimal attenuation.
    *   **Velocity Sensitivity:** Can detect the specific rotor signature of small UAS (micro-Doppler).
*   **Weaknesses:**
    *   **Low Spatial Resolution:** Antenna beamwidth limits angular resolution. It knows *something* is there and how fast it is moving, but not precisely *what* shape it is.
*   **Role in HALO-AD:**
    *   **Coarse Tracker:** Maintains track on targets when optical sensors are blinded by weather.
    *   **Velocity Anchor:** Provides ground-truth velocity data to the PentaTrack prediction engine.

### 2.3 Scanning LiDAR (Light Detection and Ranging)

Time-of-flight systems that emit laser pulses (typically 905nm or 1550nm) and measure return time.

*   **Strengths:**
    *   **Geometric Precision:** Centimeter-level range accuracy.
    *   **High Resolution:** Dense point clouds allow for shape classification (e.g., distinguishing a bird from a drone).
*   **Weaknesses:**
    *   **Scan Latency:** Mechanical or MEMS scanning creates a "time lag" between the first point and the last point in a scan cycle (typically 50-100ms).
    *   **Atmospheric Failure:** Light scatters heavily in fog/rain.
*   **Role in HALO-AD:**
    *   **Geometry Refiner:** Once a track is initiated, LiDAR refines the position and classifies the object type.
    *   **Volume Mapping:** Maps the background environment (terrain, obstacles) for the coverage geometry engine.

### 2.4 Acoustic Echo Profiling (Sonar)

While traditionally associated with underwater sensing, atmospheric acoustic sensing provides unique capabilities in the lower atmosphere.

*   **Strengths:**
    *   **Material Penetration:** Sound propagates through aerosols (smoke/dust) that blind optical systems.
    *   **Full Echo Analysis:** Unlike simple ranging, analyzing the *full echo return* (intensity over time, frequency shifts) reveals:
        *   **Surface Roughness:** Texture signatures.
        *   **Material Density:** Distinguishing a plastic drone body from a metal casing.
        *   **Internal Structure:** Resonances within a target.
*   **Weaknesses:**
    *   **Low Resolution:** Wavelengths are orders of magnitude larger than light, limiting spatial detail.
    *   **Speed of Sound:** Propagation is slow (~343 m/s). For long-range detection, the round-trip time is significant (e.g., ~3 seconds for 1 km range).
*   **Role in HALO-AD:**
    *   **Short-Range Perimeter Defense:** Monitoring the "Shadow Cone" directly beneath the mast where optical angles fail.
    *   **Classification Aid:** Providing material classification hints to disambiguate decoys from threats.

## 3. The Fusion Architecture

The fusion pipeline is structured hierarchically to prioritize speed for detection and accuracy for tracking.

### 3.1 Layer 1: The "Fast Trigger" (Event + Radar)

This layer operates on the microsecond-to-millisecond timescale.
*   **Process:** The Event Camera detects a motion edge. Simultaneously, the Radar detects a Doppler velocity anomaly.
*   **Fusion:** The system fuses these to create a **"Coarse Track"** (Position estimate with high uncertainty + Velocity estimate with high certainty).
*   **Output:** A "Wake Up" signal is sent to the LiDAR to slew towards the coarse coordinates.

### 3.2 Layer 2: The "Geometry Lock" (LiDAR)

This layer operates on the 10-100ms timescale.
*   **Process:** The LiDAR scans the volume indicated by Layer 1.
*   **Fusion:** The high-resolution point cloud is merged with the Coarse Track. The track uncertainty collapses.
*   **Output:** A precise 3D bounding box is generated.

### 3.3 Layer 3: The "Deep Analysis" (Full Echo & Acoustic)

This layer operates asynchronously to classify the target.
*   **Process:** If the target is within range, acoustic sensors analyze the echo profile. Radar analyzes micro-Doppler signatures.
*   **Fusion:** Material properties and engine signatures are added to the track metadata.
*   **Output:** Target Classification (e.g., "Drone, Plastic Frame, Rotor-Engaged").

## 4. Latency Budgeting & Reaction Time

A critical capability of the fusion engine is the calculation of the **Reaction Time Budget**. The system must know if it can physically react to a threat before that threat exits the engagement volume.

**Formula:**
$$T_{reaction} = T_{detect} + T_{process} + T_{steer}$$

*   **$T_{detect}$ (Sensor Latency):**
    *   Event Camera: ~1 µs
    *   Radar: ~1 ms
    *   LiDAR: ~50-100 ms (scan cycle)
*   **$T_{process}$ (Fusion Latency):** ~5-10 ms (compute time for PentaTrack update).
*   **$T_{steer}$ (Actuation Latency):** Time to mechanically or electronically point the effector.

**Constraint:**
If $T_{reaction} > \frac{Distance_{target}}{Velocity_{target}}$, the system is "Reaction Limited."

**HALO-AD Optimization:** The simulator uses the Event+Radar fusion to minimize $T_{detect}$, and Solid-State arrays to minimize $T_{steer}$, ensuring the Reaction Time Budget is satisfied for hypersonic or fast-crossing targets.

## 5. Mast-Frame Stabilization (Drift Compensation)

Mast-mounted sensors face environmental motion:
*   **Wind Sway:** Masts oscillate in wind.
*   **Vibration:** Active cooling or machinery generates vibration.
*   **Thermal Expansion:** Structures expand/contract with temperature.

This introduces **Drift** into the sensor readings relative to the ground frame.

### 5.1 The "Body-Frame" Problem
This is analogous to the "Body-Frame Drift" in SENTINEL-WEAR (wearables). The sensor is moving relative to the world, but the readings appear as if the world is moving.

### 5.2 Stabilization Architecture
1.  **High-Frequency IMU:** Each mast node contains an Inertial Measurement Unit (IMU) sampling at >200 Hz.
2.  **Drift Subtraction:** The IMU measures mast sway in real-time. The fusion engine subtracts this "ego-motion" from the Event and LiDAR streams.
3.  **Radar Anchor:** Since Radar measures velocity relative to the ground (Doppler), it acts as a "truth anchor" to correct drift in the optical tracks.

**Outcome:** A stable, ground-referenced track is maintained even if the mast is swaying violently in wind.

## 6. Atmospheric Coupling & Degradation Logic

The fusion engine actively monitors atmospheric conditions to weight sensor inputs.

*   **Clear Air:**
    *   LiDAR Weight: HIGH
    *   Radar Weight: MEDIUM
    *   Event Weight: HIGH
*   **Fog / Rain / Dust:**
    *   LiDAR Weight: LOW (Signal attenuated)
    *   Radar Weight: HIGH (RF penetrates)
    *   Event Weight: LOW (Obscured optics)
*   **Smoke / Chemical Obscurant:**
    *   LiDAR Weight: ZERO
    *   Radar Weight: HIGH
    *   Acoustic Weight: MEDIUM (Sound penetrates smoke better than light).

The simulator dynamically adjusts these weights in real-time based on the scenario parameters.

## 7. Multi-Mast Track Fusion

In the distributed array topology, multiple masts observe the same target from different angles.

### 7.1 Geometric Diversity
Mast A sees the target from the left; Mast B sees it from the right.
*   **Benefit:** Shadows (occlusion) on one side are visible from the other.
*   **3D Triangulation:** Intersection of bearing lines from multiple masts provides high-accuracy 3D position without requiring high-resolution range-finding on a single mast.

### 7.2 Handoff Synchronization
As described in `zone_handoff.md`, tracks are handed between masts.
*   **Prediction Extrapolation:** When Mast A hands off to Mast B, Mast B uses PentaTrack to predict where the target *will be* when it enters FoV, pre-positioning its scan window to reduce acquisition time.

## 8. Full Echo Analysis (Material & Density Hints)

Incorporating the "Rich Echo" capabilities of Radar and Acoustic sensors, the fusion engine adds a classification layer:

*   **Echo Intensity:** Indicates surface reflectivity (Metal vs. Plastic vs. Wood).
*   **Frequency Shift (Non-Doppler):** Acoustic returns shift frequency when hitting soft materials (cloth) vs. hard materials (steel).
*   **Multi-Bounce:** Complex echo profiles (multiple return peaks) indicate complex internal geometry or cavities (e.g., "Hollow" vs. "Solid").

**Simulation Output:** The track object is tagged with a `material_probability` vector (e.g., `{metal: 0.1, plastic: 0.8, biological: 0.1}`).

## 9. Civilian Transfer Applications

The fusion architecture developed for HALO-AD transfers directly to:

1.  **Autonomous Driving:** Fusing LiDAR, Radar, and Cameras for robust perception in rain/fog.
2.  **Air Traffic Management:** Multi-sensor fusion for tracking aircraft in all weather conditions.
3.  **Maritime Navigation:** Radar + Optical fusion for collision avoidance at sea.
4.  **Industrial Safety:** Monitoring exclusion zones in factories where dust or steam blinds standard cameras.

## 10. Conclusion

The HALO-AD sensor fusion layer is not merely a sum of parts; it is an orchestrated hierarchy designed to defeat latency and atmospheric degradation. By prioritizing **Event-Based Vision** for speed, **Radar** for weather robustness, **LiDAR** for geometry, and **Acoustics** for material analysis, the system creates a perception bubble that is resilient to the failure of any single modality. This fused data feeds the PentaTrack predictive engine, ensuring that the coordination layer has the highest possible quality input to make assignment decisions.
