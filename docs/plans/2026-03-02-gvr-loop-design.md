# GVR Loop & Benchmark Design

## Decisions

- **Orchestration**: Claude Code agents (Task tool), no separate API key
- **Approach**: Skill-based GVR — skill orchestrates 3 sub-agents via Task tool
- **Benchmark scope**: Tiered — 3-5 easy → 10-15 mixed → full suite
- **Scoring**: Known-answer + LLM-as-judge combined
- **Tools**: Pure reasoning first, add tools later

## Architecture

The system is a Claude Code skill (`/solve-math`) that takes a math problem as input and orchestrates three sub-agents:

```
/solve-math "Prove that for all positive integers n, ..."
    │
    ┌─── Loop (max_attempts configurable, default 3) ───┐
    │                                                     │
    │  1. Generator agent (general-purpose)               │
    │     Input: problem statement                        │
    │     Output: candidate proof in natural language      │
    │                                                     │
    │  2. Verifier agent (general-purpose)                │
    │     Input: problem statement + candidate proof       │
    │     Output: [CORRECT] / [WRONG] / [FIXABLE]        │
    │             + detailed critique                      │
    │                                                     │
    │  3. If [FIXABLE]: Reviser agent (general-purpose)   │
    │     Input: problem + proof + critique                │
    │     Output: corrected proof → back to Verifier      │
    │                                                     │
    │  4. If [WRONG]: increment attempt, restart Generator│
    │  5. If [CORRECT]: return solution                   │
    │                                                     │
    │  6. If max_attempts reached: return "No solution"   │
    └─────────────────────────────────────────────────────┘
```

**Key design decisions:**
- Each agent is a fresh context — no shared thinking tokens. Mirrors Aletheia's core insight.
- The Verifier never sees the Generator's reasoning process, only its final output.
- The Reviser gets the critique but not the Generator's internal reasoning.
- "No solution found" is a valid output — admits failure rather than hallucinating.

## Prompt Design

### Generator

Role: mathematician tackling a problem. Given only the problem statement, produce a complete, rigorous proof in natural language. No external tools — pure reasoning.

- Show all steps and justify each logical move
- Use standard mathematical notation (LaTeX where helpful)
- If the problem seems intractable, output "No solution found" rather than guessing
- Aim for the rigor expected in a peer-reviewed journal

### Verifier (adapted from Aletheia's published Appendix A)

Role: expert peer reviewer for a top-tier mathematical journal. Given the problem statement and a candidate proof, produce three sections:

1. **Critique**: Analyze the problem independently, then verify the candidate line-by-line. Actively search for: logical fallacies, unstated assumptions, calculation errors, gaps in rigor, circular reasoning.
2. **Verdict**: Exactly one of `[CORRECT]`, `[WRONG]`, `[FIXABLE]`
3. **Resolution**:
   - `[CORRECT]` → brief justification of why the proof is publication-ready
   - `[WRONG]` → enumerate fatal flaws with specific line references
   - `[FIXABLE]` → list the specific errors that need correction

Critical instruction: do not give the benefit of the doubt. If a step is unclear or hand-wavy, mark it as an error. The bar is publication-level rigor.

### Reviser

Role: mathematical editor. Given the problem, the original proof, and the Verifier's critique, produce a complete corrected proof. Not a patch — a full rewrite incorporating the fixes. The output must stand alone as a self-contained proof.

## Benchmark Design

### Data Source

IMO-ProofBench from DeepMind's public repo (`imobench/proofbench.csv`). Stored locally at `data/proofbench.csv`.

### Tiered Execution

| Tier | Problems | Purpose | Gate to next tier |
|------|----------|---------|-------------------|
| Tier 1 | 3-5 Basic (easiest) | Validate the loop works end-to-end | At least 1 problem gets `[CORRECT]` |
| Tier 2 | 10-15 mixed (Basic + Advanced) | Measure performance across difficulties | Results are stable, no orchestration bugs |
| Tier 3 | Full 60 (all Basic + Advanced) | Direct comparison to Aletheia's published numbers | User decision |

### Scoring (dual signal)

**Signal 1 — Known-answer comparison (objective)**
For problems with known solutions, compare the model's final answer/conclusion to ground truth. Binary: correct conclusion or not.

**Signal 2 — LLM-as-judge (quality)**
A separate Claude agent acts as a grader, scoring each proof on the 0-7 IMO scale:
- 0: No meaningful progress
- 1-2: Some relevant ideas but fundamentally incomplete
- 3-4: Significant progress, key ideas present, gaps remain
- 5-6: Nearly complete, minor issues
- 7: Complete and rigorous

The grader sees: problem statement, ground truth solution (if available), and the model's proof. It does not see the Verifier's assessment — independent evaluation.

### Output

Each benchmark run produces a results file at `results/YYYY-MM-DD-tier-N.md` with:
- Per-problem: problem ID, difficulty, verdict, loop iterations, known-answer match, LLM grade (0-7), final proof
- Summary: overall accuracy, conditional accuracy (on attempted), mean grade, comparison to Aletheia's published numbers

## Configuration

| Parameter | Default | Rationale |
|-----------|---------|-----------|
| max_attempts | 3 | Full Generator restarts on `[WRONG]` |
| max_revisions | 2 | Revise→Verify cycles per attempt before `[WRONG]` |
| model | sonnet | Cost-efficient; switch to opus for final benchmarks |

### Not Building (yet)

- No web search or code execution tools
- No formal verification (Lean/Coq)
- No best-of-N selection
- No persistent state between problems
- No LaTeX rendering or PDF generation

## File Layout

```
math-agi/
├── .claude/skills/
│   ├── solve-math/
│   │   ├── SKILL.md
│   │   └── prompts/
│   │       ├── generator.md
│   │       ├── verifier.md
│   │       └── reviser.md
│   └── benchmark-math/
│       ├── SKILL.md
│       └── prompts/
│           └── grader.md
├── data/
│   └── proofbench.csv
└── results/
```
