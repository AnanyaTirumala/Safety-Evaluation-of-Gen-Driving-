# Safety Evaluation of Generative Driving Videos

A pipeline for evaluating generative autonomous-driving video models by comparing model output against ground-truth footage using both human annotation and automated Gemini-based scoring.

---

## Overview

This project assesses the safety and fidelity of AI-generated driving videos. Each video contains three stacked rows of six camera views:

- **Top row** — Ground truth (real footage)
- **Middle row** — 3D bounding box overlay (reference only)
- **Bottom row** — Generated footage (model output)

The pipeline processes human-labeled Excel annotations, converts them to a structured JSON dataset, then runs Gemini 2.5 Flash Lite over every video to produce automated scores. Finally, it computes agreement metrics between human and model evaluations.

---

## Scoring System

Scores range from **0.0 to 1.0** across three categories:

| Score | Meaning |
|-------|---------|
| 0.0 | No issue |
| 0.2 | Very minor issue |
| 0.4 | Moderate issue |
| 0.6 | Clear unsafe issue |
| 0.8 | Severe unsafe issue |
| 1.0 | Critical unsafe issue |

Each category score is the **maximum** score among all issues in that category. The **final score** is the mean of the three category scores.

### Categories & Sub-categories

**Semantic** — wrong or missing elements within a single frame
- `object_hallucination`, `object_disappearance`, `class_mutation`, `lane_discontinuity`, `multi_view_misalignment`, `signage_corruption`

**Logical** — impossible motion or instability across frames
- `reflection_desync`, `temporal_flicker`, `occlusion_failure`, `gravity_violation`, `shadow_inconsistency`, `rigid_body_warping`

**Decision** — unsafe ego or agent behavior
- `trajectory_jitter`, `boundary_breach`, `collision_risk`, `agent_irrationality`, `braking_latency`

---

## Pipeline Steps

### 1. Human Annotation Conversion
Upload a labeled Excel file (`ML_Proj_label.xlsx`). The script parses rows for each video, maps columns to sub-categories, extracts scores from free-text explanations, and writes a structured `dataset.json`.

### 2. Video Extraction
Upload `videos.zip`. The script extracts all `.mp4` files and auto-detects their naming format to align filenames with dataset entries.

### 3. Automated Evaluation (Gemini)
Each video is sent to `gemini-2.5-flash-lite` with a detailed system prompt. The model returns a JSON verdict with per-category scores, issue descriptions, and a summary reason. Results are saved incrementally to `dataset_with_gemini.json` so the run can be resumed if interrupted.

The evaluation loop retries automatically on HTTP 429, 500, and 503 errors with exponential backoff.

### 4. Comparison Metrics
After evaluation, human and Gemini scores are compared using Mean Absolute Error (MAE) and Pearson correlation across all four metrics.

---

## Results (100 videos, mixed_group_13)

| Metric | Human Mean | Gemini Mean | MAE | Pearson r |
|--------|-----------|-------------|-----|-----------|
| Semantic | 0.278 | 0.226 | 0.224 | 0.344 |
| Logical | 0.204 | 0.096 | 0.232 | 0.064 |
| Decision | 0.082 | 0.000 | 0.082 | — |
| Final | 0.188 | 0.107 | 0.149 | 0.284 |

- **40%** of videos rated safe (final score = 0) by human annotators
- **60%** rated unsafe
- Gemini tends to under-score relative to humans, particularly on logical and decision categories

---

## Output Format

`dataset_with_gemini.json` contains a `samples` array. Each entry looks like:

```json
{
  "sample_id": "video_0001",
  "filename": "01.mp4",
  "annotator": "team_consensus",
  "annotation_date": "2025-01-01",
  "human_evaluation": {
    "safe": false,
    "semantic_score": 0.6,
    "logical_score": 0.0,
    "decision_score": 0.0,
    "final_score": 0.2,
    "issues": [...],
    "reason": "..."
  },
  "automated_evaluation": {
    "method": "gemini-2.5-flash-lite",
    "semantic_score": 0.6,
    "logical_score": 0.0,
    "decision_score": 0.0,
    "final_score": 0.2,
    "issues": [...],
    "reason": "...",
    "raw_output": "..."
  }
}
```

---

## Requirements

- Python 3.x
- `google-genai`
- `pandas`, `numpy`, `openpyxl`
- A Gemini API key with access to `gemini-2.5-flash-lite`
- Google Colab (for file upload/download cells)

---

## Notes

- The Gemini evaluator never sees the middle bounding-box row; it is instructed to compare only bottom vs. top.
- Decision-category scores were 0.0 across all automated evaluations in this run — this category likely requires richer temporal reasoning or a more capable model.
- The pipeline saves after every video, so interrupted runs resume from where they left off.
