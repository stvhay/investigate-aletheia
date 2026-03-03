# README.md Rewrite Implementation Plan

> **For Claude:** Execute this plan using subagent-driven-development (same session) or executing-plans (separate session / teammate).

**Goal:** Replace the placeholder README with a contributor-focused README that explains the GVR architecture, how to use the two main skills, and how to set up the project.

**Architecture:** Single-file replacement. The new README follows the design in `docs/plans/2026-03-03-readme-design.md`: title, one-paragraph description (citing the Aletheia paper, stating the Opus recreation mission, one-sentence SETI@home vision), GVR architecture diagram, usage, setup, project structure, contributing link, license.

**Tech Stack:** Markdown

**Acceptance Criteria — what must be TRUE when this plan is done:**
- [ ] `README.md` contains all 7 sections from the design doc
- [ ] The Aletheia paper (arXiv:2602.10177) is cited with a hyperlink
- [ ] The mission to recreate results with Opus 4.6 is stated
- [ ] The SETI@home-for-math vision is included as one sentence
- [ ] The GVR architecture diagram is present and accurate
- [ ] `/solve-math` and `/benchmark-math` usage is documented
- [ ] Prerequisites list matches current requirements (Claude Code, direnv, Nix)
- [ ] `docs/FIRST_RUN.md` step is included in setup
- [ ] Project structure tree is present and accurate
- [ ] No benchmark results are included (per design decision)
- [ ] Contributing links to `CONTRIBUTING.md`; License links to `LICENSE`

**Dependencies:** None

---

### Task 1: Rewrite README.md

**Context:** The project is an open-source recreation of Google DeepMind's Aletheia math research agent using Claude Code. The current `README.md` at the project root is a placeholder with just the title, one-line description, prerequisites, clone command, and license. It needs to be replaced with a full contributor-focused README per the approved design.

The project implements a Generator→Verifier→Reviser (GVR) loop as Claude Code skills. The two main skills are:
- `/solve-math` — solves a single math problem through the GVR loop
- `/benchmark-math` — runs the GVR loop against IMO-ProofBench problems

The repo is at `git@github.com:stvhay/investigate-aletheia.git`.

**Files:**
- Modify: `README.md` (full replacement)

**Depends on:** Independent

**Step 1: Write the new README.md**

Replace the entire contents of `README.md` with:

```markdown
# Investigate Aletheia

Open-source recreation of Google DeepMind's [Aletheia](https://arxiv.org/abs/2602.10177) — an autonomous mathematical research agent — using Claude Code. Aletheia uses a Generator→Verifier→Reviser (GVR) loop to produce and self-check mathematical proofs in natural language. This project implements the same architecture with Claude Opus 4.6 and benchmarks it against [IMO-ProofBench](https://github.com/google-deepmind/superhuman/blob/main/imobench/proofbench.csv). See [`docs/aletheia-analysis.md`](docs/aletheia-analysis.md) for the full feasibility analysis.

Long-term goal: distributed mathematical research — anyone with Claude Code can contribute compute to open problems.

## Architecture

The GVR loop runs three independent sub-agents with the same base model (Opus 4.6) and different prompt scaffolding:

```
Problem Statement
       │
       ▼
  ┌──────────┐
  │ Generator │ ── Produces candidate proof in natural language
  └────┬─────┘
       │
       ▼
  ┌──────────┐     [WRONG]   → discard, restart Generator (up to max_attempts)
  │ Verifier  │ ── [FIXABLE] → send to Reviser
  └────┬─────┘     [CORRECT] → accept, output solution
       │
       ▼
  ┌──────────┐
  │  Reviser  │ ── Corrects errors, sends back to Verifier (up to max_revisions)
  └──────────┘
```

Each sub-agent gets fresh context. The Verifier never sees the Generator's reasoning process — only its final output. The system can output "No solution found" rather than hallucinate.

## Usage

### Solve a problem

```
/solve-math Prove that for all positive integers n, the sum 1 + 1/2 + ... + 1/n is not an integer for n >= 2.
```

Outputs a verified proof or "No solution found."

| Parameter | Default | Description |
|-----------|---------|-------------|
| max_attempts | 3 | Full Generator restarts on `[WRONG]` |
| max_revisions | 2 | Revise→Verify cycles per attempt |

### Run benchmarks

```
/benchmark-math tier1
```

Runs the GVR loop against [IMO-ProofBench](https://github.com/google-deepmind/superhuman/blob/main/imobench/proofbench.csv) problems and scores results using LLM-as-judge grading on the 0–7 IMO scale.

| Tier | Problems | Purpose |
|------|----------|---------|
| tier1 | 3–5 easiest | Validate the loop works end-to-end |
| tier2 | ~15 mixed difficulty | Cross-difficulty measurement |
| tier3 | All | Direct comparison to Aletheia |

Results are written to `results/YYYY-MM-DD-tierN.md`.

## Setup

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [direnv](https://direnv.net/)
- [Nix](https://nixos.org/download/) with flakes enabled

### Getting started

```bash
git clone git@github.com:stvhay/investigate-aletheia.git
cd investigate-aletheia
direnv allow
```

Follow [`docs/FIRST_RUN.md`](docs/FIRST_RUN.md) to initialize project memory, then:

```bash
claude
```

## Project Structure

```
math-agi/
├── .claude/skills/
│   ├── solve-math/          # GVR loop skill
│   ├── benchmark-math/      # Benchmark runner skill
│   └── ...                  # Development workflow skills
├── data/
│   └── proofbench.csv       # IMO-ProofBench dataset
├── docs/
│   ├── aletheia-analysis.md # Aletheia feasibility analysis
│   └── plans/               # Design and implementation plans
├── results/                 # Benchmark outputs
└── src/                     # Source code
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE)
```

**Step 2: Verify the file renders correctly**

Run: `head -80 README.md`

Expected: The file starts with `# Investigate Aletheia` and contains all 7 sections (title/description, Architecture, Usage, Setup, Project Structure, Contributing, License).

**Step 3: Commit**

```bash
git add README.md
git commit -m "docs: rewrite README with architecture, usage, and setup"
```
