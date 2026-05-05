# HALO-AD Simulator API Reference

**Implementation:** `halo-cli` binary, `halo-sdk` crate, `halo-simulator::harness`

---

## CLI Interface

### `halo-cli run <scenario.toml>`

Runs a single scenario and produces output in `runs/<scenario_name>_<timestamp>/`.

**Arguments:**
- `--output-dir <path>` — Override the default output directory.
- `--seed <u64>` — Random seed for deterministic simulation. Defaults to current timestamp.
- `--verbose` — Emit per-timestep metrics to stdout in addition to output files.

**Example:**
```bash
halo-cli run scenarios/mast_elevated.toml --seed 42
```

---

### `halo-cli compare <scenario.toml>`

Runs a scenario against multiple topologies and produces a comparison report.

**Arguments:**
- `--topologies <list>` — Comma-separated list of topologies. Values: `GroundStatic`, `MastElevated`, `ConeToCenter`, `ParallelArray`, `DivergentFan`. Defaults to all five.
- `--output <path>` — Output HTML report path.
- `--seed <u64>` — Same seed used for all topology runs (ensures identical threat trajectory injection).

**Example:**
```bash
halo-cli compare scenarios/baseline.toml --topologies GroundStatic,MastElevated,ConeToCenter
```

---

### `halo-cli report`

Aggregates results from a `runs/` directory into an HTML report.

**Arguments:**
- `--inputs <path>` — Directory containing run subdirectories.
- `--output <path>` — Output HTML report path.

---

## Scenario File Format (TOML)

### Top-level fields

```toml
[scenario]
name = "My Scenario"
duration_seconds = 120.0
voxel_resolution = 0.5          # Meters per voxel side
random_seed = 42                # Optional; deterministic if specified
```

### Region

```toml
[region]
bounds_m = [5000.0, 5000.0, 1000.0]  # x, y, z in meters
```

### Topology

```toml
[topology]
type = "MastElevated"           # GroundStatic | MastElevated | ConeToCenter | ParallelArray | DivergentFan
mast_height_m = 30.0            # For MastElevated, ConeToCenter, DivergentFan
focal_point_m = [2500.0, 2500.0, 200.0]  # For ConeToCenter only
array_direction = [1.0, 0.0, 0.0]         # For ParallelArray only
array_spacing_m = 500.0                    # For ParallelArray only
inner_angle_deg = 30.0                     # For DivergentFan only
outer_angle_deg = 90.0                     # For DivergentFan only
```

### Masts

```toml
[[masts]]
id = "mast_north"
position_m = [500.0, 4500.0, 0.0]          # Base position (elevation is added by topology)
sensors = ["MmWaveRadar", "ScanningLidar", "EventCamera"]   # Sensor types installed
depression_limit_degrees = 45.0             # Maximum below-horizontal depression angle
beam_quality_m2 = 1.5                       # For optical sensors
wavelength_nm = 1550.0                      # For optical sensors
```

### Threat Trajectories

```toml
[[threat_trajectories]]
type = "LinearFlight"                       # LinearFlight | TerrainFollowing | Evasive | Swarm
entry_point_m = [0.0, 2500.0, 50.0]
exit_point_m = [5000.0, 2500.0, 50.0]
velocity_ms = 30.0
uas_class = "ConsumerMultiRotor"            # Determines PentaTrack drift profile

[[threat_trajectories]]
type = "TerrainFollowing"
terrain_clearance_m = 10.0
horizontal_path = [[0.0, 500.0], [1000.0, 1500.0], [3000.0, 1000.0]]
velocity_ms = 20.0
uas_class = "LowAltitudeUAS"

[[threat_trajectories]]
type = "Evasive"
entry_point_m = [0.0, 1000.0, 100.0]
exit_point_m = [5000.0, 1000.0, 100.0]
nominal_velocity_ms = 40.0
max_turn_rate_deg_per_sec = 60.0            # Maneuver aggressiveness
uas_class = "HighPerformance"
```

### Atmosphere

```toml
[atmosphere]
profile = "Clear"    # Clear | Hazy | LightFog | HeavyFog | LightRain | Smoke
# For Custom profile:
# wavelength_extinction = { "1550" = 0.001, "77000" = 0.00001 }  # nm → 1/m
```

### Zones

```toml
[[zones]]
id = "northern_sector"
bounds_m = [[0.0, 2500.0, 0.0], [5000.0, 5000.0, 1000.0]]
assigned_masts = ["mast_north", "mast_northeast"]
coverage_depth_required = 2
```

### Tracking

```toml
[tracking]
enabled = true
pentatrack_depth = 3
enable_diagonals = true
enable_velocity_weighting = true
enable_drift_analysis = true
enable_adaptive_drift = true
```

---

## Output Format Reference

Each `halo-cli run` produces a directory containing:

### `metrics_summary.json`

```json
{
  "scenario_name": "string",
  "topology": "MastElevated",
  "duration_seconds": 120.0,
  "coverage_fraction": 0.78,
  "mean_coverage_depth": 2.3,
  "time_to_first_track_ms": {
    "mean": 340.0,
    "p50": 280.0,
    "p90": 620.0,
    "p99": 1200.0
  },
  "track_continuity_fraction": 0.91,
  "handoff_success_rate": 0.96,
  "handoff_latency_ms": {
    "mean": 85.0,
    "p90": 180.0
  },
  "atmospheric_effective_range_m": {
    "MmWaveRadar": 8500.0,
    "ScanningLidar": 3200.0,
    "EventCamera": 4100.0
  }
}
```

### `coverage_report.csv`

One row per voxel in the engagement volume:
```
voxel_x_m, voxel_y_m, voxel_z_m, observable_count, mast_ids, coverage_depth, atmospheric_quality
```

### `track_log.csv`

One row per tracking frame per threat:
```
timestamp_ns, threat_id, true_x_m, true_y_m, true_z_m, true_vx_ms, true_vy_ms, true_vz_ms, estimated_x_m, estimated_y_m, estimated_z_m, position_error_m, track_state, pentatrack_best_center_confidence
```

### `handoff_log.csv`

One row per zone handoff event:
```
timestamp_ns, threat_id, source_zone, destination_zone, trigger_type, state_at_completion, latency_ms, success
```

### `assignment_trace.csv` (when `[tracking]` enabled)

One row per assignment per frame:
```
timestamp_ns, threat_id, effector_slot_id, addressed_center_label, los_quality, predicted_center_confidence, slew_time_ms
```

---

## SDK Crate API (`halo-sdk`)

The `halo-sdk` crate exposes the simulation internals for custom scenario development in Rust.

### Key types

```rust
use halo_sdk::prelude::*;

// Load and run a scenario programmatically
let config = ScenarioConfig::from_toml("scenarios/my_scenario.toml")?;
let mut harness = SimulatorHarness::new(config);
harness.initialize()?;

// Run simulation and collect metrics
let metrics = harness.run_to_completion()?;
println!("Coverage fraction: {}", metrics.coverage_fraction);

// Access per-frame state during a run
harness.run_with_callback(|frame| {
    println!("Frame {}: {} active tracks", frame.timestamp_ns, frame.tracks.len());
    for track in &frame.tracks {
        println!("  Track {}: est_pos={:?}", track.id, track.estimated_position);
    }
    ControlFlow::Continue
})?;
```

### Custom threat generators

```rust
use halo_sdk::threat::{ThreatGenerator, ThreatFrame};

struct MyCustomThreat { /* ... */ }

impl ThreatGenerator for MyCustomThreat {
    fn next_frame(&mut self, dt: f32) -> Option<ThreatFrame> {
        // Return None when threat exits the scenario
        Some(ThreatFrame {
            threat_id: self.id,
            position_m: self.compute_position(dt),
            velocity_ms: self.velocity,
            uas_class: UasClass::ConsumerMultiRotor,
        })
    }
}

// Register and run with custom threat
harness.add_threat_generator(Box::new(MyCustomThreat::new()));
```

### Custom effector configurations

```rust
use halo_sdk::effector::EffectorDescriptor;

let custom_array = EffectorDescriptor {
    id: EffectorId::from("my_4_element_array"),
    position_m: Vector3::new(500.0, 500.0, 30.0),
    element_count: 4,
    field_of_regard_degrees: 120.0,
    slew_rate_deg_per_sec: 60.0,
    nominal_range_m: 500.0,
    engagement_cycle_time_ms: 50.0,
};

harness.add_effector(custom_array);
```
