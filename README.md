# Synth Riders Kinematics

**[Open the Synth Riders Kinematic Analyzer](https://marinusofvr.github.io/Synth_Riders_Kinematics/Synth_Riders_Kinematic_Analyzer.html)**

**A standalone Synth Riders beatmap viewer and biomechanical analysis tool.** Reconstruct hypothetical hand trajectories from a map, infer body movement, and explore joint kinematics, muscle loading, fatigue, overhead exposure, and estimated exercise energy.

Use the hosted web page or download the standalone HTML for offline use. Analysis runs locally in your browser in either case. No installation, account, or analysis server is required. Map files are read without modification.

The central question is: **What movement and physical demand could this choreography produce under a specified player and movement strategy?** The tool models that question; it does not record actual gameplay.

## Getting started

1. [Open the analyzer](https://marinusofvr.github.io/Synth_Riders_Kinematics/Synth_Riders_Kinematic_Analyzer.html) in a desktop browser with WebGL and WebAssembly enabled. For offline use, download `Synth_Riders_Kinematic_Analyzer.html` from the repository and open it locally. Download the file itself; GitHub's source preview does not run the application.
2. Choose an **unencrypted `.synth` file provided by its mapper**. Encrypted downloads are unsupported.
3. Select a difficulty, body preset, movement style, and targeting strategy. Adjust optional measurements and model settings as needed.
4. Optionally select a MIDI file to add instrument lanes to the timeline.
5. Click **Analyze map**. The progress window shows trajectory planning and body analysis; **Stop analysis** cancels the calculation.
6. Press **Play**, seek using either timeline, or select a muscle on the figure or in the sidebar to follow its modeled demand.

Use **Files & settings**, below the muscle graphs, to change several settings together. Changes take effect when **Analyze map** is selected; Cancel restores the applied settings.

## Viewer and outputs

- A 3D player, incoming notes and rails, approximate wall geometry, hand paths, and optional target-zone spheres.
- Behind, Side, and Front camera presets. The camera starts locked so page scrolling does not alter the view. **Unlock View** enables wheel zoom, drag orbit, and right-drag or Shift-drag pan.
- Artist, song, map time, bar/beat, and wall count at the upper-left of the frame.
- Live muscle-demand bars, a song-wide demand timeline, and a close-up hand/MIDI timeline.
- A joint-kinematics panel with rear and side skeleton projections, hip shift, knee dip, trunk angles, shoulder elevation, and elbow flexion.
- Movement advisories with episode timing, rationale, model assumptions, and research references.
- Experimental energy estimates and a simulated metabolic-power trace.
- **Download infographic:** portrait 1080 × 1350 or landscape 1600 × 900 PNG, with current player settings, motion statistics, demand summaries, and a muscle-colored pose.
- **Download report:** a self-contained HTML report with its infographic, analysis settings, results, coverage, and references. Use the browser's print function for PDF.

Measurements are shown in metric and US units. Exported summaries describe the currently selected scenario. The optional player-ID field is omitted from the exported report.

## Analysis pipeline

The numerical pipeline has distinct stages:

1. **Decode the chart:** read native note, rail, timing, and wall data.
2. **Reconstruct controller paths:** optimize timed hand trajectories within declared target tolerances.
3. **Infer body pose:** fit a body and movement-style hypothesis to the reconstructed hands.
4. **Calculate mechanical demand:** estimate joint moments from gravity and segment accelerations.
5. **Allocate muscle demand:** solve for a torque-balanced allocation under the selected capacity and stabilization hypotheses.
6. **Accumulate state and exposure:** track peripheral fatigue and repeated/elevated movement.
7. **Estimate energy:** calculate experimental movement-derived expenditure or a separate fixed-MET scenario.

The avatar is a visualization of these assumptions. It is not motion capture, and matching the hands does not uniquely determine the elbows, shoulders, spine, pelvis, or legs.

### 1. Hand-trajectory reconstruction

The planner operates in the map's horizontal/vertical hit plane at **60 samples per second**, using approximately **12-second windows with 2.5 seconds of overlap**. Clarabel, compiled to WebAssembly, solves a quadratic objective with second-order-cone target constraints.

The objective penalizes discrete jerk, acceleration, velocity, and displacement from a resting hand position. “Effort” here is a trajectory regularization term; the planner does not directly minimize the later muscle-force or calorie result.

Three targeting hypotheses are available:

| Targeting strategy | Head radius | Rail-tube radius |
|---|---:|---:|
| Precise | 1 cm | 5 cm |
| Relaxed rails | 1 cm | 15 cm |
| Relaxed targets | 5 cm | 15 cm |

Notes, rails, bundles, clusters, and special notes are converted into hand targets using explicit reconstruction rules. These rules are hypothetical player strategies. Bundled rails and scattered clusters retain simplifications.

If a window cannot be solved with hard targets, a relaxed solve permits penalized target violations. Reconstruction residuals remain reported; a returned path does not prove every object was successfully played.

The displayed **5/10/15 cm perfect/good/poor radii are unverified visualization assumptions**, separate from the planner's targeting tolerances. Target-zone counts are geometric fit estimates, not official scoring predictions.

### 2. Body and pose assumptions

The model uses stature-scaled segment lengths, sex-specific segment-mass fractions, approximate attachment geometry, and controller mass. The initial body presets are:

| Preset | Height | Mass |
|---|---:|---:|
| 5th percentile woman | 1.52 m | 50 kg |
| Median woman | 1.63 m | 66 kg |
| Median man | 1.76 m | 85 kg |
| 95th percentile man | 1.88 m | 108 kg |

These are rounded anthropometric scenarios. Their labels do not establish corresponding strength percentiles, and a preset need not match an individual player's proportions or capacity. Stature and body-mass overrides are available.

The movement styles **Planted**, **Typical**, and **Dancer** change the assumed distribution of movement between the arms and body. They are chosen posture/coordination priors, not categories fitted to measured gameplay.

The analysis coordinate system is X toward player-right, Y upward, and Z forward toward incoming notes. Map coordinates retain their spacing rather than scaling with player height. The default map-center height is 0.686 × stature; the default hit plane is 35 cm ahead of the shoulders, with 25 and 45 cm sensitivity options. Each controller has a nominal mass of 0.15 kg and an approximate 8 cm grip-to-hit-point reach.

Feet remain fixed. Pelvis lowering helps satisfy leg-length closure during inferred movement. This is a geometric solution, not a validated stepping, crouching, or balance strategy. Unreachable hand targets and incompatible skeletons are retained as quality failures.

**Walls are displayed, but wall avoidance does not constrain the inferred body pose in this HTML tool.** Wall shapes and rotations use approximate native templates.

### 3. Joint mechanics and muscle loading

Joint moments use point-mass inverse dynamics with gravitational and acceleration terms. Moving shoulder and trunk positions affect the moments. Segment rotational inertia, full joint-contact mechanics, and several anatomical degrees of freedom are omitted.

For each solved arm frame, the muscle-sharing problem is:

```math
\begin{aligned}
\underset{a,z}{\operatorname{minimize}}\quad &z\\
\text{subject to}\quad &A a=b,\\
&0\leq a_j\leq z.
\end{aligned}
```

Here, `b` is the required joint-moment vector, `A` contains modeled muscle torque capacities, and `a` contains normalized muscle-force demands. The solution minimizes the largest required demand while balancing the modeled torques. Biceps and long-head triceps couple shoulder and elbow actions within the same solve.

Capacity depends on nominal force constants, simplified moment arms, body/capacity scaling, the selected speed-capacity approximation, and remaining non-fatigued capacity. Optional rotator-cuff inequalities can be included inside the equilibrium solve. Their fixed ratios are a stabilization sensitivity hypothesis, not a complete shoulder-stability model.

The solver verifies equilibrium, nonnegativity, and selected constraints. Failed or pose-incompatible frames remain unresolved.

| Displayed region | Interpretation |
|---|---|
| Arm muscles | Allocated force relative to modeled available force capacity |
| Trunk | Allocation among lumped multiaxis torque actuators |
| Scapular muscles | Heuristic reaction-torque allocation |
| Legs | Approximate hip/knee torque allocation |

A displayed 50% is **not measured EMG, tissue strain, or a calibrated percentage of that player's maximum voluntary contraction**. An optimal peak above 100% means the modeled demand exceeds the selected scenario's capacity; it does not establish injury or actual delivered force.

Individual muscle allocations are nonunique. The minimum possible peak is conditional on the specified model, while one muscle's value can change substantially under another equally feasible recruitment strategy. Display smoothing occurs after the solve and does not itself preserve instantaneous torque balance.

### 4. Fatigue and movement exposure

The peripheral fatigue model tracks resting, active, and fatigued fractions whose sum is one. Activation transfers resting capacity to active capacity; fatigue transfers active capacity to fatigued capacity; recovery returns fatigued capacity to rest. The integration uses bounded steps no longer than 0.005 seconds.

Available capacity is `1 − fatigued fraction`. Fatigue/recovery rates and conditioned-player changes are engineering sensitivities, not individual endurance measurements. Unknown force or pose frames freeze state rather than becoming rest; subsequent history retains incomplete-confidence information.

Exposure summaries distinguish upper-arm elevation from hand height above the modeled head. They include duration, bouts, longest bout, and rolling duty fractions. Advisories use these characteristics to identify sections for review. They do not calculate an injury probability, certify gameplay safety, or automatically reject a map. Effort colors are a visualization scale, not clinical limits.

Advanced comparisons include fresh versus fatigue-adjusted capacity, torque-only versus cuff-constrained recruitment, joint-speed versus isometric capacity, and filtered versus prescribed-pose trunk moments. The filtered-trunk option does not solve the corresponding compensatory movement; neither trunk option constitutes complete spine dynamics.

### 5. Exercise-energy estimation

**Motion-derived experimental model** is the default. It transfers a published joint-space heat/work relationship to the inferred gameplay movement:

```math
P_{\mathrm{active}}=\sum_i\left(h_{am,i}|\tau_i|+h_{sl,i}|\tau_i\dot q_i|+\tau_i\dot q_i\right).
```

Torque-dependent heat allows static muscular support to consume energy; the absolute-power term and signed mechanical work distinguish movement from support and positive from negative work. Power integrates over time, with **4,184 J = 1 kcal**. The mechanics layer evaluates 23 joint degrees of freedom using relative joint velocities.

The model assumes fixed feet, equal upper-body wrench sharing between hips, and simplified leg centers of mass. It omits extra coactivation, stepping, wrist/neck/scapular work, passive energy storage, and movement not implied by the map. Published coefficients were developed for walking; their accuracy has not been established for Synth Riders.

The default metabolic scenario uses age 40 and group-average maximum knee torque of 107.6 N·m for female scenarios or 212.4 N·m for male scenarios. These are explicit defaults, not measurements of the player. Age and optional knee torque can be changed. A strength multiplier alone does not create a training-efficiency discount for identical motion.

The viewer extrapolates the mean cost of compatible samples across missing time and labels the result conditional. Inspect coverage: excluded difficult poses may have systematically different costs. This extrapolation is not equivalent to measuring the missing movement.

**Fixed MET activity scenario** is a separate comparison method:

```text
gross kcal = MET × 3.5 × mass_kg / 1000 × 5 × duration_minutes
active kcal = gross kcal − the 1 MET resting baseline
```

With these conventions, 1 MET equals 1.05 kcal/kg/hour. MET selection is an intensity assumption, not a measured map-specific result. Total/gross includes the baseline; active excludes it. Menus, breaks, additional dancing, and post-exercise recovery are outside the analyzed map interval. The power trace represents simulated demand, not heart-rate timing.

## MIDI and timing

MIDI is optional and adds instrument timing lanes; it does not drive the muscle calculation. To obtain MIDI, transcribe source audio using [MuScriptor](https://muscriptor.kyutai.org/) and select the downloaded file manually. Transcription is an external step; the analyzer does not upload audio for you.

The map's `Offset` is stored in milliseconds:

```text
map time = source-audio time + Offset / 1000
MIDI map time = original MIDI time + Offset / 1000 + MIDI fine adjustment
```

**MIDI fine adjustment** is under Advanced model settings and normally stays at zero. Original MIDI timestamps are retained, so repeated analysis does not accumulate offsets. Audio playback follows the same map/audio clock conversion.

## Interpreting comparisons

Compare maps or edits with the same body dimensions, movement style, targeting policy, capacity, fatigue, hit-plane depth, stabilization, trunk-dynamics, and metabolic settings. Changing a setting can change the inferred movement as well as its relative demand.

- **P95 demand** summarizes the 95th percentile of resolved samples; check its usable-time coverage.
- **Total energy** includes duration. Use kcal/min when comparing modeled intensity.
- **Hand travel and speed** describe reconstructed paths, not observed controllers.
- **Missing data** are excluded from force statistics rather than assigned zero. Energy extrapolation is separately labeled.
- **High demand** is a prompt to inspect choreography, reconstruction, and assumptions; it is not proof of harm.
- **Low demand** does not establish low injury risk, balanced exercise, or enjoyable mapping.

## Verification and limits

The accompanying source package documents numerical and browser checks. These verify implementation behavior under specified fixtures; they do not validate actual player motion or physiology.

The retained Python CVXPY/Clarabel comparison reports a maximum sampled trajectory discrepancy of approximately **1.537 × 10⁻⁶ m** and an RMS discrepancy of **4.470 × 10⁻⁷ m** for the included chart excerpt. This is solver agreement for that case, not gameplay prediction accuracy.

Browser checks cover import, timing offsets, analysis controls, camera locking, coordinate orientation, joint-panel updates, exports, missing-data handling, and clearing/reopening maps. Archive checks cover corruption, duplicate entries, encrypted entries, and metadata decoding. Documented browser verification uses Chromium on Linux; other browsers and physical mobile devices are not established by those checks.

Major unresolved uncertainties include inferred posture, simplified muscle geometry and moment arms, muscle-sharing choices, shoulder stabilization, trunk mechanics, speed/capacity scaling, fatigue calibration, and transfer of metabolic coefficients. There is no matched gameplay motion-capture, EMG, dynamometry, or calorimetry validation establishing individual accuracy.

## Files, privacy, and runtime

The importer supports unencrypted ZIP-based `.synth` archives with JSON `beatmap.meta.bin`, optional `track.data.json`, and charted difficulty tables. The current import path requires identifiable Ogg or MP3 audio. STORE and DEFLATE compression and UTF-8/BOM or UTF-16 metadata are supported.

Encrypted entries, unsupported compression, multi-volume archives, ZIP64, duplicate names, unsafe archive paths, and CRC corruption are rejected. Current limits include 768 MiB archive/expanded data, 100,000 chart objects per difficulty, and one hour of chart duration; available browser memory can impose lower limits.

Parsing, path optimization, and body analysis run in a Web Worker. Libraries and WebAssembly are embedded in the HTML, so analysis does not require a network connection. Selected map/audio/MIDI data are processed locally. External reference links and manual use of MuScriptor are separate network activity.

Matching calculations can be reused within the current session. Closing/reopening the application starts fresh: this release has no persistent saved-analysis cache and does not write MIDI or analysis back into `.synth` files. **Clear map** releases the loaded files, audio, worker, and results.

## Building from source

Running the HTML requires no build tools. If using the accompanying source distribution, its root contains `build.py`, `package.json`, `src/`, `vendor/`, `tests/`, and `reference/`.

From that directory, with Python 3 and Node.js installed:

```sh
npm ci
npm run build
npm test
```

The build produces `Synth_Riders_Movement_Analyzer.html`. Browser integration checks additionally require FFmpeg and Playwright's Chromium:

```sh
npx playwright install chromium
npm run test:browser
```

The source package includes `VALIDATION.md`, model engineering notes, a pose/recruitment assessment, metabolic-model notes, and third-party notices. Some reference documents discuss the broader modeling project; those interfaces and generation features are not necessarily exposed by this HTML application.

## Scientific basis

The embedded evidence registry provides source links and the limits of transferring each finding to this model. Selected foundations are:

| Reference | Role in the model |
|---|---|
| de Leva (1996), *Adjustments to Zatsiorsky–Seluyanov's segment inertia parameters*, Journal of Biomechanics 29:1223–1230 | Approximate segment masses and centers of mass |
| Drillis & Contini (1966), as tabulated in Winter's *Biomechanics and Motor Control of Human Movement* | Stature-scaled segment lengths |
| Gordon et al. (2014), ANSUR II | Rounded anthropometric body scenarios |
| [Holzbaur, Murray & Delp (2005)](https://doi.org/10.1007/s10439-005-3320-7) | Upper-limb maximum-force constants and musculoskeletal modeling context |
| [Murray, Delp & Buchanan (1995)](https://pubmed.ncbi.nlm.nih.gov/7775488/) | Dependence of moment arms on joint position |
| [Belli et al. (2023)](https://doi.org/10.1371/journal.pone.0295003) | Shoulder-stability and recruitment uncertainty |
| [Michaud et al. (2023)](https://doi.org/10.3389/fphys.2023.1167748), [Michaud et al. (2024)](https://doi.org/10.3389/fphys.2024.1366172) | Fatigue/recruitment coupling and three-compartment-model limitations |
| [Roberts, Hillstrom & Kim (2016)](https://doi.org/10.1371/journal.pone.0168070) | Experimental joint-space heat/work energy relationship |
| [Cruz & Yang (2022)](https://doi.org/10.1371/journal.pone.0267120) | Knee-torque scenarios and metabolic-transfer considerations |
| [Hicks et al. (2015)](https://pmc.ncbi.nlm.nih.gov/articles/PMC4321112/) | Task-specific verification, validation, and sensitivity practices |

These sources support components and modeling approaches. They do not collectively validate this application's predictions or establish Synth Riders injury thresholds.

## Third-party components

| Component | Use | Upstream license |
|---|---|---|
| [Three.js](https://github.com/mrdoob/three.js), including OrbitControls | 3D rendering and camera controls | MIT |
| [fflate](https://github.com/101arrowz/fflate) | DEFLATE decompression | MIT |
| [Tone.js MIDI](https://github.com/Tonejs/Midi), midi-file, array-flatten | MIDI parsing and support utilities | MIT |
| [Clarabel](https://github.com/oxfordcontrol/Clarabel.rs) | Conic optimization | Apache 2.0 |
| [cvxjs](https://github.com/SteveDiamond/cvxjs) | Clarabel WebAssembly wrapper | Apache 2.0 |
| [Synth Mapping Helper](https://github.com/adosikas/synth_mapping_helper) | Format reference and inherited wall definitions | MIT |

Third-party components retain their respective licenses and attribution requirements. The source distribution contains additional license and provenance records in `THIRD_PARTY_NOTICES.md` and `vendor/`.
