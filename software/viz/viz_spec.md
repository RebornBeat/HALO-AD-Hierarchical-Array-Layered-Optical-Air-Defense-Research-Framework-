# Visualization Specification — HALO-AD

**Component:** `software/viz` (implemented by `crates/halo-viz`)
**Role:** Transforms simulation outputs into human-readable visual reports (3D renders, heatmaps, trajectory plots).

---

## 1. Purpose

The `viz` module bridges the gap between raw simulation data (voxel grids, CSV logs) and actionable insight. It produces the graphical outputs that allow researchers to:

1.  **Verify Geometry:** Visually confirm that mast placements produce the intended coverage.
2.  **Analyze Performance:** See where tracks were lost, where handoffs failed, and where blind spots exist.
3.  **Compare Topologies:** Visually distinguish between `ConeToCenter`, `ParallelArray`, and other deployments.
4.  **Debug Scenarios:** Inspect threat trajectories and sensor fields of view in 3D.

The primary implementation lives in `crates/halo-viz`. This document specifies the inputs, outputs, and visualization types.

---

## 2. Inputs

The visualization module consumes the output products of the simulator and geometry engine:

### 2.1 Geometry Data (`halo-core`, `halo-geometry`)

- **`VoxelGrid`**: The 3D occupancy map of the region (Open, Structure, Obstruction).
- **`CoverageMap`**: Per-voxel observability data:
    - `observable_count`: Number of masts with Line-of-Sight (LOS) to this voxel.
    - `los_quality`: Average LOS quality score (0.0–1.0).
    - `atmospheric_visibility`: Atmospheric transmission factor.
- **`MastDescriptors`**: Positions, heights, sensor types, depression limits.
- **`ShadowCones`**: Computed blind-spot volumes under each mast.
- **`ZoneBoundaries`**: 3D polygons defining districts and zones.

### 2.2 Simulation Data (`halo-simulator`)

- **`ThreatTrajectories`**: True paths of simulated threats.
- **`TrackLog`**: Estimated positions and states from the perception pipeline.
- **`HandoffLog`**: Zone handoff events (successful and failed).
- **`CoverageMetrics`**: Time-series of coverage fraction, depth, etc.

---

## 3. Outputs

The module produces two classes of output:

### 3.1 Static Images (Raster/Vector)

- **PNG/SVG** for embedding in PDFs, slide decks, and papers.
- Used for:
    - 2D slice views (horizontal/vertical cross-sections).
    - Topology diagrams.

### 3.2 Interactive Reports (HTML/JS)

- **Self-contained HTML files** embedding all data and JS logic.
- No external server required; open in a browser.
- Used for:
    - 3D scene exploration.
    - Time-series playback (trajectory animation).
    - Topology comparison dashboards.

---

## 4. Visualization Types

### 4.1 3D Coverage Density Heatmap

**Visual:**
A 3D volumetric render where voxels are colored by observability count.

- **Green:** High coverage (3+ masts).
- **Yellow:** Moderate coverage (1-2 masts).
- **Red:** Observable but low quality (marginal LOS).
- **Transparent:** Unobservable (engagement volume void).

**Controls:**
- Opacity slider.
- Slice plane (X, Y, or Z).
- Toggle visibility of:
    - Mast positions.
    - Shadow cones.
    - Zone boundaries.

### 4.2 Trajectory Overlay

**Visual:**
A 3D plot showing threat paths and tracking performance.

- **Solid line:** True trajectory.
- **Dashed line:** Estimated track.
- **Circles:** Handoff event points (color: success/fail).
- **X:** Track loss.

**Purpose:**
To visually correlate tracking errors with coverage gaps.

### 4.3 Topology Comparison View

**Visual:**
A side-by-side 3D view for A/B testing.

- **Left Pane:** Topology A (e.g., `GroundStatic`).
- **Right Pane:** Topology B (e.g., `MastElevated`).
- **Delta View:** Difference map highlighting gained/lost coverage.

### 4.4 Shadow Cone Visualization

**Visual:**
Renders the blind spot beneath each mast as a semi-transparent cone.

- **Purpose:** To verify that neighboring masts' coverage volumes intersect the shadow cones of their neighbors, ensuring no unobserved voids.

### 4.5 Zone Partitioning Render

**Visual:**
Overlays the defined zone boundaries onto the coverage heatmap.

- **Lines:** Zone borders.
- **Labels:** Zone IDs and assigned masts.
- **Purpose:** To debug handoff logic and zoning config.

---

## 5. Technology Stack

The implementation (`crates/halo-viz`) uses Rust to generate the data structures and JSON outputs, which are then rendered via embedded JavaScript libraries.

- **Rust:**
    - `serde` for JSON serialization.
    - `nalgebra` for geometry transformations.
- **HTML/JS:**
    - **Three.js** for 3D WebGL rendering.
    - **Plotly.js** for 2D/3D scientific plots.
    - **D3.js** for topology diagrams.

---

## 6. API (Internal)

The visualization module is typically called by `halo-cli` at the end of a run.

```rust
// Implementation in crates/halo-viz/src/lib.rs

pub struct VizConfig {
    pub output_path: PathBuf,
    pub generate_interactive_html: bool,
    pub generate_static_images: bool,
}

pub fn generate_visualization(
    scenario: &ScenarioConfig,
    coverage_map: &CoverageMap,
    track_log: &[TrackRecord],
    config: VizConfig,
) -> Result<(), VizError>;
```

---

## 7. File Output Structure

```
runs/<scenario_name>_<timestamp>/
├── coverage_heatmap.html
├── topology_comparison.html
├── static/
│   ├── slice_xy_500m.png
│   ├── slice_xz_100m.png
│   └── topology_diagram.svg
└── data/
    ├── coverage_voxels.json
    └── trajectories.json
```

---

## 8. Integration with Reports

The `halo-reports` crate embeds the static images into the main HTML report, while linking to the interactive HTML files for deeper analysis.

---

## 9. Alignment Note

This spec describes the **visualization logic**.
The actual code lives in `crates/halo-viz`.
The `software/viz/` directory exists only to hold this spec and any supplementary asset files (e.g. colormap definitions).
