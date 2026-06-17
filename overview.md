# Muay Thai Kick Form Analysis

## What this is
A personal project to analyze roundhouse kick form from video using pose estimation, score deviation against a self-chosen reference rep, and generate coaching feedback via an LLM. Built as a learning project; not a product.

## Status
Pre-build. Architecture decided, no code written yet. See `docs/architecture.md` and `docs/pipeline.mermaid` for the full pipeline spec.

## Key decisions already made (don't re-litigate without reason)

- **Pose model:** MediaPipe BlazePose (monocular). Chosen for being free, fast, and good enough for a v1 — not OpenPose, not a paid API.
- **v1 scope:** one kick type (roundhouse), one camera angle (side-on), single camera. Multi-angle/3D is a deliberate v2, not v1.
- **Capture target:** kicking a heavy bag, not air-kicking. Decided this is better data, not worse — fixes target distance/height across reps (removes a confound) and gives real impact dynamics. Tradeoff: bag occludes the shin/foot briefly at the moment of contact, so low-visibility frames at that checkpoint should be flagged, not silently smoothed over.
- **Camera placement:** side-on, ~90° to the kicker-bag line, roughly chest height, framed wide enough to keep the bag's swing arc in shot. Rationale: keeps the kick's main rotational motion in the camera's image plane rather than its depth axis, since monocular pose estimation is weakest at depth.
- **Data layering:** medallion-style — bronze (raw landmarks), silver (smoothed/normalized joint angles), gold (deviation scores vs. reference rep). Parquet files, one per rep, no DB yet.
- **Reference rep:** your own cleanest rep, not a coach's or a population average. Alignment via DTW since rep tempo varies.
- **LLM's role:** explicitly scoped to translation/synthesis, not geometry. It takes already-computed deviation reports (and optionally key frame images, multimodal) and turns them into ranked natural-language coaching cues. It does not compute angles or judge raw landmarks itself.
- **Known failure mode to design around:** occlusion when the kicking leg crosses in front of the torso, plus the brief bag occlusion at contact — both degrade landmark confidence and need explicit handling (drop/flag, don't trust blindly).

## Architecture at a glance

```
Capture → Pose Extraction (BlazePose) → Bronze
  → Feature Engineering (angles, smoothing) → Silver
  → DTW Alignment vs Reference → Deviation Scoring → Gold
  → LLM Feedback (Claude API, multimodal) + Visualization → Coaching cues
  → Iterate (re-record)
```

Full stage-by-stage spec, schemas, and folder layout: `docs/architecture.md`.
Visual flowchart: `docs/pipeline.mermaid`.

## Open questions for planning

- Auto-detect kick start/end per rep, or keep manually clipping clips during capture?
- When do we move past local Parquet files to something queryable (DuckDB/SQLite)?
- How much should the LLM feedback layer be grounded with real coaching literature (RAG) vs. relying on deviation-from-reference alone?
- At what point does this need a second camera angle for hip rotation accuracy?
