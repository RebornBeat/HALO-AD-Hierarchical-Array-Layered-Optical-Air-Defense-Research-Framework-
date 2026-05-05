# Geometry Engine Specification — HALO-AD

**Component:** `halo-geometry` crate
**Version:** 0.1.0
**Status:** Implementation Specification

---

## 1. Overview

The Geometry Engine is the computational core of HALO-AD. It is responsible for all spatial calculations related to sensor coverage, line-of-sight (LOS), atmospheric effects on beam propagation, and optimal placement/orientation of emitter masts.

While `docs/theory/coverage_geometry.md` establishes the physical laws and mathematical foundations, this document specifies the **implementation behavior**: data structures, algorithmic choices, input/output contracts, and performance requirements.

The engine consumes **scenario configuration** (mast positions, topologies, threat trajectories) and produces **coverage reports** (voxel observability, blind spots, quality metrics).

---

## 2. Core Data Structures

All structures are defined in `halo-core::geometry` and consumed by `halo-geometry`.

### 2.1 VoxelMap

The fundamental spatial representation. The 3D volume of interest is discretized into a grid of voxels (cubes).

```rust
pub struct VoxelMap {
    pub bounds: Bounds,             // Axis-aligned bounding box of the region
    pub resolution: f32,            // Edge length of each voxel (meters)
    pub data: Vec<VoxelState>,      // Flattened 3D array
}

pub enum VoxelState {
    Open,                           // Empty space
    Structure,                      // Part of ground/buildings (permanent)
    Obstruction,                    // Furniture, mobile objects
    Unknown,                        // Unmapped
}

pub struct Bounds {
    pub min: Vector3<f32>,
    pub max: Vector3<f32>,
}
```

**Implementation Note:**
- **Storage:** `Vec<VoxelState>` is memory-intensive for large volumes. For sparse environments (mostly air), consider a hierarchical representation (Octree) or spatial hashing in future optimizations.
- **Indexing:** `(x, y, z)` world coordinates map to `(i, j, k)` indices via `floor((coord - min) / resolution)`.

### 2.2 MastDescriptor

Describes a single emitter mast and its sensor suite.

```rust
pub struct MastDescriptor {
    pub id: String,
    pub position_m: Vector3<f32>,         // Base position
    pub height_m: f32,                     // Mast height
    pub sensors: Vec<SensorConfig>,
    pub depression_limit_deg: f32,         // Max down-look angle
}

pub struct SensorConfig {
    pub sensor_type: SensorType,
    pub orientation: Vector3<f32>,          // Aiming direction
    pub beam_width: f32,                    // For beam divergence calcs
}
```

### 2.3 CoverageReport

The primary output of the geometry engine.

```rust
pub struct CoverageReport {
    pub total_voxels: u64,
    pub observable_voxels: u64,
    pub coverage_fraction: f32,
    pub voxel_depth_histogram: HashMap<u8, u64>, // Key: count of observers, Value: voxel count
    pub blind_spots: Vec<VoxelCoord>,
    pub per_mast_coverage: HashMap<String, f32>, // Mast ID -> % of volume it sees
}
```

---

## 3. Algorithms

### 3.1 Horizon Calculation

**Function:** `calc_horizon_distance(sensor_h: f32, target_h: f32) -> f32`

Calculates the geometric horizon distance due to Earth's curvature.

**Formula:**
$$d \approx 3.57 \times (\sqrt{h_s} + \sqrt{h_t})$$
Where:
- $d$ = distance in km
- $h_s$ = sensor height in meters
- $h_t$ = target height in meters

**Implementation Logic:**
1. Check if `target_h` is below the minimum detectable altitude for `sensor_h`.
2. If `sensor_h` is elevated (mast), the horizon distance increases.
3. Return `0.0` if target is geometrically impossible to see (below horizon).

---

### 3.2 Shadow Cone Calculation

**Function:** `calc_shadow_cone(mast: &MastDescriptor) -> Cone`

Calculates the blind spot directly beneath a mast.

**Inputs:**
- `mast.height_m`
- `mast.depression_limit_deg`

**Geometry:**
The shadow is a cone emanating from the sensor position downwards.
- **Apex:** Mast position at `height_m`.
- **Angle:** `90° - depression_limit`.
- **Radius at ground:** `shadow_radius = height_m * tan(90° - depression_limit)`.

**Output:**
A `Cone` struct used for voxel exclusion:
```rust
pub struct Cone {
    pub origin: Vector3<f32>,
    pub direction: Vector3<f32>, // Typically (0, -1, 0)
    pub angle_rad: f32,
}
```

---

### 3.3 Beam Propagation & Atmospheric Attenuation

**Function:** `calc_effective_range(sensor: &SensorConfig, atmosphere: &Atmosphere) -> f32`

Calculates the distance at which a sensor's signal strength drops below a detection threshold.

**Steps:**

1.  **Beam Divergence ($w(z)$):**
    $$w(z) = w_0 + z \cdot \theta$$
    Where $\theta = \frac{M^2 \cdot \lambda}{\pi \cdot w_0}$.

2.  **Power Density ($I(z)$):**
    $$I(z) = \frac{P_{total}}{\pi \cdot w(z)^2}$$

3.  **Atmospheric Transmission ($T(z)$):**
    $$T(z) = e^{-\alpha \cdot z}$$ (Beer-Lambert Law).
    The extinction coefficient $\alpha$ is retrieved from `Atmosphere` profile based on wavelength.

4.  **Effective Range Calculation:**
    Iterate $z$ until:
    $$I(z) \cdot T(z)^2 < \text{threshold}$$
    (The square on $T(z)$ accounts for two-way path for active sensors).

**Implementation Note:** Use binary search or Newton's method for efficient range finding, rather than linear stepping.

---

### 3.4 Line-of-Sight (LOS) Analysis

**Function:** `analyze_los(origin: Vector3<f32>, target: Vector3<f32>, voxel_map: &VoxelMap) -> f32`

Determines if a target is visible from a sensor position and returns a **quality score** ($0.0$ to $1.0$), not just a boolean.

**Algorithm (Ray Marching):**
1.  **Ray Casting:** Define a ray from `origin` to `target`.
2.  **Traversal:** Step along the ray through the `VoxelMap`. Use a 3D DDA (Digital Differential Analyzer) algorithm for efficient voxel traversal.
3.  **Intersection Check:**
    - If voxel state is `Structure` or `Obstruction`:
        - Increment an obstruction counter.
        - Track the penetration depth (how many obstructed voxels are crossed).
    - If voxel state is `Open`:
        - Continue.
4.  **Quality Calculation:**
    - `LOS_Quality = 1.0 - (penetration_depth / total_ray_length)`
    - **Refinement:** Account for partial occlusions. If the ray clips the corner of an obstruction, `LOS_Quality` might be $0.7$ rather than $0.0$.
    - **Atmospheric Degration:** Multiply result by atmospheric transmission $T(z)$.

---

### 3.5 Coverage Solver (Placement Optimizer)

**Function:** `solve_placement(region: &Region, budget: u32, voxel_map: &VoxelMap) -> Vec<MastDescriptor>`

**Objective:** Find the minimum set of mast positions to observe all high-priority voxels.

**Algorithm:** Weighted Greedy Set Cover.

**Inputs:**
- `Region`: Defined bounds and priority weights per voxel.
- `budget`: Maximum number of masts allowed.
- `VoxelMap`: Geometry of the space.

**Logic:**
1.  **Candidate Generation:** Generate a grid of candidate positions (e.g., every 50m).
2.  **Initial Scoring:** For each candidate, calculate the marginal coverage gain (how many uncovered high-priority voxels it would see).
3.  **Greedy Selection:**
    - Pick the candidate with the highest marginal gain.
    - Add to `SolutionSet`.
    - Update `CoveredSet`.
    - Repeat until `CoveredSet` covers all priority voxels OR `budget` reached.
4.  **Interference Check:** Ensure no two mmWave radars are within interference distance (parameterized).
5.  **Output:** List of `MastDescriptor` objects.

---

## 4. Performance Requirements

| Operation | Target Latency | Note |
| :--- | :--- | :--- |
| Horizon Calculation | < 1 µs | Trivial calculation. |
| Shadow Cone Calculation | < 1 µs | Trivial calculation. |
| Beam Propagation (Range) | < 10 µs | Iterative search required. |
| LOS Analysis (per pair) | < 100 µs | Ray marching can be expensive. Must be optimized (DDA/Spatial Hash). |
| Full Coverage Report (1000 masts) | < 2 s | Parallelize per-mast analysis. |

**Optimization Strategies:**
- **Spatial Hashing:** For O(1) lookup of obstructions near a ray.
- **Parallelism:** LOS checks for multiple masts can run in parallel (Rayon crate).
- **Caching:** Cache LOS results for static geometry; only recompute for dynamic obstructions.

---

## 5. Interface (API)

The crate exposes a high-level `CoverageAnalyzer` struct.

```rust
impl CoverageAnalyzer {
    /// Creates a new analyzer for a given region and atmospheric model.
    pub fn new(region: Region, atmosphere: Atmosphere) -> Self;

    /// Adds a candidate mast position.
    pub fn add_candidate(&mut self, mast: MastDescriptor);

    /// Runs the full analysis pipeline.
    pub fn analyze(&mut self) -> CoverageReport;

    /// Returns the best placement configuration.
    pub fn get_optimal_placement(&self) -> Vec<MastDescriptor>;
}
```

---

## 6. Integration Points

- **`halo-simulator`:** Calls `analyze()` to generate scenario metrics.
- **`halo-perception`:** Uses `analyze_los()` to determine sensor activation.
- **`halo-atmospherics`:** Provides the `Atmosphere` struct and extinction coefficients.
