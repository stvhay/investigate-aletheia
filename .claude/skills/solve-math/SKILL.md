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
