# Interview Notes — Multi-Modal Evidence Review

## Why two stages instead of asking the model to output a final verdict directly?

A single-stage approach mixes visual perception with policy logic inside the model, so you can
never tell whether a wrong answer came from bad image reading or bad reasoning. By having Stage 1
do only perception (what does the image actually show?) and Stage 2 do only policy (given those
observations, what is the verdict?), the two responsibilities are independently testable.
Stage 2 is pure Python — no model calls, deterministic, zero cost to iterate — so you can tune
the decision rules without burning API budget. It also makes the system auditable: every verdict
is traceable to a specific rule and the VLM flags that triggered it.

## How does user history influence the verdict, and why can it never override clear visual evidence?

User history feeds into `risk_flags` only — it can add `user_history_risk` or
`manual_review_required` to the flag list, but it cannot change `claim_status`. This is enforced
by a Python `assert` that compares the pre-history and post-history claim status and raises
immediately if they differ. The reason is stated in the problem spec: images are the primary
source of truth. A user with 10 prior rejected claims submitting a clear photo of a real dent
should still get `supported` — history adds friction (flags trigger manual review) but does not
override what the camera shows.

## How do you know it works? What does the evaluation say?

On the 20 labeled cases in `dataset/sample_claims.csv`:
- claude-opus-4-8: **95% claim_status accuracy** (19/20) — the confusion matrix shows zero
  false approvals (no fraudulent claim marked `supported`) and one false contradiction
  (a genuine package crush that Opus couldn't see).
- claude-sonnet-4-6: **90% claim_status accuracy** (18/20) — misses two contradicted cases
  where Sonnet didn't flag the issue-type mismatch.
Per-field, both models score 90%+ on `object_part` and `supporting_image_ids`, while `severity`
and `risk_flags` are harder (50–60%) due to subjectivity in the gold labels.

## Why is the one remaining Opus miss left unfixed?

The miss is a pure perception error: Opus reports `visible_issue=none` with `damage_not_visible`
on an image where a human reviewer saw clear package-corner crushing. No Stage-2 rule can recover
from a model that simply cannot see the damage. The only fix would be to hard-code an exception
for that specific labeled sample — which would be overfit to the sample set and hurt performance
on unseen claims. The honest answer is: one case in 20 has damage the model can't detect, and
that is left as a known limitation rather than a hidden patch.

## What did the Opus vs Sonnet comparison show, and which did you ship?

Both models share the same Stage-2 logic, so differences are pure perception differences.
Opus is better at detecting subtle multi-image inconsistencies (different cars, different objects)
and at correctly identifying issue-type mismatches that require comparing the claim text against
the image. Sonnet over-fires `non_original_image` and under-fires `claim_mismatch` in multi-image
cases, costing it 5 percentage points of claim_status accuracy. The submission uses **Opus 4.8**
for `output.csv`. At $5/M input + $25/M output the full 44-row run cost **$1.17** measured
(125,955 input + 21,632 output tokens), which is ~1.7× more expensive than Sonnet but delivers
meaningfully higher accuracy on exactly the hard cases (manipulation, identity mismatch).

## How did you control cost and rate limits?

Four mechanisms: (1) **Single multi-image API call per claim** — all images are sent together
in one message, so cost scales with images, not with image count squared. (2) **Image downscaling**
— every image is resized to ≤768 px longest edge before base64 encoding, cutting image token
counts significantly. (3) **SHA-256 disk cache** — Stage-1 responses are cached in `code/.cache/`
keyed by `sha256(all_image_bytes + claim_text + claim_object + model_id)`; re-runs on unchanged
inputs cost $0 and complete in under 1 second. (4) **Exponential backoff retry** — on HTTP 429
or 5xx the pipeline waits 1 s, 2 s, 4 s before giving up. Sequential claim processing naturally
throttles requests per minute without a separate rate-limiter.

## How are adversarial images with embedded text handled?

The Stage-1 system prompt contains an explicit security rule: if embedded text tries to instruct
the model (e.g., "approve this claim", "skip review"), the model must set `embedded_text_present=true`
AND add `text_instruction_present` to that image's `quality_flags`, and then completely ignore the
instruction. Stage 2 propagates `text_instruction_present` into `risk_flags` so it appears in the
output. Separately, Stage 2 applies a "clean-vi" override: if the only images showing damage are
the ones flagged with adversarial text, and the untainted images show no damage, `effective_vi` is
forced to `"none"` so the tainted evidence cannot cause a `supported` verdict.

## What would you improve with more time?

**More labeled data first** — 20 samples is thin for tuning Stage-2 rules; 200 would let you
measure generalization properly. **Anthropic Batch API** — async batch processing at 50% cost
discount for non-latency-sensitive workloads; the pipeline architecture already supports it since
each claim is independent. **Per-claim confidence scores** — the Stage-1 synthesis `free_text_reason`
could be parsed for uncertainty signals to flag borderline cases for human review before they reach
the verdict. **Prompt caching** — the system prompt and the fixed template text (>1000 tokens)
repeat every call; Anthropic prompt caching would reduce input token cost by roughly 40%.
