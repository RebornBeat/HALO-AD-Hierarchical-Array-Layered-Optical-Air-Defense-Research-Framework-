# Coverage Geometry — Engagement Volume, Mast Elevation, and Distributed Topologies

**Project:** HALO-AD
**Domain:** Geometric foundations of distributed-sensing coverage analysis

## 1. Purpose

This document specifies the geometric framework HALO-AD's simulator uses to compute engagement volumes, characterize coverage from distributed mast-mounted sensor architectures, and analyze the trade-offs between alternative topologies. It is the foundation document for the simulator; subsequent documents (`steering_latency_and_field_of_view.md`, `atmospheric_models.md`, `zone_handoff.md`) build on the framework established here.

The framework draws on three established research substrates: published radar-coverage geometry (the substantial literature on radar horizon, multipath, and ground-clutter analysis); published sensor-network coverage theory (the operations-research literature on geometric set cover, art-gallery problems, and visibility computation); and published cellular-network planning (the engineering literature on multi-base-station coverage, handoff geometry, and frequency reuse). HALO-AD's contribution is the synthesis of these substrates into a single open simulation framework parameterized for distributed mast-mounted architectures.

## 2. The Engagement Volume Concept

Define the *engagement volume* as the set of three-dimensional points within which a target is observable from at least one sensor in the deployed array. The simulator's core output is, for any deployment configuration, the engagement volume as a 3D voxel set, plus derived metrics:

- **Coverage fraction.** Volume of engagement set divided by volume of region of interest.
- **Coverage depth.** For each voxel in the engagement set, the number of sensors that can observe it.
- **Coverage continuity.** For trajectories crossing the region of interest, the fraction of the trajectory that lies in the engagement set.
- **Coverage robustness.** How the above metrics degrade as sensors fail, weather changes, or adversarial conditions develop.

Each of these is a different aspect of "how good is this deployment?" and the simulator produces all of them per scenario.

## 3. The Earth-Curvature Constraint

Ground-mounted sensors face a fundamental geometric limit: the Earth's curvature places a horizon between the sensor and any sufficiently distant low-altitude target. The horizon distance for a sensor at height *h_s* observing a target at altitude *h_t* (both measured from local ground level, both in meters, with the result in kilometers) is approximately:

```
d_horizon ≈ 3.57 · (√h_s + √h_t)
```

For a sensor at 2 meters and a target at 100 meters altitude, the horizon distance is roughly 41 km. A target below this altitude at greater range is geometrically unobservable to the sensor regardless of sensor performance.

The constraint has direct consequences for ground-based sensor deployment:

- **Roughly half the potential engagement volume is geometrically inaccessible** to a ground-mounted sensor — specifically, the lower-elevation portion of the volume below the horizon.
- **Low-altitude crossing trajectories** can transit the region while remaining beneath the horizon, observable only briefly when they pass very close to the sensor.
- **Multi-sensor ground arrays** can extend coverage horizontally but cannot extend coverage downward — the horizon constraint applies to each sensor independently.

This is the geometric problem that mast elevation addresses.

## 4. Mast Elevation as Coverage Geometry

Lifting a sensor to height *H_mast* above ground level shifts the horizon outward and downward. For a sensor at height *H_mast* + 2 meters (2 meters being the sensor's mount height above the mast top):

```
d_horizon ≈ 3.57 · (√(H_mast + 2) + √h_t)
```

A 30-meter mast extends the horizon for a 100-meter-altitude target from 41 km to roughly 56 km — a 37% increase. More importantly, the mast extends the horizon *downward*: targets at altitudes below what the ground-mounted sensor could see become geometrically observable.

The mast-elevation effect is *multiplicative* with sensor count. N ground-mounted sensors do not address the horizon constraint; N mast-mounted sensors collectively address it across N geographically-distributed mast locations, producing a coverage volume that approximates a continuous low-altitude coverage shell over the defended region.

### 4.1 The Mast Shadow Cone

Mast elevation creates a new geometric artifact: the *shadow cone* directly beneath the mast, where the sensor's down-look angle is geometrically limited by the mast structure or by the sensor's mechanical/electronic depression limit. For a sensor with maximum down-look angle *α* mounted at height *H_mast*:

```
shadow_radius = H_mast · tan(90° - α)
```

For a 30-meter mast with a sensor capable of 60° depression, the shadow radius is roughly 17 meters — a small dead zone immediately beneath the mast. For a sensor with only 30° depression, the shadow radius grows to roughly 52 meters.

The simulator computes the shadow cone for each mast and aggregates across the array. Single-mast shadow zones are blind spots; in a properly distributed array, neighboring masts' coverage volumes intersect each shadow zone, eliminating it.

### 4.2 Mast Height vs. Mast Count Trade-off

A single tall mast and many short masts both can achieve a given coverage volume, but with different cost-benefit profiles:

- **Single tall mast.** Lower hardware count, simpler coordination, larger single-point-of-failure risk, larger shadow cone, longer atmospheric-path effects (a tall mast has longer beam paths to low-altitude targets).
- **Many short masts.** Higher hardware count, more complex coordination, smaller shadow cones, redundancy against failure, shorter atmospheric paths.

The simulator parameterizes both dimensions and produces coverage-vs-cost curves for each scenario. The optimization is multi-objective; researchers studying a particular deployment scenario typically explore the Pareto frontier rather than identifying a single optimum.

## 5. Beam Divergence and Range-Power Trade-off

For active sensors (LiDAR, radar with directional emission), the beam diverges with range. The effective beam radius at range *z*:

```
w(z) = w_0 + z · θ
```

where *w_0* is the initial aperture radius and *θ* is the divergence half-angle. Power density on a target at range *z*:

```
power_density(z) = total_power / (π · w(z)²)
```

This relationship has direct consequences:

- **Sensitivity to range.** Power density decreases as the inverse square of effective beam radius. Doubling the range roughly halves power density (for *z* >> *w_0*).
- **Trade-off between range and resolution.** Tighter beams (smaller *θ*) maintain power density at range but cover smaller angular fields per beam, requiring more beams for the same total coverage.
- **Aperture as the design variable.** Larger initial aperture *w_0* reduces divergence proportionally; this is why long-range optical systems use telescope-class apertures.

The simulator parameterizes (*w_0*, *θ*) per sensor and computes effective sensitivity at each (sensor, target) pair.

### 5.1 Diffraction Limit

Even an ideal beam diverges due to diffraction. The diffraction-limited divergence half-angle for a Gaussian beam of wavelength λ and waist radius *w_0* is:

```
θ_diffraction = λ / (π · w_0)
```

For a 1.5-μm wavelength with a 10 cm aperture, the diffraction-limited divergence is roughly 4.8 microradians — about 1 cm spot growth per kilometer of range. Real systems do not achieve the diffraction limit; the simulator parameterizes a *beam quality factor* M² that scales the effective divergence above the diffraction limit.

## 6. Atmospheric Attenuation

The atmosphere attenuates beams through multiple mechanisms; `atmospheric_models.md` documents the simulator's parameterization. For coverage-geometry purposes, the relevant summary is that effective range is *not* the nominal range — it is the range at which the combination of beam divergence, atmospheric absorption, atmospheric scattering, and turbulence reduces detectable returns below threshold.

The simulator computes effective range per (sensor, target, atmospheric-condition) triple. Coverage-volume metrics use effective range, not nominal range. Coverage in degraded atmospheric conditions can be substantially smaller than nominal coverage; the simulator quantifies the difference.

## 7. Coverage Topologies

Three canonical topologies for distributing sensors across a region of interest:

### 7.1 Cone-to-Center

Multiple sensors arranged around a central point, each oriented inward toward the center. Coverage is concentrated near the center where all sensors' fields of view intersect; coverage at the periphery is sparse.

- **Ideal application.** Point defense of a single high-value asset within the cone.
- **Failure mode.** Targets passing outside the cone are not addressable.
- **Coverage character.** Deep at the center, shallow at the edges.

### 7.2 Parallel Array

Multiple sensors arranged along a line or curve, each oriented in the same direction (typically perpendicular to the array's axis). Coverage forms a band perpendicular to the array.

- **Ideal application.** Border or perimeter coverage along a defined boundary.
- **Failure mode.** Single-sensor failures produce a coverage gap; targets approaching obliquely receive only one sensor's coverage.
- **Coverage character.** Wide and shallow.

### 7.3 Distributed Mesh

Sensors distributed across the region with no specific orientation pattern, each contributing coverage to its local neighborhood. Coverage forms a continuous-or-near-continuous mesh across the region.

- **Ideal application.** General regional coverage where the threat distribution is unknown.
- **Failure mode.** Coverage is shallow everywhere — a high-value point cannot be uniquely well-defended.
- **Coverage character.** Uniform, with depth proportional to local sensor density.

### 7.4 Hierarchical Topology

Combination of the above: distributed mesh as a baseline, with cone-to-center clusters around high-value assets, and parallel arrays along boundaries. The hierarchical topology is the practical default for most realistic deployment scenarios.

The simulator supports each topology as a deployment pattern; researchers can combine them to study mixed scenarios.

## 8. Multi-Sensor Coverage Composition

When multiple sensors observe a single voxel, their contributions compose:

- **Independent observation.** Each sensor's detection is statistically independent (different modalities, different perspectives). Combined detection probability:

  ```
  P(detected) = 1 - Π(1 - P_i(detected))
  ```

- **Geometric diversity.** Sensors from different angles produce different perspectives on the same target. The resulting joint observation is often qualitatively richer than either single observation, allowing 3D position estimation, target-pose estimation, and occlusion-recovery that single sensors cannot.

- **Modal diversity.** Sensors of different modalities (LiDAR, radar, acoustic, event-vision) provide complementary information per `sensor_fusion.md`.

The simulator computes joint coverage metrics under independence assumptions (a conservative baseline) and under simulated modality-aware fusion (the realistic upper bound). Researchers studying particular fusion architectures can substitute their own fusion models.

## 9. Coverage Continuity Along Trajectories

For a target moving along a trajectory, the question is not whether each point along the trajectory is covered, but whether *the trajectory as a whole* is observable for purposes of tracking continuity. A trajectory that briefly enters and exits the engagement volume produces a track that briefly appears and disappears; depending on track-management policy, this may or may not be usable.

The simulator computes, per trajectory:

- **Coverage fraction.** Fraction of trajectory length within the engagement volume.
- **Continuous-coverage segment length.** Longest contiguous segment of the trajectory within the engagement volume.
- **Coverage gap statistics.** Distribution of gaps in coverage along the trajectory.
- **Track-handoff opportunities.** Points along the trajectory where coverage transitions from one sensor (or sensor cluster) to another, with time available for handoff.

These metrics are inputs to the zone-handoff analysis in `zone_handoff.md`.

## 10. Coverage Robustness

Real deployments degrade. Sensors fail, weather changes, adversarial conditions develop. The simulator computes coverage under degraded conditions:

- **Sensor failure.** Coverage with each sensor (in turn) removed from the array. Sensors whose removal substantially reduces coverage are *critical* and warrant redundancy.
- **Weather degradation.** Coverage under adverse atmospheric conditions (fog, dust, smoke). Modal diversity (radar vs. optical) provides robustness.
- **Adversarial conditions.** Coverage when adversarial obscurants, jamming, or counter-sensor effects are present in the scenario.
- **Geographic redistribution.** Coverage when subsets of the region are unavailable (e.g., due to local power loss, terrain change).

The robustness metrics complement the nominal-coverage metrics. A deployment with high nominal coverage but low robustness is fragile; a deployment with moderate nominal coverage but high robustness is resilient.

## 11. Composition with Other Theory Documents

Coverage geometry is one factor in overall deployment quality. The other theory documents address:

- `steering_latency_and_field_of_view.md` — Beam-steering architecture choice within each sensor (mechanical vs. solid-state vs. distributed-array).
- `atmospheric_models.md` — Detailed atmospheric-degradation parameterization that feeds the effective-range computation.
- `zone_handoff.md` — Multi-zone coordination logic that uses the coverage-continuity metrics computed here.
- `sensor_fusion.md` — Multi-modal fusion architecture that determines what "coverage" actually means in terms of detection probability and information richness.
- `future_research.md` — Adjacent research domains for which the coverage-geometry framework would provide upstream input.

The simulator integrates all of these into a single coverage-analysis pipeline; this document specifies the geometric foundation on which the integration rests.

## 12. Civilian Transfer

The coverage-geometry framework transfers directly to several civilian application domains:

- **Telescope-array siting.** Astronomical telescope arrays face the same multi-aperture coverage problem with similar geometric considerations (sky-coverage rather than ground-coverage, but the math is analogous).
- **Cellular base-station planning.** Multi-base-station coverage with horizon constraints (handled via published propagation models), shadow-cone analysis (urban canyon shadowing), and topology choice (hierarchical macro/micro/pico-cell deployments).
- **Sensor-network coverage planning.** Environmental monitoring, wildlife observation, traffic monitoring — each is a coverage problem with the same geometric framework.
- **Emergency-services radio-network planning.** Public-safety communications with similar coverage and redundancy requirements.
- **Optical-communication-network planning.** Free-space optical links with similar atmospheric-attenuation and beam-divergence considerations.

The civilian-transfer examples in the repository's `examples/civilian/` directory will demonstrate at least one transfer per topology family.

## 13. Summary

Coverage geometry is the foundation of HALO-AD's simulation framework. The Earth-curvature constraint motivates mast elevation; mast elevation requires distributed deployment to address shadow cones; distributed deployment requires multi-sensor fusion and zone handoff; the resulting architecture has direct civilian-transfer applications across telecommunications, scientific instrumentation, and environmental monitoring.

The simulator's core contribution is making this geometry computationally tractable for arbitrary scenarios, parameterized for realistic atmospheric and topological configurations, and producing comparable outputs across deployment alternatives.
