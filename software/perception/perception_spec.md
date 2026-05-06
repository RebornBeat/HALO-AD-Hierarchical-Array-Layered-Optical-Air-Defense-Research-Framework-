# Perception Engine Specification — `halo-perception` Crate

**Component:** `halo-perception`
**Domain:** Multi-modal perception simulation, multi-mast fusion, tracking pipeline
**Status:** Research Simulation Only — no real hardware drivers
**Version:** 1.0
**Implements:** `docs/theory/sensor_fusion.md`

---

## 1. Purpose

The perception layer sits between the **raw simulated sensors** (OMNI-SENSE) and the **predictive tracking engine** (PentaTrack).

**Responsibilities:**
1. Initialize and manage simulated sensors on each mast.
2. Simulate atmospheric degradation of sensor performance.
3. Fuse observations from multiple masts into unified tracks.
4. Transform detections into zone/district coordinate frames.
5. Feed fused tracks into PentaTrack for predictive center-field generation.

**Dependencies:** `halo-core`, `halo-atmospherics`, `omni-sense-core`, `omni-sense-radar`, `omni-sense-lidar`, `omni-sense-event`, `omni-sense-fusion`, `omni-sense-atmospherics`, `omni-sense-frames`, `omni-sense-time`, `omni-sense-drivers-mmwave` (simulation feature), `omni-sense-drivers-lidar` (simulation feature), `omni-sense-drivers-event` (simulation feature), `pentatrack`.

---

## 2. Architecture

```
Threat Generator (Truth Trajectory)
        ↓
Simulated Sensors (OMNI-SENSE Sim Impls)
        ↓ Raw Detections
Atmospheric State
        ↓ Degraded Detections
Per-Node Processing
        ↓ Local Detections
Multi-Mast Fusion (JPDA + CI)
        ↓ Fused Tracks
PentaTrack Bridge
        ↓ Predicted Center Fields
Coordinator (Zone Handoff)
```

---

## 3. Simulated Sensor Stack

HALO-AD uses three primary simulated sensor modalities.

### 3.1 Scanning LiDAR

**Implementation:** `omni-sense-drivers-lidar::SimulatedScanningLidar`

**Parameters:**
- `scan_period_ms`: Duration of one full scan sweep (e.g., 100 ms).
- `max_range_m`: Nominal maximum range (before atmospheric degradation).
- `angular_resolution_deg`: Resolution of the scan.
- `point_rate_hz`: Points per second.

**HALO-AD Specific Behavior:**
- Scan-cycle latency injection: detection timestamp reflects the moment the beam swept that angle.
- Effective range computed per-pulse based on current `AtmosphericProfile`.

**Output:** `Vec<RangeReturn>` → processed into `DetectionEvent` by `halo-perception::per_node_processing`.

### 3.2 mmWave FMCW Radar

**Implementation:** `omni-sense-drivers-mmwave::SimulatedMmWaveRadar`

**Parameters:**
- `chirp_config`: FMCW chirp parameters (bandwidth, duration).
- `num_virtual_antennas`: Array configuration for angular resolution.

**HALO-AD Specific Behavior:**
- Doppler sensitivity: direct radial velocity measurement.
- Micro-Doppler: simulates rotor-blade signatures for UAS classification.
- All-weather: unaffected by optical obscuration; slightly attenuated by heavy rain per `AtmosphericProfile::mmwave_extinction_per_m()`.

**Output:** `RangeDopplerMap` → `Vec<CfarDetection>` → `DetectionEvent`.

### 3.3 Event Camera

**Implementation:** `omni-sense-drivers-event::SimulatedEventCamera`

**Parameters:**
- `resolution`: Sensor resolution (e.g., 640×480).
- `contrast_threshold`: Trigger sensitivity.

**HALO-AD Specific Behavior:**
- Microsecond latency detection of fast transients.
- Sparse output: only events for moving targets.
- Atmospheric coupling: degrades in heavy fog/rain (reduced contrast).

**Output:** `Vec<PixelEvent>` → motion event extraction → `DetectionEvent`.

---

## 4. Atmospheric Perception Layer

### 4.1 Effective Range Calculation

For every detection, the perception layer calculates effective range based on the scenario's `AtmosphericProfile`.

```rust
fn compute_effective_range(
    nominal_range: f32,
    wavelength_nm: f32,
    atmosphere: &AtmosphericProfile
) -> f32 {
    let extinction = atmosphere.extinction_at_wavelength_nm(wavelength_nm);
    // Beer-Lambert double pass for active sensors
    // Find range where signal falls below detection threshold
    // Use binary search for efficiency
    let threshold = 0.01; // 1% of peak signal
    -threshold.ln() / (2.0 * extinction)  // simplified
}
```

### 4.2 Modality Weighting

The fusion layer weights modalities based on atmospheric state:

| Condition | LiDAR Weight | Radar Weight | Event Camera Weight |
|---|---|---|---|
| Clear | 1.0 | 0.8 | 1.0 |
| Fog / Smoke | 0.1 | 1.0 | 0.2 |
| Heavy Rain | 0.3 | 0.9 | 0.5 |

---

## 5. Multi-Mast Fusion

### 5.1 Problem: Unknown Cross-Correlation

Masts observe the same target from different angles. Observation errors are correlated (shared atmosphere, shared target motion model), but the exact correlation is unknown and hard to estimate in real-time.

### 5.2 Solution: Covariance Intersection (CI)

**Why CI?**
- Consistent: never underestimates uncertainty.
- Works without knowledge of cross-correlation.
- Robust to independent noise models of different masts.

### `struct MultiMastFuser`

Fuses `DetectionEvent` streams from all masts into a single coherent track set.

**Association strategy:** `omni-sense-fusion::JpdaTracker` for multi-target association across mast observations.

**Fusion strategy:** `omni-sense-fusion::covariance_intersection` for combining same-target observations.

```rust
fn fuse_mast_tracks(tracks: &HashMap<MastId, Track>) -> FusedTrack {
    let estimates: Vec<(Vector3<f32>, Matrix3<f32>)> = tracks.values()
        .map(|t| (t.position, t.covariance))
        .collect();
    let (fused_pos, fused_cov) = covariance_intersection_3d_multi(&estimates);
    FusedTrack { position: fused_pos, covariance: fused_cov, ... }
}
```

`fn update(&mut self, mast_detections: Vec<(MastId, Vec<DetectionEvent>)>, dt: f32) -> Vec<FusedTrack>`

`fn update_atmospheric_weights(&mut self, atmosphere: &AtmosphericProfile)`

Adjusts per-modality weights based on current atmospheric conditions.

---

## 6. Coordinate Frames

### 6.1 Frame Hierarchy

1. **World Frame:** Fixed ECEF/ENU frame.
2. **Mast Frame:** Attached to the mast structure.
3. **Sensor Frame:** Attached to the specific sensor on the mast.

### 6.2 Mast-Frame Stabilization

Masts are not perfectly rigid. Wind and structural resonance cause vibration and sway.

**Modeling Sway:** Simulated as low-frequency sinusoidal offset + high-frequency jitter. `SimulatedMast` struct updates its `pose` every simulation tick.

**Correction:** `omni-sense-frames::BodyFrameStabilizer` (configured for mast frames) subtracts IMU-estimated mast motion from sensor observations. Result: stable tracks in World/Zone frame even when the mast sways.

---

## 7. Time Synchronization

### 7.1 Simulation Clock

All simulated sensors share a monotonic simulation clock. No clock drift in simulation.

### 7.2 Latency Modeling

Transport latency is modeled even though the clock is perfect:

```toml
[perception.network]
link_latency_ms = 5
jitter_ms = 1
```

---

## 8. Per-Node Processing

### `struct SimulatedMastSensorArray`

Aggregates all simulated sensors for a single mast. Constructed from `MastConfig` and current `AtmosphericProfile`.

`fn poll_all(&mut self, scenario_state: &ScenarioState) -> Vec<DetectionEvent>`

Polls all sensors, applies atmospheric degradation, transforms to zone frame via `omni-sense-frames`, returns all `DetectionEvent` values for this mast at the current simulation time.

`fn build_sensor_array(mast: &MastConfig, atmosphere: &AtmosphericProfile) -> SimulatedMastSensorArray`

Factory function. Configures simulation parameters from mast config.

---

## 9. Global Track Manager

### `struct GlobalTrackManager`

Manages track lifecycle over the full simulation run.

- **Track initiation:** N-of-M rule over recent frames.
- **Track maintenance:** Kalman prediction between detection updates.
- **Track deletion:** Track coasted for longer than `max_coast_time`.

Underlying filter: `omni-sense-fusion::KalmanFilter6` with state `[x, y, z, vx, vy, vz]`.

`fn active_tracks(&self) -> &[Track]`

---

## 10. PentaTrack Bridge

### `struct HaloAdPentaTrackBridge`

Bridges `FusedTrack` output into PentaTrack's prediction tree.

Maintains one `PentaTracker` per active track. HALO-AD configuration: depth 2–3, diagonals enabled, cosine-softmax weighting, LSQ velocity estimation, UAS drift profiles.

```rust
impl From<FusedTrack> for DetectionEvent {
    fn from(track: FusedTrack) -> Self {
        DetectionEvent {
            position: track.position,
            velocity: track.velocity,
            position_covariance: track.covariance,
            confidence: track.confidence,
            ..Default::default()
        }
    }
}
```

`fn update_all(&mut self, fused_tracks: &[FusedTrack]) -> Vec<(TrackId, PentaPrediction)>`

`fn get_intercept_for_track(&self, track_id: TrackId, engagement_range_m: f32, effector_speed_ms: f32) -> Option<InterceptPoint>`

---

## 11. Simulation Injection Interface

The threat generator (`halo-simulator::ThreatGenerator`) injects threats into `ScenarioState`. The sensor simulation produces detections consistent with threat position, velocity, and type — including detection noise, range-dependent miss probability, and atmospheric degradation. The perception crate **never** directly reads the true threat position; it only sees what the simulated sensors report.

---

## 12. Configuration

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
