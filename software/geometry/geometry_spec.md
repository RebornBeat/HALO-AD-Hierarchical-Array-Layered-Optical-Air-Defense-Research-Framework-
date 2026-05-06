# Geometry Engine Specification — `halo-geometry` Crate

**Component:** `halo-geometry`
**Version:** 0.1.0
**Status:** Implementation Specification
**Implements:** `docs/theory/coverage_geometry.md`, `docs/theory/mast_elevation_analysis.md`

---

## 1. Overview

The Geometry Engine is the computational core of HALO-AD. It provides all physics-correct geometric computations the simulator requires — sensor coverage, line-of-sight (LOS), atmospheric beam propagation, and optimal mast placement/orientation.

It is a **pure library crate** with no I/O, no side effects, and no simulation state. Every function is a pure computation over well-defined inputs. While `docs/theory/coverage_geometry.md` establishes the physical laws and mathematical foundations, this document specifies the **implementation behavior**: data structures, algorithmic choices, input/output contracts, and performance requirements.

**Dependencies:** `halo-core`, `omni-sense-atmospherics`, `omni-sense-frames`, `nalgebra`.

---

## 2. Core Data Structures

All structures are defined in `halo-core::geometry` and consumed by `halo-geometry`.

### 2.1 VoxelMap

The fundamental spatial representation. The 3D volume of interest is discretized into a grid of voxels (cubes).

```rust
pub struct VoxelMap {
    pub bounds: Bounds,
    pub resolution: f32,       // Edge length of each voxel (meters)
    pub data: Vec<VoxelState>, // Flattened 3D array
}

pub enum VoxelState {
    Open,
    Structure,     // Ground/buildings (permanent)
    Obstruction,   // Furniture, mobile objects
    Unknown,
}

pub struct Bounds {
    pub min: Vector3<f32>,
    pub max: Vector3<f32>,
}
```

**Implementation Notes:**
- `Vec<VoxelState>` is memory-intensive for large volumes. For sparse environments, a hierarchical representation (Octree) or spatial hashing is a future optimization path.
- `(x, y, z)` world coordinates map to `(i, j, k)` indices via `floor((coord - min) / resolution)`.

### 2.2 MastDescriptor

```rust
pub struct MastDescriptor {
    pub id: String,
    pub position_m: Vector3<f32>,
    pub height_m: f32,
    pub sensors: Vec<SensorConfig>,
    pub depression_limit_deg: f32,
}

pub struct SensorConfig {
    pub sensor_type: SensorType,
    pub orientation: Vector3<f32>,
    pub beam_width: f32,
}
```

### 2.3 CoverageReport

```rust
pub struct CoverageReport {
    pub total_voxels: u64,
    pub observable_voxels: u64,
    pub coverage_fraction: f32,
    pub voxel_depth_histogram: HashMap<u8, u64>,
    pub blind_spots: Vec<VoxelCoord>,
    pub per_mast_coverage: HashMap<String, f32>,
}
```

---

## 3. Module: `horizon`

### `earth_curvature_horizon(sensor_height_m: f32, target_altitude_m: f32) -> f32`

Returns the horizon distance in kilometers.

**Formula:** $d = 3.57 \times (\sqrt{h_s} + \sqrt{h_t})$

Panics if either argument is negative.

### `mast_horizon_gain(ground_height_m: f32, mast_height_m: f32, target_altitude_m: f32) -> f32`

Returns the additional horizon distance (km) gained by lifting a sensor from `ground_height_m` to `mast_height_m`. Always non-negative.

**Implementation Logic:**
1. Check if `target_altitude_m` is below the minimum detectable altitude for the sensor height.
2. Return 0.0 if target is geometrically impossible to observe (below horizon).

---

## 4. Module: `shadow_cone`

### `mast_shadow_cone_radius(mast_height_m: f32, depression_limit_degrees: f32) -> f32`

Returns shadow cone radius at ground level.

**Formula:** $r = H \times \tan(90° - \alpha)$

### `is_in_shadow_cone(mast_position: Point3<f32>, mast_height_m: f32, depression_limit_degrees: f32, query_point: Point3<f32>) -> bool`

Returns true if `query_point` falls within the shadow cone. Used by the coverage analyzer to flag uncovered shadow regions.

**Output:**
```rust
pub struct Cone {
    pub origin: Vector3<f32>,
    pub direction: Vector3<f32>, // Typically (0, -1, 0)
    pub angle_rad: f32,
}
```

---

## 5. Module: `beam_propagation`

### `gaussian_beam_radius(initial_radius_m: f32, distance_m: f32, divergence_half_angle_rad: f32, beam_quality_m2: f32) -> f32`

Linear approximation valid for `z >> z_R`:
$$w(z) = w_0 + z \cdot M^2 \cdot \lambda / (\pi \cdot w_0)$$

### `gaussian_beam_radius_full(waist_radius_m: f32, distance_m: f32, wavelength_m: f32, beam_quality_m2: f32) -> f32`

Full Gaussian beam including Rayleigh range:
$$w(z) = w_0 \times \sqrt{1 + (z/z_R)^2}, \quad z_R = \pi w_0^2 / (M^2 \lambda)$$

### `power_density_at_range(total_power_w: f32, beam_radius_m: f32) -> f32`

Returns W/m²: $P / (\pi \times w^2)$

### `effective_range_given_atmosphere(sensor_nominal_range_m: f32, wavelength_m: f32, aperture_m: f32, beam_quality_m2: f32, atmosphere: &AtmosphericProfile, threshold_fraction: f32) -> f32`

Combines beam divergence and atmospheric attenuation to compute usable range.

**Steps:**
1. Beam divergence: $w(z) = w_0 + z \cdot \theta$
2. Power density: $I(z) = P_{total} / (\pi \cdot w(z)^2)$
3. Atmospheric transmission (Beer-Lambert): $T(z) = e^{-\alpha z}$
4. Iterate until $I(z) \cdot T(z)^2 < threshold$ (double pass for active sensors)

**Implementation Note:** Use binary search or Newton's method, not linear stepping.

### `atmospheric_path_length(mast_height_m: f32, target_altitude_m: f32, horizontal_range_m: f32) -> f32`

Returns 3D path length: $\sqrt{r^2 + (H - h_t)^2}$

---

## 6. Module: `coverage_analyzer`

### `struct CoverageAnalyzer`

**Fields:**
- `voxel_map: VoxelGrid`
- `masts: Vec<MastConfig>`
- `atmosphere: AtmosphericProfile`
- `min_coverage_depth: u32`

**Key methods:**

`fn analyze(&self) -> CoverageReport`

For each voxel in the grid, ray-marches from each mast. Computes: coverage fraction, per-voxel depth, shadow zone voxels, uncovered voxels.

`fn coverage_fraction(&self) -> f32`

Fraction of non-structure voxels observable by at least `min_coverage_depth` masts.

`fn trajectory_metrics(&self, trajectory: &[Point3<f32>]) -> TrajectoryMetrics`

Coverage continuity, gap statistics, and handoff opportunities along a trajectory.

`fn robustness_analysis(&self) -> RobustnessReport`

Runs analysis with each mast successively failed. Returns failure degradation curve.

`fn critical_sensor_set(&self) -> Vec<MastId>`

Returns masts whose removal reduces coverage fraction by more than a configurable threshold.

---

## 7. Module: `coverage_solver`

### `struct TopologyComparison`

`fn compare(scenario: &ScenarioConfig, topologies: &[CoverageTopology]) -> ComparisonReport`

Runs `CoverageAnalyzer` for each topology with the same voxel map and atmospheric conditions. Returns side-by-side metrics.

### `fn pareto_frontier(scenario: &ScenarioConfig, height_range: RangeInclusive<f32>, count_range: RangeInclusive<u32>) -> Vec<ParetoPoint>`

Sweeps the (H, N) parameter space. Returns the set of (H, N) pairs not dominated by any other on both coverage and cost.

### `fn solve_placement(region: &Region, budget: u32, voxel_map: &VoxelMap) -> Vec<MastDescriptor>`

**Algorithm:** Weighted Greedy Set Cover.

**Logic:**
1. Generate candidate positions (grid every 50 m).
2. Score each candidate by marginal coverage gain.
3. Greedy selection: pick highest marginal gain, update covered set, repeat until budget reached.
4. Interference check: no two mmWave radars within interference distance.

---

## 8. Module: `los_ray_marcher`

### `fn los_quality(from: Point3<f32>, to: Point3<f32>, voxel_map: &VoxelGrid, atmosphere: &AtmosphericProfile, wavelength_m: f32) -> f32`

Returns LOS quality in [0, 1]. Combines:
- `geometric_fraction`: unobstructed fraction via multi-ray marching with spatial hash acceleration
- `atmospheric_transmission`: Beer-Lambert double-pass for path length
- `target_visibility`: fraction of target cross-section visible at observation angle

**Algorithm (Ray Marching):**
1. Ray casting from `origin` to `target`.
2. Traversal using 3D DDA (Digital Differential Analyzer) for efficient voxel traversal.
3. For each Structure/Obstruction voxel: increment obstruction counter, track penetration depth.
4. `LOS_Quality = 1.0 - (penetration_depth / total_ray_length)` × atmospheric transmission.

### `fn los_quality_fast(from: Point3<f32>, to: Point3<f32>, voxel_map: &VoxelGrid) -> f32`

Single-ray approximation. Faster; suitable for coverage analysis where statistical accuracy over many voxels is acceptable.

---

## 9. Performance Requirements

| Operation | Target Latency | Optimization Strategy |
|---|---|---|
| Horizon Calculation | < 1 µs | Trivial |
| Shadow Cone Calculation | < 1 µs | Trivial |
| Beam Propagation (Range) | < 10 µs | Binary search |
| LOS Analysis (per pair) | < 100 µs | DDA + Spatial Hash |
| Full Coverage Report (1000 masts) | < 2 s | Rayon parallelism |

**Optimization Notes:**
- **Spatial Hashing:** O(1) lookup for obstructions near a ray.
- **Parallelism:** LOS checks for multiple masts run in parallel (Rayon).
- **Caching:** Cache LOS results for static geometry; recompute only for dynamic obstructions.

---

## 10. Interface (API)

```rust
impl CoverageAnalyzer {
    pub fn new(region: Region, atmosphere: Atmosphere) -> Self;
    pub fn add_candidate(&mut self, mast: MastDescriptor);
    pub fn analyze(&mut self) -> CoverageReport;
    pub fn get_optimal_placement(&self) -> Vec<MastDescriptor>;
}
```

---

## 11. Integration Points

- **`halo-simulator`:** Calls `analyze()` to generate scenario metrics.
- **`halo-perception`:** Uses `analyze_los()` to determine sensor activation.
- **`halo-atmospherics`:** Provides `Atmosphere` struct and extinction coefficients.

---

## 12. Testing

All functions are unit-tested against known analytical solutions documented in `docs/theory/coverage_geometry.md`.

Key test cases:
- `earth_curvature_horizon(2.0, 100.0)` ≈ 41.0 km ± 0.5 km
- `gaussian_beam_radius_full(0.05, 1000.0, 1.55e-6, 1.5)` ≈ computed Rayleigh result
- `power_density_at_range(1.0, 0.05)` = 127.3 W/m²
- Shadow cone coverage: point inside cone of mast A confirmed visible from mast B at known separation
