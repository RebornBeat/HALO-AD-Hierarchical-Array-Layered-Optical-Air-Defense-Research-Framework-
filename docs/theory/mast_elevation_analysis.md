# Mast Elevation Analysis — System-Level Trade-offs for Distributed Coverage Architectures

**Project:** HALO-AD
**Domain:** System architecture, deployment topology, mast-height vs. mast-count optimization,
shadow-cone analysis, atmospheric-path implications
**Mathematical Foundation:** `docs/theory/coverage_geometry.md`
**Implementation:** `halo-geometry::horizon`, `halo-geometry::shadow_cone`, `halo-geometry::coverage_solver`

---

## 1. Purpose

This document characterizes the **system-level trade-offs** of mast elevation in distributed coverage architectures. While `coverage_geometry.md` provides the mathematical foundation (horizon distance, shadow cone geometry, beam propagation physics), this document addresses the *engineering decision layer* above the physics: given a fixed budget of masts and sensors, how should height and count be traded to maximize coverage quality?

The central thesis: **Mast elevation is the single most impactful geometric parameter in ground-based distributed sensing, but its benefits are nonlinear and context-dependent.** There is no universally optimal height. The right choice emerges from a multi-objective optimization over coverage, cost, resilience, and operational constraints specific to each deployment scenario.

The analysis applies identically to any mast-mounted sensing or emitting system. The civilian transfer to telescope-array siting, cellular base-station planning, and environmental-monitoring networks is direct.

---

## 2. The Fundamental Trade-off Triangle

Mast elevation interacts with three primary variables:

1. **Mast Height (H)**
2. **Mast Count (N)**
3. **Coverage Depth (D)** — number of independent observers per voxel

You cannot maximize all three simultaneously under a fixed budget. The simulator explicitly models the Pareto frontier of these trade-offs for any scenario.

```
         Coverage Depth (D)
              /\
             /  \
            /    \
           /  Trade-off
          /      Zone
         /________\
Mast Height (H)  Mast Count (N)
```

**The core insight:** A tall mast (high H) can replace multiple short masts (low N) for *area coverage*, but at the cost of:
- **Reduced Coverage Depth (D):** A single mast provides only one perspective.
- **Reduced Resilience:** Failure of one tall mast creates a massive coverage hole.
- **Increased Cost:** Civil engineering, power, and maintenance scale superlinearly with height.

These variables are not independent. For a given coverage fraction target over a given region, there is a continuous trade-off surface between H and N. The simulator maps this surface rather than selecting a point on it.

---

## 3. The Horizon Constraint: The Primary Driver

The physics are unambiguous: **Earth curvature imposes a hard lower bound on the observable volume.**

A ground-mounted sensor cannot see below the horizon. A target at altitude $h_t$ is geometrically unobservable beyond distance $d_{horizon}$:

$$d_{horizon} \approx 3.57 \times (\sqrt{h_s} + \sqrt{h_t}) \text{ km}$$

where $h_s$ is the sensor height in meters.

**Implication:** For low-altitude threats ($h_t \approx 50\text{--}100\text{ m}$), a ground-mounted sensor ($h_s = 2\text{ m}$) is blind beyond ~40 km. A 30 m mast extends this to ~56 km — a **37% increase in radius**, translating to a **~85% increase in area coverage** for the same sensor. This is the primary motivator for mast elevation.

---

## 4. Single Tall Mast vs. Distributed Short Masts

The extreme cases bound the trade-off space.

### 4.1 Single Tall Mast

One mast at height H_single. To achieve a target horizon distance $d_h$ for a given target altitude $h_t$:

$$H_{single} = \left(\frac{d_h}{3.57} - \sqrt{h_t}\right)^2$$

For $h_t = 50\text{ m}$, $d_h = 50\text{ km}$: $H_{single} \approx 170\text{ m}$ — broadcast-tower scale.

**Properties:**
- Hardware count = 1. Minimum coordination complexity; no inter-mast synchronization.
- **Single point of failure.** If the mast fails, all coverage is lost.
- **Large shadow cone.** At H = 170 m with α = 45°, shadow radius = 170 m. No neighboring mast covers this blind zone.
- Long atmospheric path to low-altitude nearby targets: a target at $h_t = 10\text{ m}$ and range 1 km is observed on a path with 160 m altitude change — maximizing atmospheric volume traversed per unit of horizontal range.
- **No coverage depth.** Every voxel is seen by exactly one sensor; any single-point failure drops a detection entirely.

### 4.2 Many Short Masts

N masts each at height $H_{short} = H_{single} / K$ for some factor $K > 1$.

For the same horizon at the same target altitude, the horizon equation gives:

$$d_{horizon}(\text{N masts at } H_{short}) = \max_{i} \left\{ 3.57 \times (\sqrt{H_{short}} + \sqrt{h_t}) \right\}$$

Each individual mast has shorter horizon than the single tall mast. The distributed array compensates with spatial distribution: neighboring masts together cover corridors that neither covers alone.

**Properties:**
- Hardware count = N. Higher coordination complexity; inter-mast synchronization required.
- **Distributed failure modes.** Loss of one mast degrades coverage but does not eliminate it.
- **Smaller individual shadow cones** at different locations — neighboring masts cover each other's shadow zones.
- Shorter atmospheric paths to nearby targets.
- **Coverage depth ≥ 2** in overlap regions — improves detection probability and enables 3D localization.

---

## 5. Cost Engineering: The Nonlinear Reality

The cost of a mast is not linear with height. It scales with:

### 5.1 Civil Engineering
- Foundation size grows with wind load moment.
- Guy wires or monopole thickness.
- Permitting complexity often increases with height.

### 5.2 Infrastructure
- Power cabling length, fiber/copper data runs, lightning protection.

### 5.3 Maintenance Access
- Climbing safety equipment, crane or lift requirements, weather exposure.

**Cost Heuristic:**
$$Cost(H) \propto H^\beta, \quad \text{where } \beta > 1$$

The simulator allows user-defined cost models. The default is an exponential model.

---

## 6. The Pareto Frontier

`halo-geometry::coverage_solver::TopologyComparison` computes the full Pareto frontier between coverage quality and cost for a given scenario. The frontier is parameterized by (H, N) pairs.

Typical Pareto frontier shape for a 10 km × 10 km region defending against $h_t = 50\text{--}200\text{ m}$ altitude targets:

```
Coverage     │         ● Single 80m mast + 4 short masts
Fraction     │      ●
(%)      85  │   ●
         80  │ ●
         75  ●
         70  │
             └─────────────────────────────
             1    2    4    6    8    10    N (mast count)
             (paired with decreasing H as N increases)
```

The frontier is concave — marginal coverage gain from adding a mast decreases as the array grows. Research using HALO-AD typically identifies the **"knee point"** of the Pareto frontier as the recommended deployment: the configuration at which marginal coverage gain equals marginal cost per added mast.

The simulator produces this curve as part of `halo-cli compare` output when the `--pareto` flag is specified.

---

## 7. Resilience: The Single Point of Failure

**Failure Modes:** Power failure, structural damage (wind, impact), sensor failure.

**Impact of single tall mast failure:** Loss of entire coverage volume for that sector — potentially catastrophic.

**Distributed short masts:** Failure of one mast reduces coverage depth (D) by 1. Neighbors maintain track continuity. Graceful degradation, not catastrophic loss.

**Resilience Metric:**
$$Resilience = 1 - \frac{\text{Coverage Lost on Failure of One Mast}}{\text{Total Coverage}}$$

For N distributed short masts with symmetric coverage:

$$coverage(k \text{ failed masts}) \approx coverage_{nominal} \times (1 - k/N)^\beta$$

where β depends on coverage overlap. For heavily overlapping arrays (β ≈ 1), degradation is linear. For barely-overlapping arrays (β > 1), degradation is superlinear.

The simulator computes this curve empirically via `robustness_analysis()`, running the coverage analyzer with each mast successively removed.

---

## 8. Shadow-Cone Interaction Analysis

The shadow cone beneath each mast is the most distinctive geometric artifact of mast-elevated deployment.

### 8.1 Single-Mast Shadow Zone Geometry

For a mast at position $(x_0, y_0)$ with height $H$ and maximum depression angle $\alpha$ (measured from horizontal):

$$r_{shadow} = H \times \tan(90° - \alpha)$$
$$V_{shadow} = \frac{1}{3} \times \pi \times r_{shadow}^2 \times H$$

**Depression angle sensitivity:**

| Depression limit α | H = 15 m shadow radius | H = 30 m shadow radius | H = 60 m shadow radius |
|---|---|---|---|
| 80° | 2.6 m | 5.3 m | 10.6 m |
| 60° | 8.7 m | 17.3 m | 34.6 m |
| 45° | 15.0 m | 30.0 m | 60.0 m |
| 30° | 26.0 m | 51.9 m | 103.9 m |

A sensor with 80° depression has a negligible shadow cone. A sensor limited to 30° depression has a large shadow zone requiring neighbor coverage.

### 8.2 Shadow Zone Coverage by Neighbors

For a shadow zone of radius $r_{shadow}$ centered at mast $m_0$ to be covered by neighboring masts:

$$\forall p \in shadow\_cone(m_0): \exists \text{ neighbor } m_n : LOS\_quality(m_n, p) > threshold$$

Minimum mast spacing that guarantees shadow zone coverage:

$$spacing \leq LOS\_range(m_n, h_{target}) + r_{shadow}$$

The coverage solver automatically checks this constraint. Configurations with uncovered shadow zones are flagged as `UncoveredShadowZone` events in the placement report.

### 8.3 Shadow-Zone Coverage Strategies

**Neighbor height compensation:** Neighboring masts aim inward toward each other's shadow zones. The coverage solver accepts `shadow_zone_coverage_priority` parameter that up-weights shadow-zone voxels.

**Overlapping array geometry:** Masts placed such that each shadow cone falls within the coverage cone of at least two neighbors. For hexagonal arrays, shadow zone coverage is achieved with high redundancy.

**Sub-mast sensors:** Supplementary sensor at height 0–2 m co-located with the mast, oriented upward. Represented in the simulator as a second sensor configuration entry.

---

## 9. Atmospheric Path Length Implications

### 9.1 Path Length Formula

For a mast at height $H$ observing a target at altitude $h_t$ and horizontal range $r$:

$$path\_length = \sqrt{r^2 + (H - h_t)^2}$$

For a ground-level sensor ($H \approx 0$):

$$path\_length_{ground} = \sqrt{r^2 + h_t^2}$$

### 9.2 Near-Field vs. Far-Field Tradeoff

**Near field ($r < H$):** Steep downward path. Atmospheric path is longer than ground-level for the same horizontal range.

**Far field ($r >> H$):** Both sensors observe on nearly horizontal paths. Path lengths converge. Atmospheric difference is negligible.

**Crossover range:**

$$r_{crossover} \approx H \times h_t / \sqrt{H^2 - h_t^2} \quad (h_t < H)$$

At all practically relevant ranges where the mast matters, the coverage advantage dominates.

### 9.3 Atmospheric Interaction: The Double-Edged Sword

**Advantages of elevation:**
- Avoids ground fog and low-level aerosols.
- Reduces ground clutter (radar reflections from terrain).
- Clears obstacles (trees, buildings) for line-of-sight.

**Disadvantages:**
- Longer beam path: more thermal blooming and turbulence accumulation.
- Higher wind speeds: increased structural stress and beam jitter.

`halo-atmospherics::effective_range::effective_range_degraded()` accounts for actual 3D path length, not simplified horizontal-range approximation, for every (mast, voxel) pair.

---

## 10. Integration with Topologies

### 10.1 Cone-to-Center
- **Tall Masts:** Elevated focal volume; covers more ground area but potential hole at low altitudes near center.
- **Short Masts:** Lower focal point; better for low-altitude point defense.

### 10.2 Parallel-Array
- **Tall Masts:** Longer unobstructed parallel paths; better for perimeter coverage.
- **Short Masts:** More likely to have ground-level occlusions.

### 10.3 Divergent-Fan
- Both inner cone and outer fan are lifted with taller masts.
- The inner/outer transition angle can be tuned to match the mast height.

---

## 11. Deployment Speed: The Operational Variable

- **Short Masts:** Rapid deployment. Truck-mounted, trailer-mounted, or rapidly erected. Hours to deploy.
- **Tall Masts:** Significant civil engineering. Days to weeks for installation.

For rapidly relocatable systems (forward operating base, temporary event security), tall fixed infrastructure is not viable. The trade-off shifts strongly toward **distributed short masts**.

---

## 12. The Mast Height Optimization

Given a fixed budget (cost ∝ H × N), what (H, N) pair maximizes coverage fraction?

### 12.1 Single-Zone Simple Case

For a single circular zone of radius R against targets at altitude $h_t$, the optimal mast height given N masts arranged in a ring:

$$H_{optimal} \approx \frac{(R/N - \sqrt{h_t} \times 3.57)^2}{3.57^2} \quad \text{when } R/N > \sqrt{h_t} \times 3.57$$

This derives from setting each mast's horizon exactly to the next mast's position — no gaps, no redundancy. The simulator finds the actual optimum including redundancy preferences.

### 12.2 Multi-Zone General Case

For multi-zone scenarios, optimal (H, N) allocation varies per zone. High-priority zones benefit from taller masts (more depth per unit cost); perimeter zones benefit from more short masts (breadth).

The coverage solver's `minimize_cost_for_coverage_target()` solves this as a mixed-integer optimization over the Pareto space for each zone, then combines allocations.

---

## 13. Summary Trade-off Table

| Configuration | Coverage | Depth | Shadow | Failure Robustness | Cost |
|---|---|---|---|---|---|
| 1 very tall mast | Moderate (horizon limited) | 1 | Large, uncovered | None (single point) | Low hardware, high structure |
| 2 tall masts | Better | 2 in overlap | Partially covered | 1 failure → moderate | Moderate |
| 4 medium masts | Good | 2–4 in core | Covered by neighbors | Graceful | Higher |
| 8 short masts | Good-excellent | 3–6 in core | Fully covered | Robust | High hardware, low structure |
| N adaptive (solver output) | Optimal for scenario | Configurable | Configurable | Configurable | Pareto-optimal |

---

## 14. Research Framework

The simulator does not dictate a "correct" height. It answers:

1. **For a given budget, what is the Pareto-optimal mix of height and count?**
2. **What is the resilience of a tall-mast vs. distributed architecture?**
3. **What is the coverage cost-per-voxel for different mast heights?**

The `halo-cli compare` tool allows sweeping mast height as a variable and generating a **Height vs. Coverage** curve for any scenario.

---

## 15. Civilian Transfer

The mast-height vs. mast-count trade-off appears identically in:

- **Cellular Networks:** Macro-cell towers vs. small cells. Same Pareto analysis applies.
- **Radio/TV Broadcast:** Tall tower vs. distributed relay.
- **Wind Turbines:** Hub height vs. number of turbines.
- **Environmental Monitoring:** Air-quality or weather sensors on poles.
- **Emergency-Services Communications:** Repeater towers for rural coverage (APCO-25 planning).
- **Astronomical Telescope Arrays:** Effective collecting aperture and array geometry produce identical coverage-vs-count Pareto structure.

The `halo-geometry::coverage_solver` is agnostic to the application. Cost functions are user-defined.
