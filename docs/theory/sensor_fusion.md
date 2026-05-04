# Sensor Fusion — Multi-Modal Long-Range Outdoor Perception

**Project:** HALO-AD
**Domain:** Multi-modal sensor fusion at engagement timescales for distributed outdoor coverage

## 1. Purpose

This document characterizes HALO-AD's approach to multi-modal sensor fusion in the long-range outdoor coverage context. It is structurally similar to AEGIS-MESH's `sensor_fusion.md` (which covers residential indoor sensing) but is parameterized for the longer-range, outdoor-atmospheric, fast-target-velocity regime that HALO-AD addresses.

The fusion architecture combines event-based vision, mmWave Doppler radar, and scanning LiDAR. Each modality contributes distinct information; their fusion produces perception capabilities that no single modality achieves alone. This document specifies what each modality contributes, how the fusion is structured, and how it composes with PentaTrack's predictive-tracking layer.

## 2. The Three Primary Modalities

### 2.1 Event-Based Vision (Neuromorphic)

Event-based image sensors emit asynchronous events when individual pixels detect illumination changes above a threshold. Reference parts include sensors from Prophesee and iniVation, with a published academic literature spanning roughly 15 years.

For long-range outdoor coverage:

- **Latency.** Microsecond-scale per event, vastly faster than frame-based imaging.
- **Sparse output.** Static-scene background produces few events; moving targets produce a temporally compact stream of events along their apparent trajectory.
- **High effective frame rate.** Equivalent to 10⁵+ fps for fast-moving objects, without the bandwidth and processing burden of true high-rate frame-based imaging.
- **Limitations.** Requires illumination contrast; sensor-physics constraints on dynamic range; current commercial sensors have lower resolution than premium frame-based imagers (though this gap is closing).

In HALO-AD's role, event-based vision excels at *fast-target detection* — the initial detection of a target appearing in the field. The microsecond latency means the simulator can model detection-to-track-initiation as essentially instantaneous, in contrast to the 50-100 ms typical for scanning LiDAR.

### 2.2 mmWave Doppler Radar

Continuous-wave or frequency-modulated continuous-wave radar at 24, 60, 77, or 94 GHz emits radio-frequency signal and observes the Doppler shift of returns. Reference parts include Texas Instruments IWR-series and Infineon BGT-series chipsets, with published academic and industrial literature on the underlying physics, the chipset architectures, and the application domains.

For long-range outdoor coverage:

- **Direct velocity readout.** Doppler shift gives radial velocity in a single measurement, with no frame-to-frame differentiation.
- **Through-obscuration robustness.** Operates through fog, smoke, dust, light vegetation. Some attenuation but no hard cutoff.
- **All-weather operation.** Rain attenuates somewhat at higher frequencies (94 GHz more than 24 GHz) but radar remains operable in conditions that disable optical sensors.
- **Limited angular resolution.** Antenna geometry sets the angular resolution; mast-mounted antennas can be larger than wearable antennas, providing better resolution, but resolution is still coarser than LiDAR.

In HALO-AD's role, mmWave radar excels at *track initiation in degraded conditions* — when fog, dust, or smoke disable the optical sensors, radar maintains coverage. The direct velocity readout also makes radar the natural primary sensor for fast-target tracking, where Doppler velocity provides the single most informative observation per frame.

### 2.3 Scanning LiDAR

Time-of-flight LiDAR with mechanical or solid-state scanning. Reference architectures span the published spectrum from spinning-mirror systems through rotating-prism systems to solid-state phased-array systems.

For long-range outdoor coverage:

- **High spatial resolution.** Sub-degree angular resolution, centimeter-class range resolution, producing dense 3D point clouds.
- **Direct geometric measurement.** Provides position, shape, and (with multi-frame analysis) trajectory of observed objects.
- **Range and atmospheric sensitivity.** Effective range degrades substantially with atmospheric attenuation; performance in fog, smoke, dust is poor.
- **Scan-cycle latency.** Typical scanning LiDAR completes one full scan in 50-100 ms; partial-field scans can be faster but produce gaps elsewhere.

In HALO-AD's role, LiDAR excels at *high-resolution refinement* — once event-based vision or radar has detected and coarsely-tracked a target, LiDAR adds detailed geometric information that supports classification, pose estimation, and precise trajectory characterization.

## 3. The Fusion Architecture

The three modalities feed into a fusion pipeline structured as follows:

### 3.1 Detection Layer (Modality-Native)

Each modality processes its input in its native form:

- **Event-based vision.** Spike trains processed with neuromorphic-aware algorithms (event-clustering, optical-flow on spike streams).
- **mmWave radar.** Range-Doppler maps processed with detection-and-tracking algorithms native to FMCW radar (CFAR detection, Doppler-bin tracking).
- **LiDAR.** Point clouds processed with point-cloud algorithms (clustering, segmentation, semantic classification).

Each detection layer outputs *detections*: time-stamped observations with position estimates, velocity estimates (where available), and confidence scores.

### 3.2 Track-Level Fusion

Modality-specific detections feed into a track-level fusion layer that associates detections from different modalities to a common set of tracks. Standard track-level-fusion algorithms (Joint Probabilistic Data Association, Multiple Hypothesis Tracking) provide the mathematical substrate; the published literature on radar-camera fusion in autonomous-driving applications is the closest direct precedent.

The fusion layer maintains track state that integrates contributions from each modality:

- A track may be initiated by event-based vision, refined by radar (which adds velocity), and further refined by LiDAR (which adds high-resolution geometry).
- A track may be initiated by radar in degraded conditions, then refined by event-based vision once optical conditions improve.
- A track that loses one modality's contribution (e.g., LiDAR drops due to atmospheric degradation) continues with the remaining modalities, with appropriate confidence reduction.

### 3.3 PentaTrack Integration

The fused tracks feed PentaTrack as a predictive-tracking substrate. PentaTrack's predictive-center field is computed for each track based on the fused state; the predictive-center field supports coverage-handoff decisions, multi-effector-engagement formulations (relevant to TALON-MESH; cross-referenced here), and trajectory-extrapolation queries.

PentaTrack's parameter choices for the HALO-AD context follow `physics_models.md` (cross-project document): recursion depth 2-3, velocity-weighted prediction with cosine strategy, object-type awareness for the typical threat-class taxonomy, drift analysis with adaptive-drift enabled.

## 4. Modality Complementarity

The fusion architecture exploits the modalities' complementarity along several dimensions:

| Dimension | Event Vision | mmWave Radar | LiDAR |
|---|---|---|---|
| Latency | Microseconds | Milliseconds | Tens of milliseconds |
| Spatial resolution | Moderate | Coarse | High |
| Velocity readout | Indirect (gradient on spike stream) | Direct (Doppler) | Indirect (frame-to-frame) |
| Through-obscuration | No | Yes | No |
| All-weather | No | Yes | No |
| Static-scene observability | No (events require change) | No (motion-mode default) | Yes |
| Range | Shorter (illumination-limited) | Longer | Long (clear weather only) |

The fusion architecture's value is in the *complementarity*: each modality covers failure modes of the others. Optical-sensor failure in fog is covered by radar; radar's coarse spatial resolution is covered by LiDAR; LiDAR's scan-cycle latency is covered by event-based vision.

## 5. Atmospheric Coupling

Each modality's effective range depends on atmospheric conditions. The fusion layer incorporates atmospheric awareness:

- In clear conditions, all three modalities operate at their nominal ranges; the fusion layer prioritizes LiDAR's high-resolution information for established tracks while using event-based vision and radar for early detection.
- In degraded optical conditions (fog, smoke), the fusion layer downweights or excludes LiDAR and event-based vision, relying on radar.
- In severely-degraded radar conditions (heavy rain at 94 GHz), the fusion layer adapts its radar-confidence weighting and may suggest sensor-frequency reconfiguration if the array supports multi-frequency operation.

Atmospheric awareness is computed continuously from environmental sensors (humidity, visibility, wind) plus the sensors' own degradation observations (signal-to-noise on returns, observed clutter levels). The simulator's atmospheric-models documentation specifies the underlying parameterization.

## 6. Multi-Mast Track Fusion

When multiple masts observe the same target, their per-mast tracks must be fused into a single shared track. The geometric problem is more complex than single-mast fusion because:

- Each mast observes the target from a different perspective, producing different per-modality detections.
- Inter-mast time synchronization is finite (sub-millisecond is achievable but requires explicit infrastructure).
- Track-association across masts must reconcile observations that may have moderate position-estimate disagreement due to perspective differences.

The simulator implements published multi-sensor track-fusion algorithms (covariance intersection, federated Kalman filtering) and characterizes the fusion-quality vs. inter-mast-bandwidth trade-off. Higher bandwidth supports more granular fusion (sharing raw detections rather than only fused tracks); lower bandwidth supports fewer granular fusion (track-state-only sharing). Researchers studying particular operational constraints can select the appropriate fusion granularity for their scenario.

## 7. Adversarial Robustness

Each modality has characteristic adversarial-failure modes:

- **Event-based vision.** Vulnerable to bright illumination (saturating events), to scene-rate-overload (more events than the sensor bandwidth supports), and to optical decoys (hot or bright objects that produce events similar to threats).
- **mmWave radar.** Vulnerable to RF jamming, to chaff (released to confuse the radar), and to RF decoys (small emitters mimicking target returns).
- **LiDAR.** Vulnerable to optical jamming (intentional bright illumination at the sensor's operating wavelength), to retroreflective decoys, and to optical-obscurant attacks (deliberate fog or smoke).

The fusion architecture's value here is *modal redundancy*: an attack effective against one modality typically does not also disable the others. Researchers studying adversarial-robustness can configure scenarios that exercise specific attack vectors and observe the fusion-architecture's degraded performance.

## 8. The Composition with Coverage Geometry

The fusion architecture's outputs feed back into the coverage-geometry analysis. A scenario's *effective coverage* is not just the geometric coverage from `coverage_geometry.md`; it is the geometric coverage filtered through the fusion-architecture's detection-probability and track-quality outputs.

A voxel that is geometrically observable from three masts but where atmospheric conditions disable two of those masts' optical sensors has effective coverage from one radar; the detection probability and track quality are correspondingly reduced.

The simulator integrates fusion-output and coverage-geometry into a single *effective-engagement-volume* computation per scenario. This is the simulator's primary research output for each deployment-scenario configuration.

## 9. Civilian Transfer

The fusion architecture transfers to several civilian outdoor-perception domains:

- **Autonomous-driving perception.** Multi-modal radar-LiDAR-camera fusion is the dominant published approach; HALO-AD's framework adapts directly with appropriate parameter changes for the shorter-range automotive context.
- **Air-traffic-management surveillance.** Multi-sensor fusion for airspace surveillance shares many architectural elements; the parameter regime is similar to HALO-AD's.
- **Maritime surveillance.** Coastal-surveillance and harbor-monitoring systems use similar multi-modal architectures; the LiDAR component is sometimes replaced by surface-radar, but the fusion architecture is otherwise analogous.
- **Wildlife monitoring at large scale.** Distributed sensor networks for wildlife observation use a subset of the modalities (typically event-based vision plus environmental sensors, with radar at larger installations) with similar fusion principles.
- **Sports tracking at scale.** Stadium-scale player and ball tracking uses multi-modal fusion (typically multiple cameras plus radar for high-speed events) with the same fundamental architecture.

The civilian-transfer examples will include at least one example per civilian domain.

## 10. References

- Published mmWave radar literature: IEEE Transactions on Aerospace and Electronic Systems; conference proceedings of IEEE Radar Conference.
- Published event-based vision literature: IEEE Transactions on Pattern Analysis and Machine Intelligence; conference proceedings of CVPR, ICCV, ECCV.
- Published LiDAR literature: Optics Express; conference proceedings of SPIE.
- Published multi-sensor-fusion literature: IEEE Transactions on Aerospace and Electronic Systems (multi-target tracking); conference proceedings of FUSION (International Conference on Information Fusion).
- Published autonomous-driving sensor-fusion literature: IEEE Transactions on Intelligent Vehicles; conference proceedings of IV (Intelligent Vehicles Symposium).

The maintainers update this references section as the published literature evolves.
