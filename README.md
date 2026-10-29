# LiRIO: Multi-Modal, GNSS-Light Localisation for Off-Highway Vehicles

> **Team FusionForge** | Tata Elxsi TELIPORT Season 3 — Case Study on Off-Highway Vehicles
>
> *Gulshan Choudhary • Lavanya Bhatnagar • Sibasish Barik*

---

## About This Repository

This repository documents **our approach** to Case Study 3: reliable localisation for off-highway vehicles (mining, forestry, construction) where GNSS is weak or unavailable and sensors like LiDAR can be degraded by dust, smoke or featureless terrain.

**Note on scope:** This is a design and approach proposal. We have **not** deployed this system on a physical vehicle, and we do not report our own field results. The sensor setup and fusion architecture are inspired by published work (see [References](#references)), and any performance figures mentioned there belong to those authors.

---

## The Problem, As We Understood It

Off-highway vehicles face conditions that break single-sensor localisation:

- **GNSS** drops out in mines and tunnels and suffers multipath near walls and structures.
- **LiDAR** loses useful features in smoke, dust or geometrically repetitive areas (long tunnels, open fields).
- **IMU-only** estimates drift quickly over time.

So the core question became: *how do we keep a trustworthy pose estimate when any one sensor becomes unreliable?*

---

## Our Approach

The idea is to combine sensors whose failure modes don't overlap, and to let the system notice when one is degrading and lean on the others.

| Sensor | Why it helps | Where it struggles |
| :--- | :--- | :--- |
| 3D LiDAR | Detailed geometry for scan matching | Smoke, dust, featureless areas |
| 4D FMCW Radar | Sees through obscurants, gives direct Doppler velocity | Sparse, noisy point clouds |
| IMU | High-rate motion, always available | Drifts over time |
| GNSS (RTK) | Global position anchor | Blocked or multipath-affected |

### Key ideas we explored

1. **Radar-assisted removal of moving objects.** Radar's Doppler velocity can flag moving objects (people, other vehicles). Matching LiDAR points can then be masked so they don't corrupt the map.
2. **Efficient local mapping.** Use an incremental voxel map (iVox-style) with GICP scan-to-map matching, which suits unstructured outdoor terrain.
3. **Adaptive switching between LiDAR and Radar.** Monitor LiDAR feature quality; when it falls below a threshold, switch from LiDAR-Inertial Odometry (LIO) to Radar-Inertial Odometry (RIO) until LiDAR recovers.
4. **Global consistency through a factor graph.** A high-rate filter (IESKF) handles local state estimation, while a factor graph backend (e.g. GTSAM) adds loop closures and GNSS factors. GNSS measurements are weighted adaptively, using a residual consistency check to reject outliers such as multipath at tunnel exits.

---

## Workflow

```
   Sensors: LiDAR · 4D Radar · IMU · GNSS
                     │
                     ▼
   1. Pre-processing
      - IMU pre-integration, LiDAR motion undistortion
      - Radar noise filtering (CFAR), moving-object masking
                     │
                     ▼
   2. Odometry (front-end)
      - Check LiDAR feature quality
      - Good   → LiDAR-Inertial Odometry
      - Poor   → Radar-Inertial Odometry (fallback)
      - IESKF for high-rate state updates
                     │
                     ▼
   3. Optimisation (back-end)
      - Factor graph with odometry, loop closures, GNSS
      - Adaptive GNSS weighting / outlier rejection
                     │
                     ▼
   Output: continuous pose estimate
```

---

## Target Requirements We Designed Toward

These are the goals we used to guide design choices. They are targets, not measured results.

- Position accuracy in the sub-metre range for navigation, tighter for precision tasks
- Low heading error under vehicle dynamics
- High availability over long runs, including GNSS-denied stretches
- Low latency with a steady pose update rate
- Fits a modest power budget on embedded/edge compute
- Quick recalibration after a sensor is replaced

We also kept functional safety standards for off-highway and road vehicles (ISO 19014, ISO 26262) in mind as context for why redundancy matters; this proposal does not claim compliance with either.

---

## Possible Next Steps

- Validate the pipeline on public datasets with LiDAR, radar and IMU data
- Tune the LIO/RIO switching threshold and study transitions for pose jumps
- Compare against existing LiDAR-inertial baselines on the same data
- Move toward hardware testing if resources allow

---

## References

1. Nissov et al. (NASA JPL / NTNU) — *ROAMER: Robust Offroad Autonomy using Multimodal State Estimation with Radar Velocity Integration.*
2. Qian et al. (Zhejiang University) — *AF-RLIO: Adaptive Fusion of Radar-LiDAR-Inertial Information for Robust Odometry.*
3. Zhu et al. (Anhui Jianzhu University) — *Research on 3D LiDAR Outdoor SLAM Algorithm Based on LiDAR/IMU Tight Coupling.*
4. Tata Elxsi TELIPORT Season 3 — *Case Study 3: Off-Highway Vehicle Navigation.*
