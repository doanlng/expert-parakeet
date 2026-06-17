# Muay Thai Kick Form Analysis — Data Architecture

## Overview

A v1-scoped pipeline for analyzing roundhouse kick form from single-camera slow-mo video, scoring deviation against a self-chosen reference rep, and generating coaching feedback via a vision-capable LLM. Structured as a small medallion-style architecture (bronze/silver/gold) so each stage has a clear contract and can be iterated independently.

## Data layers

| Layer | Contents | Format |
|---|---|---|
| Bronze | Raw per-frame pose landmarks, unfiltered | Parquet, one file per rep |
| Silver | Smoothed, phase-normalized joint angle time series | Parquet, one file per rep |
| Gold | DTW-aligned deviation scores vs. reference rep | Parquet or JSON, one record per rep |
| Feedback | LLM-generated coaching cues, ranked | JSON |

## Pipeline stages

| # | Stage | Input | Process | Output | Tech |
|---|---|---|---|---|---|
| 1 | Capture | — | Phone slow-mo (120–240fps), tripod, fixed framing | Raw video clips, one per rep | Phone camera |
| 2 | Pose extraction | Video clip | BlazePose inference, frame by frame | Bronze: landmark coords + visibility | `mediapipe` (Python) |
| 3 | Feature engineering | Bronze | Joint angle calc (knee chamber, hip rotation, pivot foot, torso lean), Savitzky-Golay smoothing, normalize timeline to 0–100% of kick | Silver: angle time series | `numpy`, `scipy` |
| 4 | Reference alignment | Silver (current rep + reference rep) | Dynamic time warping to align curves despite tempo differences | Aligned angle curves | `dtaidistance` or `tslearn` |
| 5 | Deviation scoring | Aligned curves | Diff at fixed checkpoints (e.g. chamber @ 40%, impact @ 80%); rank by magnitude | Gold: ranked deviation report | Plain Python |
| 6 | LLM feedback | Gold + key frame images at checkpoints | Multimodal call: structured deviations + frames → ranked natural-language cues | Coaching cues (JSON) | Claude API (vision) |
| 7 | Visualization | Silver + Gold | Skeleton overlay on video, angle-vs-phase curve plots (rep vs. reference) | Charts / annotated video | `matplotlib`, OpenCV |
| 8 | Iteration | Gold across sessions | Track checkpoint deviation trend over time, LLM writes progress summary | Session-over-session report | Claude API (text) |

## Schemas

**Bronze**
```
rep_id, frame_idx, timestamp_ms, landmark_id, landmark_name, x, y, z, visibility
```

**Silver**
```
rep_id, phase_pct, angle_name, angle_value_deg, smoothed (bool)
```

**Gold**
```
rep_id, checkpoint_phase, angle_name, your_value, reference_value, deviation, deviation_rank
```

**Feedback**
```
rep_id, cue_text, priority_rank, source_checkpoint
```

## Folder layout (local, v1)

```
data/
  raw_video/{rep_id}.mp4
  bronze/{rep_id}.parquet
  silver/{rep_id}.parquet
  gold/{rep_id}.parquet
  feedback/{rep_id}.json
reference/
  reference_rep.parquet   # your cleanest rep, used as alignment target
```

## v1 scope vs. future extensions

| Aspect | v1 | Extension |
|---|---|---|
| Camera | Single, side-on | Add front-on for hip rotation accuracy |
| Reference | Your own cleanest rep | Coach's rep, or population average from multiple good reps |
| Feedback | Numeric deviations → LLM cues | Add RAG grounding from coaching literature |
| Segmentation | Manual clipping per rep | Auto-detect kick start/end |
| Storage | Local Parquet files | DuckDB or SQLite for querying across sessions |

The biggest known failure point stays the same as discussed: BlazePose's depth and landmark confidence degrade when the kicking leg occludes the torso mid-kick, so the silver-layer step should drop or flag low-visibility frames rather than silently smoothing over bad data.
