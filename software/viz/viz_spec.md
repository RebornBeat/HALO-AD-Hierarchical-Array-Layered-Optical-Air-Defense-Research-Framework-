# Visualization Engine Specification — `halo-viz` Crate

**Component:** `halo-viz` (implemented in `crates/halo-viz`)
**Role:** Transforms simulation outputs into human-readable visual reports
**Version:** 1.0

---

## 1. Purpose

The `viz` module bridges raw simulation data (voxel grids, CSV logs) and actionable insight. It produces graphical outputs that allow researchers to:

1. **Verify Geometry:** Visually confirm mast placements produce intended coverage.
2. **Analyze Performance:** See where tracks were lost, handoffs failed, and blind spots exist.
3. **Compare Topologies:** Visually distinguish `ConeToCenter`, `ParallelArray`, and other deployments.
4. **Debug Scenarios:** Inspect threat trajectories and sensor fields of view in 3D.

All outputs are static files (HTML, SVG, PNG) or embedded data in self-contained HTML reports — no real-time rendering, no GPU dependency. Suitable for research papers, presentations, and reproducible comparison.

**Dependencies:** `halo-core`, `halo-geometry`, `halo-simulator`, `plotters` (2D charts), `serde_json` (embedding data in HTML), `base64` (embedding images).

---

## 2. Inputs

### 2.1 Geometry Data (`halo-core`, `halo-geometry`)

- **`VoxelGrid`**: 3D occupancy map (Open, Structure, Obstruction).
- **`CoverageMap`**: Per-voxel observability data (`observable_count`, `los_quality`, `atmospheric_visibility`).
- **`MastDescriptors`**: Positions, heights, sensor types, depression limits.
- **`ShadowCones`**: Computed blind-spot volumes under each mast.
- **`ZoneBoundaries`**: 3D polygons defining districts and zones.

### 2.2 Simulation Data (`halo-simulator`)

- **`ThreatTrajectories`**: True paths of simulated threats.
- **`TrackLog`**: Estimated positions and states from the perception pipeline.
- **`HandoffLog`**: Zone handoff events (successful and failed).
- **`CoverageMetrics`**: Time-series of coverage fraction, depth, etc.

---

## 3. Output Classes

### 3.1 Static Images (PNG/SVG)

For embedding in PDFs, slide decks, and papers:
- 2D slice views (horizontal/vertical cross-sections).
- Topology diagrams.

### 3.2 Interactive Reports (Self-Contained HTML)

- No external server required; open in a browser.
- 3D scene exploration.
- Time-series playback (trajectory animation).
- Topology comparison dashboards.

---

## 4. Visualization Types

### 4.1 3D Coverage Density Heatmap

**Visual:** 3D volumetric render where voxels are colored by observability count.
- **Green:** High coverage (3+ masts).
- **Yellow:** Moderate coverage (1–2 masts).
- **Red:** Observable but low quality (marginal LOS).
- **Transparent:** Unobservable.

**Controls:** Opacity slider, slice plane (X/Y/Z), toggle mast positions, shadow cones, zone boundaries.

**Implementation:** `CoverageHeatmapRenderer` produces series of 2D slice images (one per altitude band) assembled into HTML with JavaScript altitude slider. Color scheme: Okabe-Ito palette (colorblind-friendly).

```rust
fn render(coverage_report: &CoverageReport, output_path: &Path) -> Result<(), VizError>
```

### 4.2 Trajectory Overlay

**Visual:** 3D plot showing threat paths and tracking performance.
- **Solid line:** True trajectory.
- **Dashed line:** Estimated track.
- **Circles:** Handoff event points (color: success/fail).
- **X:** Track loss.
- **Colored segments:** Trajectory colored by coverage volume membership.

```rust
fn render(coverage_report: &CoverageReport, track_log: &TrackLog, output_path: &Path) -> Result<(), VizError>
```

### 4.3 Topology Comparison View

**Visual:** Side-by-side 3D view for A/B testing.
- Left Pane: Topology A.
- Right Pane: Topology B.
- Delta View: Difference map highlighting gained/lost coverage.

### 4.4 Shadow Cone Visualization

Renders blind spots beneath each mast as semi-transparent cones.

**Purpose:** Verify that neighboring masts' coverage volumes intersect each other's shadow cones.

### 4.5 Zone Partitioning Render

Overlays defined zone boundaries onto coverage heatmap with zone IDs and assigned masts.

---

## 5. Technology Stack

- **Rust:** `serde` for JSON serialization, `nalgebra` for geometry transformations, `plotters` for 2D/3D SVG charts.
- **HTML/JS (embedded):** Three.js for 3D WebGL rendering, Plotly.js for scientific plots, D3.js for topology diagrams.

---

## 6. API

```rust
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

## 7. Report Generator

`fn generate_html_report(run_dir: &Path, output_path: &Path) -> Result<(), VizError>`

Assembles all run outputs into a single self-contained HTML report containing:
- Scenario configuration summary
- Key metrics table (coverage fraction, TTFT, handoff success rate)
- Interactive coverage heatmap
- Trajectory overlay per threat
- Atmospheric condition impact table (nominal vs. degraded coverage)
- If comparison run: side-by-side topology comparison with bar charts

The report is fully self-contained (all assets embedded as base64 or inline SVG) for distribution as a single file.

`fn render_topology_comparison(comparison_report: &ComparisonReport, output_path: &Path) -> Result<(), VizError>`

Grouped bar chart comparing topology variants on: coverage fraction, mean coverage depth, time-to-first-track P50/P90, handoff success rate.

---

## 8. File Output Structure

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

## 9. Integration with Reports

`halo-reports` embeds static images into the main HTML report, while linking to interactive HTML files for deeper analysis.

## 10. Alignment Note

Visualization logic lives in `crates/halo-viz`. The `software/viz/` directory holds only this spec and supplementary asset files (colormap definitions).
