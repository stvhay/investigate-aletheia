---
name: benchmark-math
description: Run the GVR loop against IMO-ProofBench problems and score results
---

# Benchmark Math

## Overview

Run the solve-math GVR loop against IMO-ProofBench problems, score results using LLM-as-judge grading, and produce a structured results report.

## Input

The user specifies a tier when invoking the skill:

```
/benchmark-math tier1
/benchmark-math tier2
/benchmark-math tier3
```

Or a specific problem ID:

```
/benchmark-math PB-Basic-001
```

## Tier Definitions

| Tier | Selection | Count | Purpose |
|------|-----------|-------|---------|
| tier1 | First 5 problems where Level = `pre-IMO` or `IMO-easy` | 3-5 | Validate loop works |
| tier2 | 5 `pre-IMO`/`IMO-easy` + 5 `IMO-medium` + 5 `IMO-hard` (first of each) | ~15 | Cross-difficulty measurement |
| tier3 | All problems | All | Full comparison to Aletheia |

If a specific Problem ID is given, run only that problem.

## Procedure

### Step 1: Load data

Read `data/proofbench.csv`. Parse it as CSV. Each row has columns: `Problem ID`, `Problem`, `Solution`, `Grading guidelines`, `Category`, `Level`, `Short Answer`, `Source`.

### Step 2: Select problems

Based on the tier argument, filter to the appropriate subset. Record the selected Problem IDs.

### Step 3: Run GVR loop on each problem

For each selected problem, invoke the solve-math skill with the problem text from the `Problem` column. Record:
- The final verdict (`CORRECT`, `WRONG`, `NO SOLUTION FOUND`)
- The number of attempts and revisions used
- The final proof text (if any)
- The Verifier's final assessment

**Run problems sequentially** — one at a time. This avoids overwhelming concurrent sub-agents and makes it easy to track progress.

Report progress to the user after each problem:

```
[3/5] PB-Basic-003 (Algebra, IMO-easy): CORRECT (attempt 1, 0 revisions)
```

### Step 4: Grade each result

For each problem that produced a proof (not "NO SOLUTION FOUND"), spawn a **general-purpose** sub-agent using the Task tool with model `opus`:
- Use the Grader prompt from `.claude/skills/benchmark-math/prompts/grader.md`
- Replace `{PROBLEM}` with the problem text
- Replace `{SOLUTION}` with the reference solution from the CSV
- Replace `{GRADING_GUIDELINES}` with the grading guidelines from the CSV
- Replace `{SHORT_ANSWER}` with the short answer from the CSV (or "N/A" if empty)
- Replace `{CANDIDATE_PROOF}` with the GVR loop's final proof
- The sub-agent's task description should be: "Grade mathematical proof"

Parse the Grader's response to extract:
- `[SCORE: N]` — the 0-7 grade
- `[ANSWER: MATCH/MISMATCH/N/A]` — answer correctness

For problems that returned "NO SOLUTION FOUND," assign score 0 and answer N/A.

### Step 5: Produce results report

Write a report to `results/YYYY-MM-DD-tierN.md` (using today's date and the tier).

The report format:

```markdown
# Benchmark Results: Tier N

**Date:** YYYY-MM-DD
**Model:** opus
**Parameters:** max_attempts=3, max_revisions=2

## Summary

| Metric | Value |
|--------|-------|
| Problems attempted | X / Y |
| Solutions found | Z / Y |
| Mean score (all) | N.N / 7 |
| Mean score (attempted) | N.N / 7 |
| Score 7 (perfect) | N / Y |
| Score >= 5 (near-complete) | N / Y |
| Answer match rate | N / M (where applicable) |

## Comparison to Aletheia

| Metric | Aletheia (published) | Claude GVR |
|--------|---------------------|------------|
| IMO-ProofBench accuracy | 95.1% | TBD% |
| Conditional accuracy | 98.3% | TBD% |

## Per-Problem Results

| Problem ID | Category | Level | Verdict | Attempts | Revisions | Score | Answer |
|------------|----------|-------|---------|----------|-----------|-------|--------|
| PB-Basic-001 | Algebra | IMO-easy | CORRECT | 1 | 0 | 7 | MATCH |
| ... | ... | ... | ... | ... | ... | ... | ... |

## Detailed Results

### PB-Basic-001

**Problem:** [problem text]
**Verdict:** CORRECT (attempt 1, 0 revisions)
**Score:** 7/7
**Answer:** MATCH

**Proof:**
[final proof text]

**Grader Assessment:**
[grader's full analysis]

---

[repeat for each problem]
```

### Step 6: Report to user

Display the Summary and Per-Problem Results tables. Tell the user the full report is saved at the results path.

## Important Notes

- **Sequential execution.** Run one problem at a time to keep things manageable and observable.
- **The Grader is independent.** It does not see the Verifier's assessment — only the problem, reference solution, grading guidelines, and candidate proof.
- **"NO SOLUTION FOUND" is scored as 0.** This is a real data point — it tells us the system's refusal rate.
- **Preserve raw data.** The detailed results section includes full proofs and assessments for post-hoc analysis.
