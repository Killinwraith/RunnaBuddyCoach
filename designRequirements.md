# Running Form Analyzer — Requirements

Nov 16, 2026 · @Hetang Mehta

## Overview

A personal tool that turns a slo-mo treadmill clip into a running-form report, built first for one user (Hetang) training for IRONMAN 70.3. It has two goals of equal weight:

- **Form:** trustworthy metrics, tracked over time, that show what to change.
- **Learning:** the full vision-ML lifecycle: start from a pretrained model, then train, evaluate, deploy on-device and keep it reliable.

v0 ships as soon as possible with no model training, so it is useful from the first week. Other runners are a later goal and are not designed for until after v2.

The tool tracks Hetang's own changes over time. It does not judge his form against an ideal. Any single biomechanical variable explains only 4–12% of the differences in running economy between runners ([Van Hooren et al. 2024](https://pmc.ncbi.nlm.nih.gov/articles/PMC11127892)), so the report shows trends and leaves conclusions to him.

## Scope and constraints

| Area             | Decision                                                                                                                            |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Capture          | iPhone 18 Pro Max on a tripod, side view, 240 fps slo-mo, on a treadmill; belt speed entered by hand                                |
| Capture protocol | Same belt speed, taped tripod spot, lens, and minute of the run every session, so trends show changes in form rather than in setup  |
| Video ingestion  | Export the unmodified original file and read per-frame timestamps; reject clips exported at 30 fps or with the slow motion baked in |
| Symmetry         | Only from rear- or front-view clips; side-view clips never report left vs right                                                     |
| Analysis timing  | After the run (upload a clip, get a report); live cues come later                                                                   |
| Platform         | Python command-line tool on a laptop first, iPhone app later                                                                        |
| Stack            | Python, PyTorch, OpenCV, a pretrained pose model, SQLite                                                                            |
| Effort           | No fixed weekly hours, so work is split into tasks of 1–2 hours that fit any free slot                                              |
| Out of scope     | Outdoor footage, more than one runner in frame, injury diagnosis or medical advice, cloud hosting                                   |

## Milestones

&#91;embedded content: roadmap · 4 milestones, 3 gates\]

The order is fixed: metrics and trends, then symmetry, then cues. Training starts in v1 and deployment in v3, so each milestone also teaches one stage of the ML lifecycle.

## Architecture

&#91;embedded content: architecture · 8 stages, validation loop\]

The pipeline is a chain of stages that each read and write files, and the validation loop scores every change. Three rules keep it reliable:

- **Files between stages:** any stage can be rerun alone, so the slow pose step never repeats when only smoothing or metrics change.
- **One keypoint format, models behind adapters:** swapping the pose model or event detector never touches the metrics code, and the Core ML port in v3 replaces one adapter.
- **Versioned outputs:** every keypoint file, metric row and report records the model and config version, so old sessions can be re-scored after a model change.

## Metrics and accuracy targets

Metrics are ordered by how strongly they relate to running economy, and every metric is checked against a reference before the report shows it. In a 2024 meta-analysis, bounce was a moderate predictor of economy, cadence a small one, and contact time and joint angles at contact showed no clear link ([Van Hooren et al. 2024](https://pmc.ncbi.nlm.nih.gov/articles/PMC11127892)). The targets are proposals to confirm once v0 produces real numbers.

| Metric                                | Definition                                                                                  | View | Checked against                 | Milestone |
| ------------------------------------- | ------------------------------------------------------------------------------------------- | ---- | ------------------------------- | --------- |
| Vertical bounce (% leg length, or cm) | Hip rise and fall per stride; cm only when a known-length marker is in the running plane    | Side | Marker in frame                 | v0        |
| Cadence (steps/min)                   | Steps per minute, from foot-strike events                                                   | Side | Apple Watch cadence; hand count | v0        |
| Shin angle at contact (°)             | Shin vs vertical at first contact; replaces overstride in cm, since it needs no pixel scale | Side | Hand-measured on 10 frames      | v0        |
| Foot strike type                      | Rear, mid or forefoot, from foot angle at first contact                                     | Side | Frame-by-frame review           | v0        |
| Knee angle at contact (°)             | Thigh–shin angle at first contact                                                           | Side | Hand-measured on 10 frames      | v0        |
| Trunk lean (°)                        | Shoulder–hip line vs vertical, averaged per stride                                          | Side | Hand-measured on 10 frames      | v0        |
| Ground contact time (ms)              | Foot strike to toe-off; kept mainly as a learning exercise                                  | Side | Hand-labelled frames            | v1        |
| Hip drop (°)                          | Pelvis tilt during stance; experimental until it passes the repeatability check             | Rear | Hand-measured on 10 frames      | v2        |
| Left vs right difference (%)          | Gap between sides for contact time, knee angle, hip drop                                    | Rear | Same as each metric             | v2        |

**Accuracy tolerances:** cadence within ±2 steps/min; foot strike and toe-off within ±2 frames at 240 fps (about 8 ms); sagittal angles within ±5°; hip drop within ±2°. Hip drop is only a few degrees, and hand-measured 2D pelvic drop has a smallest detectable change of about 2.7° ([Dingenen et al. 2018](https://research.monash.edu/en/publications/test-retest-reliability-of-two-dimensional-video-analysis-during-/)), so a looser tolerance would hide real change.

**Repeatability tolerances (from v0):** two clips from the same session agree within 2–3° for angles, 2 steps/min for cadence and 0.5% of leg length for bounce. A metric that fails this is hidden from trend charts.

## Data

Public running datasets are small, so they serve for benchmarking and the training data comes from Hetang's own labelled clips. The pose model itself is never trained from scratch. It is a pretrained model, and it must output heel and toe keypoints, since the standard 17-point COCO skeleton has none.

**Public datasets**

| Dataset                                                                           | What it has                                                                | Use here                                                    |
| --------------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ----------------------------------------------------------- |
| [Ole Miss stride-level benchmark](https://egrove.olemiss.edu/hon_thesis/3469)     | 73 treadmill videos, 15 runners, 4,600+ stride labels                      | Compare against form labels; availability not yet confirmed |
| [RunningPose](https://tifx04-22-10.github.io/runningpose)                         | 3D keypoint model for runners, built with Qualisys and Swedish Athletics   | Reference for keypoint choices and 3D lifting               |
| [Mendeley contact/toe-off timings](https://data.mendeley.com/datasets/pxy6j5hyjz) | 14 runners, 6 speeds, force-plate event times; no video                    | Test the event-detection logic                              |
| [Bath FootNet](https://researchdata.bath.ac.uk/965)                               | Kinematic features plus ground reaction forces for foot strike and toe-off | Pretraining idea for the v1 event model                     |

**Own-data protocol**

1. Each session: a 30 s side clip at 240 fps, at a logged belt speed, following the capture protocol in Scope. A 30 s rear clip is added from v2.
2. Log the date, speed, shoes and session type with each clip.
3. Label by stepping frame by frame and marking foot strike and toe-off for each foot. That gives about 90 steps per clip.
4. Split train and test by session, never by frame, so near-identical frames don't leak between them.
5. Lock about 5 sessions as the test set early, and never train on them.

## Reliability requirements

A metric shows in the report only if it passes the locked test set. These rules apply from v0 onward.

- **Ingestion check:** reads per-frame timestamps and rejects clips that are variable-rate beyond tolerance, exported at 30 fps, or below 200 fps on average.
- **Locked test set:** hand-labelled clips that are never trained on, used to score every version.
- **Repeatability check:** two clips per session for the first 5 sessions; a metric must meet the repeatability tolerances before it appears in trends.
- **Regression check:** runs on every change to the rules or the model, and fails if any metric's error goes past its tolerance.
- **Clip quality check:** flags or rejects clips with low keypoint confidence or the runner leaving the frame.
- **Reproducible reports:** every report records the model version, config and clip details, so old sessions can be re-scored.
- **Replace only on a win:** a new model replaces the old one only if it scores better on the test set.
- **On-device match (v3):** Core ML output must match PyTorch output within tolerance on the test set.

## v0 task breakdown

v0 is done when every treadmill run produces a report, cadence is within ±2 steps/min of the Apple Watch, and two clips from one session agree within the repeatability tolerances. Each task fits in one session of 1–2 hours.

- [ ] Record 3 test clips under the capture protocol; tape the tripod spot and place a known-length marker in the running plane
- [ ] Build ingestion: export originals, read per-frame timestamps, reject bad clips
- [ ] Compare 2–3 pretrained pose models that output feet (e.g. RTMW, MediaPipe Pose) on one clip, and pick one
- [ ] Save keypoints and confidence for every frame to a file
- [ ] Smooth the keypoints and fill short dropouts
- [ ] Detect foot strikes from heel and toe motion, using rules
- [ ] Compute cadence and vertical bounce; compare cadence with the Apple Watch
- [ ] Compute shin angle, foot strike type, knee angle at contact and trunk lean
- [ ] Hand-label one clip's foot strikes and measure the event error
- [ ] Set up SQLite storage for sessions, metrics and versions
- [ ] Run the repeatability check on two clips from one session
- [ ] Generate an HTML report: skeleton-overlay video plus the metrics
- [ ] Wrap it as a command, e.g. `analyze clip.mov --speed 10`

## Risks and open questions

| Risk                                                                                                                                                                 | Mitigation                                                                         |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Slo-mo files are variable frame rate or exported at 30 fps, so every timing metric is wrong without any error                                                        | Ingestion check reads per-frame timestamps and rejects bad clips                   |
| Rear-view hip measures are weak: one phone-video study found hip adduction poorly valid against 3D capture ([Hudl study](https://core.ac.uk/download/326029936.pdf)) | Hip drop stays experimental until it passes the repeatability check                |
| Setup drift between sessions looks like a change in form                                                                                                             | Capture protocol; repeatability check                                              |
| Treadmill frame or console hides the feet                                                                                                                            | Camera at shin height, square to the belt; check on the first test clip            |
| Far leg hidden in side view                                                                                                                                          | v0 reports the near leg only; both sides come from separate clips or the rear view |
| Jittery foot keypoints                                                                                                                                               | Smoothing plus 240 fps; flag frames with low confidence                            |
| Scope creep across four outputs                                                                                                                                      | Hold the milestone order; nothing from v1 starts until v0's gate passes            |

**Open questions**

- [ ] Which pose model? Decided by the v0 bake-off. RTMW's authors call MediaPipe's accuracy unsatisfactory ([RTMW](https://arxiv.org/html/2407.08634v1)), while one gait study rated MediaPipe best for lower-limb kinematics ([MediaPipe gait study](https://mdpi.com/1424-8220/23/14/6489)).
- [ ] Which v1 track: train an event model, or fine-tune the pose model on labelled foot frames? Decided by v0's error analysis.
- [ ] Rear view in v2: a second tripod, or a separate clip after the side one?
- [ ] Is there room behind the treadmill for a rear camera, and is the lighting good enough at 240 fps?

**Sources:** dataset links in the Data section, from web search results on 2026-11-16; open each before relying on it. Evidence: [Van Hooren et al. 2024](https://pmc.ncbi.nlm.nih.gov/articles/PMC11127892) · [Lambricht et al. 2025](https://mdpi.com/1424-8220/25/3/934) · [Dingenen et al. 2018](https://research.monash.edu/en/publications/test-retest-reliability-of-two-dimensional-video-analysis-during-/) · [Hudl validity study](https://core.ac.uk/download/326029936.pdf) · [RTMW](https://arxiv.org/html/2407.08634v1) · [MediaPipe gait study](https://mdpi.com/1424-8220/23/14/6489).
