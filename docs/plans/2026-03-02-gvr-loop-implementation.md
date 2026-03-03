# GVR Loop & Benchmark Implementation Plan

> **For Claude:** Execute this plan using subagent-driven-development (same session) or executing-plans (separate session / teammate).

**Goal:** Build an Aletheia-style Generator→Verifier→Reviser loop as Claude Code skills, plus a benchmark runner that evaluates performance against IMO-ProofBench.

**Architecture:** Two Claude Code skills — `solve-math` (GVR orchestration via Task tool sub-agents) and `benchmark-math` (runs solve-math against ProofBench problems, scores results). All prompts are standalone markdown files read by the orchestrating skill and injected into sub-agent prompts.

**Tech Stack:** Claude Code skills (markdown), CSV benchmark data, no external dependencies.

**Acceptance Criteria — what must be TRUE when this plan is done:**
- [ ] `data/proofbench.csv` exists with 105 rows of IMO-ProofBench data
- [ ] `/solve-math` skill exists and can be invoked with a math problem
- [ ] The skill spawns Generator, Verifier, and Reviser as independent sub-agents
- [ ] The loop terminates on `[CORRECT]`, retries on `[WRONG]` (up to 3), and revises on `[FIXABLE]` (up to 2 per attempt)
- [ ] `/benchmark-math` skill exists and can run a tiered benchmark
- [ ] Smoke test: `/solve-math` produces a coherent result on one easy problem

**Dependencies:** None

---

### Task 1: Data Setup [Independent]

**Context:** We need the IMO-ProofBench dataset locally and proper directory structure for benchmark results. The CSV is hosted at `https://raw.githubusercontent.com/google-deepmind/superhuman/main/imobench/proofbench.csv`. It has 105 rows with columns: `Problem ID`, `Problem`, `Solution`, `Grading guidelines`, `Category`, `Level`, `Short Answer`, `Source`. Problem IDs follow the pattern `PB-Basic-NNN` / `PB-Advanced-NNN`. Levels include `pre-IMO`, `IMO-easy`, `IMO-medium`, `IMO-hard`.

**Files:**
- Create: `data/proofbench.csv` (downloaded)
- Create: `results/.gitkeep`
- Modify: `.gitignore` (add `results/` exclusion)

**Step 1: Create directories**

```bash
mkdir -p /Users/hays/Projects/math-agi/data
mkdir -p /Users/hays/Projects/math-agi/results
touch /Users/hays/Projects/math-agi/results/.gitkeep
```

**Step 2: Download ProofBench CSV**

```bash
curl -L -o /Users/hays/Projects/math-agi/data/proofbench.csv \
  "https://raw.githubusercontent.com/google-deepmind/superhuman/main/imobench/proofbench.csv"
```

**Step 3: Verify download**

```bash
head -1 /Users/hays/Projects/math-agi/data/proofbench.csv
wc -l /Users/hays/Projects/math-agi/data/proofbench.csv
```

Expected: First line contains column headers (`Problem ID,Problem,...`). Line count is ~106 (header + 105 data rows).

**Step 4: Add results/ to .gitignore**

Read `/Users/hays/Projects/math-agi/.gitignore` and append:

```
# Benchmark results (generated, not committed)
results/
!results/.gitkeep
```

**Step 5: Commit**

```bash
git add data/proofbench.csv results/.gitkeep .gitignore
git commit -m "data: add IMO-ProofBench dataset and results directory"
```

---

### Task 2: GVR Prompts [Independent]

**Context:** These are the three prompt files that define the Generator, Verifier, and Reviser roles. Each prompt is a standalone markdown file that the orchestrating skill will read and inject into a sub-agent's prompt via the Task tool. The prompts must be self-contained — the sub-agent receives only the prompt text plus the problem/proof data.

The Verifier prompt is adapted from Aletheia's published Appendix A (arXiv:2602.21201). The Generator and Reviser prompts are our design, since DeepMind did not publish theirs.

Key principle from Aletheia: decoupling verification from generation breaks confirmation bias. Each agent gets fresh context with no access to another agent's reasoning process.

**Files:**
- Create: `.claude/skills/solve-math/prompts/generator.md`
- Create: `.claude/skills/solve-math/prompts/verifier.md`
- Create: `.claude/skills/solve-math/prompts/reviser.md`

**Step 1: Create prompt directory**

```bash
mkdir -p /Users/hays/Projects/math-agi/.claude/skills/solve-math/prompts
```

**Step 2: Write Generator prompt**

Create `.claude/skills/solve-math/prompts/generator.md` with the following content:

```markdown
# Generator

You are a research mathematician. Your task is to produce a complete, rigorous proof for the given problem.

## Instructions

- Read the problem statement carefully. Identify what must be proved or determined.
- Produce a complete proof in natural language. Show every step. Justify each logical move.
- Use standard mathematical notation. LaTeX is encouraged for formulas (e.g., `$n^2$`, `$\sum_{i=1}^{n}$`).
- Structure the proof clearly: state what you will show, proceed step by step, conclude explicitly.
- The standard of rigor is that of a peer-reviewed mathematics journal. Every claim must be justified — no hand-waving, no "it is obvious that," no skipped steps.
- If the problem asks you to "find all" or "determine," you must prove both existence and uniqueness/completeness (i.e., show your answer works AND that no other answer works).
- If you cannot solve the problem or are not confident in your solution, output exactly: `NO SOLUTION FOUND`. Do not guess. Do not produce a partial or speculative proof. An honest "no solution" is better than a flawed proof.

## Output Format

If you have a solution, output the proof directly. No preamble, no meta-commentary. Just the proof.

If you cannot solve it, output exactly:

```
NO SOLUTION FOUND
```

## Problem

{PROBLEM}
```

**Step 3: Write Verifier prompt**

Create `.claude/skills/solve-math/prompts/verifier.md` with the following content:

```markdown
# Verifier

You are an expert peer reviewer for a top-tier mathematical journal. Your task is to rigorously evaluate a candidate proof.

## Instructions

You will receive a problem statement and a candidate proof. You must:

1. **Analyze the problem independently.** Before reading the candidate proof, form your own understanding of what the problem requires and what a correct solution would involve.

2. **Verify the candidate proof line-by-line.** Actively search for:
   - Logical fallacies or non-sequiturs
   - Unstated or unjustified assumptions
   - Calculation errors
   - Gaps in rigor (steps that skip justification)
   - Circular reasoning
   - Incorrect use of theorems or lemmas
   - Missing cases (e.g., "find all" problems that don't prove completeness)
   - Specification gaming (answering an easier question than what was asked)

3. **Do not give the benefit of the doubt.** If a step is unclear, hand-wavy, or lacks justification, treat it as an error. The bar is publication-level rigor.

## Output Format

Your response must contain exactly three sections:

### Critique

Your independent analysis of the problem followed by a line-by-line evaluation of the candidate proof. Be specific — reference exact steps or claims in the proof.

### Verdict

State exactly one of:

- `[CORRECT]` — The proof is flawless, completely rigorous, and requires no changes.
- `[WRONG]` — The proof is fundamentally flawed. The core approach is invalid, or there are fatal logical errors that cannot be salvaged with minor fixes.
- `[FIXABLE]` — The proof has the right core approach but contains errors that can be corrected. The logical structure is sound but specific steps need fixing.

### Resolution

Based on your verdict:

- **If `[CORRECT]`:** Briefly justify why the proof meets the standard for publication. Identify the key insights that make it work.
- **If `[WRONG]`:** Enumerate every fatal flaw. Be specific — explain exactly what is wrong and why it cannot be fixed within the current approach.
- **If `[FIXABLE]`:** List every error that must be corrected. Be specific — for each error, state what is wrong and what the correct version should be. Do NOT write the corrected proof yourself.

## Problem

{PROBLEM}

## Candidate Proof

{PROOF}
```

**Step 4: Write Reviser prompt**

Create `.claude/skills/solve-math/prompts/reviser.md` with the following content:

```markdown
# Reviser

You are a mathematical editor. Your task is to produce a corrected, complete proof that addresses all issues identified by the reviewer.

## Instructions

- You will receive: the problem statement, a candidate proof, and a reviewer's critique identifying specific errors.
- Produce a **complete, self-contained proof** — not a patch or a list of fixes. The output must stand alone as a finished proof that someone could read without seeing the original.
- Address every error identified in the critique. Do not skip any.
- Maintain the same overall approach as the original proof unless the critique indicates the approach itself is flawed.
- The standard of rigor is that of a peer-reviewed mathematics journal. Every claim must be justified.
- If, after considering the critique, you believe the problem cannot be solved with the current approach, output exactly: `NO SOLUTION FOUND`.

## Output Format

Output the complete corrected proof directly. No preamble, no meta-commentary about what you changed. Just the proof.

If the approach is unsalvageable, output exactly:

```
NO SOLUTION FOUND
```

## Problem

{PROBLEM}

## Original Proof

{PROOF}

## Reviewer Critique

{CRITIQUE}
```

**Step 5: Commit**

```bash
git add .claude/skills/solve-math/prompts/
git commit -m "feat: add Generator, Verifier, and Reviser prompts for GVR loop"
```

---

### Task 3: solve-math Skill [Depends on: Task 2]

**Context:** This is the orchestration skill that implements the GVR loop. When a user invokes `/solve-math`, this skill reads the prompt files from `.claude/skills/solve-math/prompts/`, then spawns sub-agents via the Task tool in a loop: Generator → Verifier → (Reviser if FIXABLE) → repeat.

The skill is a SKILL.md file — it contains instructions that Claude Code follows when the skill is invoked. The actual orchestration happens through Claude Code's Task tool (spawning general-purpose sub-agents).

**Key parameters:**
- `max_attempts = 3` — full Generator restarts on `[WRONG]`
- `max_revisions = 2` — Revise→Verify cycles per attempt before declaring `[WRONG]`
- Default model: `sonnet` for all sub-agents

The skill must parse the Verifier's output to extract the verdict tag (`[CORRECT]`, `[WRONG]`, or `[FIXABLE]`), then branch accordingly. It reports the final result to the user: the accepted proof, or "No solution found" with a summary of what happened.

**Files:**
- Create: `.claude/skills/solve-math/SKILL.md`

**Step 1: Write the skill file**

Create `.claude/skills/solve-math/SKILL.md` with the following content:

````markdown
---
name: solve-math
description: Solve a math problem using a Generator→Verifier→Reviser loop (Aletheia-style)
---

# Solve Math (GVR Loop)

## Overview

Solve a mathematical proof problem using an iterative Generate→Verify→Revise loop. Each step runs in an independent sub-agent with fresh context, mirroring DeepMind Aletheia's architecture.

## Parameters

- **max_attempts**: 3 (full Generator restarts on `[WRONG]`)
- **max_revisions**: 2 (Revise→Verify cycles per attempt before escalating to `[WRONG]`)
- **model**: sonnet (override with opus for harder problems)

## Input

The user provides a math problem as the argument to `/solve-math`. Example:

```
/solve-math Prove that for all positive integers n, the sum 1 + 1/2 + ... + 1/n is not an integer for n >= 2.
```

## Procedure

### Step 1: Read prompts

Read the following files:
- `.claude/skills/solve-math/prompts/generator.md`
- `.claude/skills/solve-math/prompts/verifier.md`
- `.claude/skills/solve-math/prompts/reviser.md`

### Step 2: Run the GVR loop

Set `attempt = 1`.

**Outer loop** (up to `max_attempts`):

#### 2a. Generate

Spawn a **general-purpose** sub-agent using the Task tool with model `sonnet`:
- Use the Generator prompt from `generator.md`
- Replace `{PROBLEM}` with the user's problem statement
- The sub-agent's task description should be: "Generate mathematical proof"
- Capture the sub-agent's full response as `candidate_proof`

If the Generator returns `NO SOLUTION FOUND`, increment `attempt` and restart the outer loop. If all attempts exhausted, go to Step 3 (failure).

#### 2b. Verify

Set `revision = 0`.

**Inner loop** (up to `max_revisions + 1` verification passes — the first verification is for the Generator's output, subsequent ones are for revised proofs):

Spawn a **general-purpose** sub-agent using the Task tool with model `sonnet`:
- Use the Verifier prompt from `verifier.md`
- Replace `{PROBLEM}` with the user's problem statement
- Replace `{PROOF}` with the current `candidate_proof`
- The sub-agent's task description should be: "Verify mathematical proof"
- Capture the sub-agent's full response as `verification`

Parse the verdict from the Verifier's response. Look for exactly one of: `[CORRECT]`, `[WRONG]`, `[FIXABLE]`.

**If `[CORRECT]`:** Go to Step 3 (success). The current `candidate_proof` is the final answer.

**If `[WRONG]`:** Break the inner loop. Increment `attempt` and restart the outer loop with a fresh Generator call.

**If `[FIXABLE]`:** If `revision < max_revisions`, go to 2c (Revise). Otherwise, treat as `[WRONG]` — break inner loop, increment `attempt`, restart outer loop.

#### 2c. Revise

Spawn a **general-purpose** sub-agent using the Task tool with model `sonnet`:
- Use the Reviser prompt from `reviser.md`
- Replace `{PROBLEM}` with the user's problem statement
- Replace `{PROOF}` with the current `candidate_proof`
- Replace `{CRITIQUE}` with the Verifier's full response (`verification`)
- The sub-agent's task description should be: "Revise mathematical proof"
- Capture the sub-agent's full response as the new `candidate_proof`

If the Reviser returns `NO SOLUTION FOUND`, treat as `[WRONG]` — break inner loop, increment `attempt`, restart outer loop.

Increment `revision`. Return to 2b (Verify the revised proof).

### Step 3: Report result

**On success (`[CORRECT]`):**

Report to the user:
```
## Result: CORRECT

**Attempts:** {attempt} | **Revisions in final attempt:** {revision}

### Proof

{candidate_proof}

### Verifier Assessment

{verification}
```

**On failure (all attempts exhausted):**

Report to the user:
```
## Result: NO SOLUTION FOUND

**Attempts exhausted:** {max_attempts} attempts, each with up to {max_revisions} revisions.

The GVR loop was unable to produce a proof that passed verification.

### Last Verifier Feedback

{verification}
```

## Important Notes

- **Do not edit or improve proofs yourself.** The orchestrator only routes data between sub-agents. All mathematical work happens inside the Generator, Verifier, and Reviser.
- **Each sub-agent gets fresh context.** Never pass one agent's internal reasoning to another. The Verifier sees only the final proof, not the Generator's thought process.
- **Parse verdicts carefully.** Look for the literal strings `[CORRECT]`, `[WRONG]`, `[FIXABLE]` in the Verifier's output. If none is found, treat as `[WRONG]` and log a warning.
- **Respect "No solution found."** This is a valid outcome. Do not retry beyond max_attempts or override the system's judgment.
````

**Step 2: Commit**

```bash
git add .claude/skills/solve-math/SKILL.md
git commit -m "feat: add solve-math skill with GVR loop orchestration"
```

---

### Task 4: Benchmark Skill [Depends on: Task 1, Task 3]

**Context:** The benchmark skill runs `/solve-math` against problems from `data/proofbench.csv` and scores the results using dual signals: known-answer comparison and LLM-as-judge grading. It supports tiered execution (Tier 1: 3-5 easy, Tier 2: 10-15 mixed, Tier 3: full 60).

The CSV has columns: `Problem ID`, `Problem`, `Solution`, `Grading guidelines`, `Category`, `Level`, `Short Answer`, `Source`. Problem IDs follow pattern `PB-Basic-NNN` / `PB-Advanced-NNN`. Levels: `pre-IMO`, `IMO-easy`, `IMO-medium`, `IMO-hard`.

The benchmark runner is itself a Claude Code skill. When invoked, it reads the CSV, selects problems for the chosen tier, runs solve-math on each, then spawns a Grader sub-agent to score each result.

**Files:**
- Create: `.claude/skills/benchmark-math/SKILL.md`
- Create: `.claude/skills/benchmark-math/prompts/grader.md`

**Step 1: Create directory**

```bash
mkdir -p /Users/hays/Projects/math-agi/.claude/skills/benchmark-math/prompts
```

**Step 2: Write Grader prompt**

Create `.claude/skills/benchmark-math/prompts/grader.md` with the following content:

```markdown
# Grader

You are an expert mathematical examiner grading competition proofs on the 0–7 IMO scale. Your evaluation must be independent and rigorous.

## Instructions

You will receive:
- A problem statement
- A reference solution (the known correct approach)
- Grading guidelines (partial credit rubric)
- A candidate proof to evaluate

Score the candidate proof on the standard 0–7 IMO scale:

| Score | Meaning |
|-------|---------|
| 0 | No meaningful progress. Restated the problem or wrote irrelevant content. |
| 1 | A relevant observation or isolated useful idea, but no real progress toward a solution. |
| 2 | Some relevant ideas and partial progress, but fundamentally incomplete. Key steps missing or wrong. |
| 3 | Significant progress. The main idea is present but there are substantive gaps or errors in execution. |
| 4 | Substantial progress. Most of the proof is correct but there is at least one non-trivial gap or error. |
| 5 | Nearly complete. The proof structure is correct with only minor gaps or imprecisions that could be fixed. |
| 6 | Complete proof with minor cosmetic issues (notation inconsistency, redundant steps). Mathematically sound. |
| 7 | Perfect. Complete, rigorous, well-structured. No errors, no gaps, publication-ready. |

## Evaluation Process

1. Read the problem and reference solution to fully understand what a correct proof requires.
2. Read the grading guidelines to understand what constitutes partial credit.
3. Read the candidate proof independently.
4. Compare the candidate's approach and conclusions to the reference.
5. Identify every error, gap, or weakness.
6. Assign a score based on the rubric above.

## Output Format

### Analysis

Your detailed evaluation. Reference specific parts of the candidate proof. Compare to the reference solution where relevant.

### Score

`[SCORE: N]` where N is 0–7.

### Answer Check

If a short answer is provided below, state whether the candidate's final answer matches:
`[ANSWER: MATCH]` or `[ANSWER: MISMATCH]` or `[ANSWER: N/A]` (if no short answer exists for this problem).

## Problem

{PROBLEM}

## Reference Solution

{SOLUTION}

## Grading Guidelines

{GRADING_GUIDELINES}

## Short Answer (if applicable)

{SHORT_ANSWER}

## Candidate Proof to Evaluate

{CANDIDATE_PROOF}
```

**Step 3: Write benchmark skill**

Create `.claude/skills/benchmark-math/SKILL.md` with the following content:

````markdown
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

For each problem that produced a proof (not "NO SOLUTION FOUND"), spawn a **general-purpose** sub-agent using the Task tool with model `sonnet`:
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
**Model:** sonnet
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
````

**Step 4: Commit**

```bash
git add .claude/skills/benchmark-math/
git commit -m "feat: add benchmark-math skill with LLM grader"
```

---

### Task 5: Smoke Test [Depends on: Task 1, Task 3]

**Context:** Validate the entire system works end-to-end by running `/solve-math` on one easy problem from the ProofBench dataset. This is not a code test — it's a manual invocation to confirm the skill orchestration, sub-agent spawning, verdict parsing, and result reporting all function correctly.

The easiest problems in the dataset are Level `pre-IMO`. Pick the first `pre-IMO` problem from the CSV (likely `PB-Basic-002` based on the sample data — "Show that $x^2 + y^2 + z^2 + t^2 \ge xyzt$ for any positive real numbers," Level `pre-IMO`, Category `Algebra`).

**Files:** None — this is a manual test.

**Step 1: Identify a test problem**

Read `data/proofbench.csv` and find the first problem where Level = `pre-IMO`. Note its Problem ID and Problem text.

**Step 2: Run solve-math**

Invoke: `/solve-math {problem text from Step 1}`

**Step 3: Verify behavior**

Confirm:
- [ ] Generator sub-agent was spawned and returned a proof (or NO SOLUTION FOUND)
- [ ] Verifier sub-agent was spawned and returned a verdict with `[CORRECT]`, `[WRONG]`, or `[FIXABLE]`
- [ ] If `[FIXABLE]`, Reviser was spawned and produced a corrected proof
- [ ] If `[WRONG]`, Generator was retried
- [ ] Final result was reported to the user in the expected format
- [ ] The loop terminated (did not hang or infinite-loop)

**Step 4: Report results**

Tell the user:
- Which problem was tested
- What verdict the Verifier gave
- How many attempts/revisions were needed
- Whether the system behaved as expected
- Any issues or adjustments needed

Do NOT commit anything from this task — it's a validation step.
