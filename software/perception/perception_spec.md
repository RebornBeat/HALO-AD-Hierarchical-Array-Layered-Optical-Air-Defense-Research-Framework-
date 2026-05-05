# HALO-AD Perception Specification

**Component:** `halo-perception` crate
**Domain:** Multi-modal perception simulation, multi-mast fusion, and tracking pipeline
**Status:** Research Simulation Only
**Version:** 1.0

---

## 1. Purpose

This document specifies the perception layer for the HALO-AD simulator. The perception layer sits between the **raw simulated sensors** (provided by OMNI-SENSE) and the **predictive tracking engine** (PentaTrack).

Its responsibilities are:

1.  **Initialize and manage simulated sensors** on each mast.
2.  **Simulate atmospheric degradation** of sensor performance.
3.  **Fuse observations from multiple masts** into unified tracks.
4.  **Transform detections** into the correct coordinate frames (Zone/District frames).
5.  **Feed the fused tracks** into PentaTrack for predictive center-field generation.

**Scope Constraint:** This layer is purely for simulation. It does not interact with real hardware. It consumes `SimulatedSensor` traits from OMNI-SENSE.

---

## 2. Architecture

The perception pipeline is structured as a series of transformations:

```
Threat Generator (Truth Trajectory)
        ↓
Simulated Sensors (OMNI-SENSE Sim Impls)
        ↓ Raw Detections
Atmospheric State
        ↓ Degraded Detections
Per-Node Processing
        ↓ Local Detections
Multi-Mast Fusion
        ↓ Fused Tracks
PentaTrack Bridge
        ↓ Predicted Center Fields
Coordinator (Zone Handoff)
```

---

## 3. Simulated Sensor Stack

HALO-AD uses three primary simulated sensor modalities. All are provided by the `omni-sense-drivers-*` crates with the `simulation` feature enabled.

### 3.1 Scanning LiDAR

**Implementation:** `omni-sense-drivers-lidar::SimulatedScanningLidar`

**Parameters:**
- `scan_period_ms`: Duration of one full scan sweep (e.g., 100 ms).
- `max_range_m`: Nominal maximum range (before atmospheric degradation).
- `angular_resolution_deg`: Resolution of the scan.
- `point_rate_hz`: Points per second.

**HALO-AD Specific Behavior:**
- The simulator injects **scan-cycle latency**. A detection's timestamp reflects the moment the beam swept that angle, not the frame end.
- The **effective range** is computed per-pulse based on the current `AtmosphericProfile`.

**Output:** `Vec<RangeReturn>`

---

### 3.2 mmWave FMCW Radar

**Implementation:** `omni-sense-drivers-mmwave::SimulatedMmWaveRadar`

**Parameters:**
- `chirp_config`: FMCW chirp parameters (bandwidth, duration).
- `num_virtual_antennas`: Array configuration for angular resolution.

**HALO-AD Specific Behavior:**
- **Doppler Sensitivity:** Simulates direct radial velocity measurement.
- **Micro-Doppler:** Simulates rotor-blade signatures for UAS classification.
- **All-Weather:** Unaffected by optical obscuration (fog/smoke), but slightly attenuated by heavy rain.

**Output:** `RangeDopplerMap` → `Vec<CfarDetection>`

---

### 3.3 Event Camera

**Implementation:** `omni-sense-drivers-event::SimulatedEventCamera`

**Parameters:**
- `resolution`: Sensor resolution (e.g., 640x480).
- `contrast_threshold`: Trigger sensitivity.

**HALO-AD Specific Behavior:**
- **Microsecond Latency:** Detects fast transients (supersonic threats) where LiDAR is too slow.
- **Sparse Output:** Only generates events for moving targets.
- **Atmospheric Coupling:** Degrades gracefully in heavy fog/rain (reduced contrast).

**Output:** `Vec<PixelEvent>`

---

## 4. Atmospheric Perception Layer

This module applies atmospheric effects to sensor outputs. It is unique to HALO-AD due to the long-range, outdoor nature of the simulation.

### 4.1 Effective Range Calculation

For every detection, the perception layer calculates an effective range based on the scenario's `AtmosphericProfile`.

```rust
// Logic in halo-perception::atmospheric_coupling
fn compute_effective_range(
    nominal_range: f32,
    wavelength_nm: f32,
    atmosphere: &AtmosphericProfile
) -> f32 {
    let extinction = atmosphere.extinction_at_wavelength(wavelength_nm);
    let transmission = (-extinction * nominal_range).exp(); // Beer-Lambert
    // Double pass for active sensors
    let double_pass_transmission = transmission.powi(2);
    // Find range where signal falls below threshold
    // ... iterative or analytical solver ...
}
```

### 4.2 Modality Weighting

The fusion layer weights modalities based on atmospheric state:

| Condition        | LiDAR Weight | Radar Weight | Event Camera Weight |
| ---------------- | ------------ | ------------ | ------------------- |
| Clear            | 1.0          | 0.8          | 1.0                 |
| Fog / Smoke      | 0.1          | 1.0          | 0.2                 |
| Heavy Rain       | 0.3          | 0.9          | 0.5                 |

---

## 5. Multi-Mast Fusion

HALO-AD's core perception challenge is fusing data from **multiple, distributed masts**.

### 5.1 Problem: Unknown Cross-Correlation

Masts observe the same target from different angles. Their observation errors are correlated (shared atmosphere, shared target motion model), but the exact correlation is unknown and hard to estimate in real-time.

### 5.2 Solution: Covariance Intersection (CI)

We use **Covariance Intersection** (implemented in `omni-sense-fusion::covariance_intersection`) to fuse tracks from different masts.

**Why CI?**
- It is **consistent**: It never underestimates uncertainty.
- It works **without knowledge of cross-correlation**.
- It is robust to the independent noise models of different masts.

**Implementation:**
```rust
// In halo-perception::multi_mast_fusion
fn fuse_mast_tracks(
    tracks: &HashMap<MastId, Track>
) -> FusedTrack {
    // Extract position estimates and covariances
    let estimates: Vec<(Vector3<f32>, Matrix3<f32>)> = tracks.values()
        .map(|t| (t.position, t.covariance))
        .collect();

    // Apply CI to get fused state
    let (fused_pos, fused_cov) = covariance_intersection(&estimates);

    FusedTrack { position: fused_pos, covariance: fused_cov, ... }
}
```

### 5.3 Track Association

We use **Joint Probabilistic Data Association (JPDA)** for associating new detections to existing tracks across the mast array.

---

## 6. Coordinate Frames

### 6.1 Frame Hierarchy

1.  **World Frame:** Fixed ECEF/ENU frame.
2.  **Mast Frame:** Attached to the mast structure.
3.  **Sensor Frame:** Attached to the specific sensor on the mast.

### 6.2 Mast-Frame Stabilization

Masts are not perfectly rigid. Wind and structural resonance cause **vibration and sway**.

**Modeling Sway:**
- Simulated as a low-frequency sinusoidal offset + high-frequency jitter.
- The `SimulatedMast` struct updates its `pose` every simulation tick.

**Correction:**
- The perception layer uses `omni-sense-frames::BodyFrameStabilizer` (configured for mast frames).
- It subtracts the mast's IMU-estimated motion from the sensor observation.
- **Result:** Stable tracks in the World/Zone frame, even if the mast is swaying.

---

## 7. Time Synchronization

Masts are geographically distributed. Synchronization is critical.

### 7.1 Simulation Clock

All simulated sensors share a **monotonic simulation clock**. There is no clock drift in the simulation.

### 7.2 Latency Modeling

While the clock is perfect, the **transport latency** is modeled:

```toml
[perception.network]
link_latency_ms = 5
jitter_ms = 1
```

This delay is injected between the sensor event generation and the fusion center receiving it.

---

## 8. PentaTrack Integration

The `PentaTrackBridge` is the final stage of the perception layer.

### 8.1 Conversion

Converts `FusedTrack` (from multi-mast fusion) into `DetectionEvent` for PentaTrack.

```rust
impl From<FusedTrack> for DetectionEvent {
    fn from(track: FusedTrack) -> Self {
        DetectionEvent {
            position: track.position,
            velocity: track.velocity,
            covariance: track.covariance,
            confidence: track.confidence,
            source: "HALO-AD-Fusion".into(),
            timestamp: track.timestamp,
            ..Default::default()
        }
    }
}
```

### 8.2 HALO-AD Parameters

The bridge configures PentaTrack for the defense scenario:

- **Recursion Depth:** 2-3 (predict ahead for engagement timeline).
- **Drift Profiles:** `LowAltitudeUAS`, `TerrainFollowing`, `Hypersonic`.
- **Adaptive Drift:** Enabled (for maneuvering threats).
- **Diagonals:** Enabled (3D maneuvering).

---

## 9. Configuration

```toml
[perception]
update_rate_hz = 20
fusion_method = "CovarianceIntersection"
association_method = "JPDA"

[perception.lidar]
type = "Scanning"
scan_period_ms = 100
max_range_m = 5000

[perception.radar]
type = "FMCW"
max_range_m = 10000
enable_micro_doppler = true

[perception.event_camera]
enabled = true
trigger_threshold = "FastObject"

[perception.sync]
source = "SimClock"
latency_model = "Deterministic"
link_latency_ms = 5
```
