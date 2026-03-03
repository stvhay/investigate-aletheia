# Design: Switch GVR Loop to Opus Thinking for Tier 1 Re-run

## Goal

Close the accuracy gap with Aletheia (95.1%) by switching from Sonnet to Opus 4.6 with extended thinking. Re-run Tier 1 (5 problems) to get a direct comparison against the Sonnet baseline (60% accuracy, 3/5 solved).

## Hypothesis

The Tier 1 failure mode is **capability, not architecture**. Sonnet's GVR loop has 100% conditional accuracy (all produced proofs are correct) but fails to solve 2/5 problems — both functional equation problems requiring deep case analysis. Opus with extended thinking should have more raw problem-solving power to crack these.

## Changes

### `solve-math/SKILL.md`

- Line 16: `model: sonnet` → `model: opus`
- Lines 43, 57, 74: Change sub-agent model from `sonnet` → `opus` for Generator, Verifier, and Reviser

### `benchmark-math/SKILL.md`

- Line 66: Change Grader sub-agent model from `sonnet` → `opus`
- Report header: Model reads `opus` instead of `sonnet`

### What stays the same

- All prompts (generator.md, verifier.md, reviser.md, grader.md) — unchanged
- Loop parameters (max_attempts=3, max_revisions=2) — unchanged
- GVR loop architecture — unchanged
- Results format — new file: `results/2026-03-03-tier1.md`

## Expected Behavior

Opus 4.6 sub-agents use extended thinking automatically. No explicit `budget_tokens` configuration needed. Sub-agents get more reasoning time on each generation, verification, and revision step.

## Risk

- **Cost:** Opus thinking is substantially more expensive per sub-agent call. 5 problems with up to 15 sub-agent spawns worst-case.
- **Latency:** Full Tier 1 run could take 30+ minutes.

## Success Criteria

- Solve at least 4/5 Tier 1 problems (80%+ accuracy)
- Maintain high conditional accuracy (no false positives)
- Mean score improvement over Sonnet's 3.6/7
