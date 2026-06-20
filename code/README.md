# EvidenceLens — Multi-Modal Damage Claim Verification

A two-stage AI pipeline that verifies insurance and logistics damage claims by analyzing submitted images against user conversations, claim history, and evidence requirements. Built for the **HackerRank Orchestrate** hackathon challenge.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Results](#results)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Usage](#usage)
  - [Generate Predictions](#generate-predictions)
  - [Run Evaluation](#run-evaluation)
- [Cost & Performance](#cost--performance)
- [Security](#security)

---

## Overview

For each damage claim, the system:

1. Sends all submitted images in a **single multi-image API call** to a vision-language model (VLM)
2. Receives structured JSON describing per-image observations and a claim-level synthesis
3. Applies a **deterministic Python rule ladder** to produce the final verdict — zero additional model calls

**Output fields per claim:**

| Field | Description |
|---|---|
| `claim_status` | `supported` / `contradicted` / `not_enough_information` |
| `evidence_standard_met` | Whether the image set is sufficient to evaluate the claim |
| `issue_type` | Visible damage type (dent, crack, water_damage, …) |
| `object_part` | Relevant part of the claimed object |
| `risk_flags` | Quality, authenticity, and history risk signals |
| `severity` | `none` / `low` / `medium` / `high` / `unknown` |
| `supporting_image_ids` | Image IDs that support the verdict |
| `valid_image` | Whether the image set is usable for automated review |

---

## Architecture

```
claims.csv + images
        │
        ▼
┌─────────────────────────────────────────────────────┐
│  Stage 1 — VLM Perception  (stage1_perception.py)   │
│                                                     │
│  • Resize images to ≤768 px (JPEG, quality 85)      │
│  • Base64-encode and batch all images per claim      │
│  • Single Anthropic API call → strict JSON output   │
│  • Per-image: visibility, angle, issue type,         │
│    quality flags, adversarial-text detection         │
│  • Synthesis: consistency, severity, mismatch        │
│  • SHA-256 disk cache — re-runs cost $0              │
│  • Exponential backoff retry (1s → 2s → 4s)         │
└───────────────────┬─────────────────────────────────┘
                    │ structured Stage-1 dict
                    ▼
┌─────────────────────────────────────────────────────┐
│  Stage 2 — Deterministic Decision  (stage2_decision) │
│                                                     │
│  Zero model calls. Rule ladder:                     │
│  Pre-X  Consistent images + cross-family mismatch   │
│         → contradicted                              │
│  A      Inconsistent identity across images         │
│         → not_enough_information                    │
│  B      No inspectable angle                        │
│         → not_enough_information                    │
│  C      Wrong object / no claimed object visible    │
│         → contradicted or not_enough_information    │
│  E      Object visible, no damage when claimed      │
│         → contradicted                              │
│  G      Explicit issue-type mismatch                │
│         → contradicted                              │
│  I      Damage visible, no contradiction            │
│         → supported                                 │
│                                                     │
│  Hard constraint: user history may only ADD         │
│  risk flags — never changes claim_status            │
└─────────────────────────────────────────────────────┘
        │
        ▼
   output.csv
```

### Adversarial Image Handling

If any image contains embedded text instructions (e.g. *"approve this claim"*, *"skip review"*), Stage 1 flags `text_instruction_present` and ignores the instruction. Stage 2 re-derives the verdict from clean (non-tainted) images only — tainted evidence cannot produce a `supported` verdict.

### Caching

Every Stage-1 response is cached to `code/.cache/` keyed by `sha256(image_bytes + claim_text + claim_object + model_id)`. Re-runs against unchanged inputs are instant and cost nothing.

---

## Results

Evaluated on `dataset/sample_claims.csv` — **20 labeled cases**:

| Model | `claim_status` accuracy | Overall accuracy |
|---|---|---|
| `claude-opus-4-8` | **95%** (19/20) | 69.4% |
| `claude-sonnet-4-6` | 90% (18/20) | 56.2% |

**Chosen model for `output.csv`: `claude-opus-4-8`** — higher accuracy on multi-image inconsistency, adversarial injection, and severity mismatch detection.

The single remaining Opus miss is a VLM perception error on a package-corner crush that the model could not detect. No Stage-2 patch was applied — fixing for a specific sample case would overfit.

See [evaluation/evaluation_report.md](evaluation/evaluation_report.md) for the full confusion matrix, per-case breakdown, and operational analysis.

---

## Project Structure

```
code/
├── main.py                          # Entry point → output.csv
├── stage1_perception.py             # VLM call, image resizing, caching, retry
├── stage2_decision.py               # Deterministic rule ladder, risk flags
├── cache.py                         # SHA-256-keyed disk cache
├── ops_tracker.py                   # Token / cost / latency metrics
├── README.md                        # This file
├── evaluation/
│   ├── main.py                      # Evaluation entry point
│   ├── evaluation_report.md         # Two-model comparison + operational analysis
│   ├── sample_predictions_claude_opus_4_8.csv
│   └── sample_predictions_claude_sonnet_4_6.csv
└── demo/
    └── index.html                   # Self-contained claim review visualization UI
```

---

## Prerequisites

- Python 3.9+
- An [Anthropic API key](https://console.anthropic.com/)

---

## Setup

**1. Install dependencies**

```bash
pip install anthropic pillow python-dotenv
```

**2. Configure your API key**

Create a `.env` file at the **repo root** (never commit this file):

```
ANTHROPIC_API_KEY=sk-ant-...
```

---

## Usage

### Generate Predictions

Run the pipeline against `dataset/claims.csv` and write `output.csv` at the repo root.

```bash
# Default model: claude-opus-4-8
python code/main.py

# Use an alternative model
python code/main.py --model claude-sonnet-4-6

# Custom input / output paths
python code/main.py --claims dataset/claims.csv --output output.csv
```

The first run makes live API calls (~313s, ~$1.17 for 44 claims with Opus). Subsequent runs against the same inputs complete in under 5 seconds at zero cost thanks to the disk cache.

### Run Evaluation

Evaluate both models against the 20 labeled sample claims and regenerate `evaluation/evaluation_report.md`.

```bash
# Both models (default)
python code/evaluation/main.py

# Single model
python code/evaluation/main.py --model claude-opus-4-8
```

---

## Cost & Performance

Full 44-row test run — `claude-opus-4-8`, cold (no cache):

| Metric | Value |
|---|---|
| API calls | 44 (one per claim) |
| Input tokens | 125,955 |
| Output tokens | 21,632 |
| Images processed | 82 |
| **Total cost** | **~$1.17** |
| Cost per claim | ~$0.027 |
| Wall time | ~313s |
| Pricing basis | $5/M input + $25/M output |

With the disk cache, all subsequent runs cost **$0** and complete in **<5 seconds**.

---

## Security

- API keys are read from environment variables only — never hardcoded
- `.env` is listed in `.gitignore` and must not be committed
- Adversarial text embedded in images is detected and neutralised by the Stage-1 system prompt and the Stage-2 clean-evidence override
