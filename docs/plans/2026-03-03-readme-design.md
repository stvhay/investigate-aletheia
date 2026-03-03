# Design: README.md Rewrite

## Decisions

- **Audience**: Contributors (people who will fork, run, and extend)
- **Aletheia context**: One paragraph with paper reference and mission statement; link to `docs/aletheia-analysis.md` for depth
- **Results**: Not included — sample size too small to claim anything yet
- **Workflow/skills**: Not in README — lives in `CONTRIBUTING.md`
- **Structure**: Lean and technical — Approach A

## Section Plan

### 1. Title + Description

One paragraph:
- What Aletheia is (cite [arXiv:2602.10177](https://arxiv.org/abs/2602.10177))
- What this project does: recreate Aletheia's GVR architecture using Claude Code with Opus 4.6
- Mission: reproduce Aletheia's IMO-ProofBench results with Claude
- Link to `docs/aletheia-analysis.md` for the full feasibility analysis
- One-sentence vision: long-term goal is distributed mathematical research — anyone with Claude Code can contribute compute to open problems

### 2. Architecture

ASCII GVR diagram showing:
- Problem → Generator → Verifier → branch on verdict
- `[CORRECT]` → accept
- `[FIXABLE]` → Reviser → back to Verifier
- `[WRONG]` → restart Generator (up to max_attempts)
- Note: all three agents are the same model (Opus 4.6) with different prompt scaffolding

### 3. Usage

Two subsections:

**Solve a problem** — `/solve-math "problem statement"`
- Describe output: either a verified proof or "No solution found"
- Parameters: max_attempts (default 3), max_revisions (default 2)

**Run benchmarks** — `/benchmark-math`
- Describe tiers, data source (IMO-ProofBench), output location (`results/`)

### 4. Prerequisites & Setup

- Claude Code
- direnv
- Nix with flakes
- Clone, `direnv allow`, follow `docs/FIRST_RUN.md`, then `claude`

### 5. Project Structure

```
math-agi/
├── .claude/skills/
│   ├── solve-math/          # GVR loop skill
│   ├── benchmark-math/      # Benchmark runner skill
│   └── ...                  # Development workflow skills
├── data/
│   └── proofbench.csv       # IMO-ProofBench dataset
├── docs/
│   ├── aletheia-analysis.md # Deep dive on Aletheia
│   └── plans/               # Design and implementation plans
├── results/                 # Benchmark outputs
└── src/                     # (future) source code
```

### 6. Contributing

One line: "See [CONTRIBUTING.md](CONTRIBUTING.md)."

### 7. License

"[MIT](LICENSE)"
