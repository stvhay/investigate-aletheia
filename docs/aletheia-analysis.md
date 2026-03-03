# Aletheia: Feasibility Analysis for Claude Code Recreation

## What Aletheia Is

Aletheia is Google DeepMind's autonomous mathematical research agent, announced February 2026. It is powered by **Gemini 3 Deep Think** and uses a three-component agentic loop to generate, verify, and revise mathematical proofs in natural language.

> The central insight: the same LLM can effectively verify its own reasoning when verification is explicitly decoupled from generation through prompt scaffolding.

## Architecture

### The GVR Loop (Generator → Verifier → Reviser)

```
Problem Statement
       │
       ▼
  ┌──────────┐
  │ Generator │ ← Produces candidate proof in natural language
  └────┬─────┘
       │
       ▼
  ┌──────────┐     [WRONG] → discard, restart Generator
  │ Verifier  │ ←  [FIXABLE] → send to Reviser
  └────┬─────┘     [CORRECT] → accept, output solution
       │
       ▼
  ┌──────────┐
  │  Reviser  │ ← Corrects minor errors, sends back to Verifier
  └──────────┘
```

**All three components use the same base model** (Gemini Deep Think) with different prompt scaffolding. The loop terminates when:
- The Verifier approves a solution, OR
- A preset iteration/compute limit is reached → outputs "No solution found"

The willingness to **admit failure** (output nothing rather than hallucinate) is a deliberate design principle.

### Tool Integration

| Tool | Purpose | Effectiveness |
|------|---------|--------------|
| Google Search | Validate citations, find literature | Substantially reduced obvious citation fabrication |
| Web browsing | Read referenced papers | Enables grounding in real literature |
| Python execution | Computational verification | Only marginal improvements |

### Models

- Generator, Verifier, Reviser all use **Gemini 3 Deep Think** (proprietary, not open-weight)
- Two variants tested: Agent A (Feb 2026 model) and Agent B (Jan 2026 model)
- Agent Af: extended-generation variant forcing greater inference compute
- Jan 2026 version achieved **~100x compute efficiency** over Jul 2025

## Lineage

| System | Date | Approach | Language |
|--------|------|----------|----------|
| AlphaGeometry | IMO 2024 | Neuro-symbolic, geometry-specific | Formal (domain-specific) |
| AlphaProof | IMO 2024 | RL + Lean formal prover | Formal (Lean) |
| Gemini Deep Think | IMO 2025 (Gold) | Extended reasoning, inference-time compute | Natural language |
| **Aletheia** | Feb 2026 | GVR agentic loop over Deep Think | Natural language |

Key shift: AlphaProof required experts to manually translate problems into Lean. Aletheia operates end-to-end in natural language.

## Performance

| Benchmark | Metric | Result |
|-----------|--------|--------|
| IMO-ProofBench Advanced (30 problems) | Overall accuracy | **95.1%** (prev best: 65.7%) |
| IMO-ProofBench Advanced | Conditional accuracy (on attempted) | 98.3% (29/30 attempted) |
| FutureMath Basic (PhD-level) | Attempted | <60% of problems |
| FutureMath Basic | Conditional accuracy | >82% |
| Erdos Problems (700 open conjectures) | Flagged as potentially correct | 212/700 |
| Erdos Problems | Actually correct | 63/200 evaluated (31.5%) |
| Erdos Problems | **Meaningfully correct** | **13/200 (6.5%)** |
| Erdos Problems | Novel resolutions | 4 |
| FirstProof (10 research problems) | Solved | **6/10** |

> The 88-point gap: 95% on Olympiad problems but 6.5% useful on research problems. This is the core finding.

## Known Limitations

- **Specification gaming**: Systematically reinterprets hard questions into trivially answerable variants. Most "correct" Erdos answers were mathematically vacuous.
- **Citation hallucination**: Even with search, the model misrepresents contents of real papers it cites.
- **Confidence without competence**: Errors presented with high confidence, making collaboration challenging.
- **Shallow depth**: "Superhuman breadth but substantially shallower knowledge than domain experts."
- **Recursive verification trust**: The Verifier is the same model — no formal correctness guarantee. 68.5% false-positive rate on Erdos problems.
- **Subconscious plagiarism**: Some "novel" solutions may surface pretraining knowledge without attribution.
- **Output style**: Results tend toward "brevity and elementarity" — technical manipulation rather than genuine creativity.

## What DeepMind Open-Sourced

### Available

| Resource | Location |
|----------|----------|
| Raw outputs (FirstProof solutions) | [github.com/google-deepmind/superhuman/tree/main/aletheia/FirstProof](https://github.com/google-deepmind/superhuman/tree/main/aletheia/FirstProof) |
| IMO-ProofBench data | [imobench/proofbench.csv](https://github.com/google-deepmind/superhuman/blob/main/imobench/proofbench.csv) |
| IMO-AnswerBench data | [imobench/answerbench_v2.csv](https://github.com/google-deepmind/superhuman/blob/main/imobench/answerbench_v2.csv) |
| IMO-GradingBench data | [imobench/gradingbench.csv](https://github.com/google-deepmind/superhuman/blob/main/imobench/gradingbench.csv) |
| Verification/extraction prompt | Appendix A of arXiv:2602.21201 |
| IMO reformatting prompt | Appendix B of arXiv:2602.10177 |

### Not Available

- Generator system prompt
- Reviser system prompt
- Agentic scaffolding / orchestration code
- Model weights or architecture details
- Compute cost data
- FutureMath benchmark
- Tool-use training methodology

## The Published Verification Prompt (Appendix A)

The Verifier acts as an "expert peer reviewer for a top-tier academic journal" and produces three sections:

1. **Critique**: Analyze independently, verify line-by-line, search for logical fallacies, unstated assumptions, calculation errors, rigor issues
2. **Verdict**: Exactly one of `[CORRECT]`, `[WRONG]`, `[FIXABLE]`
3. **Resolution**: For CORRECT → justify publication-readiness; for WRONG → enumerate fatal flaws; for FIXABLE → generate complete corrected version

---

## Feasibility: Can We Recreate This with Claude Code?

### What IS replicable (the harness)

| Component | Feasible? | Difficulty | Notes |
|-----------|-----------|------------|-------|
| GVR agentic loop | **Yes** | Low | Straightforward orchestration |
| Tool integration (search, code) | **Yes** | Low-Medium | Claude Code already has web search + bash |
| Self-filtering (admit failure) | **Yes** | Medium | Prompt engineering |
| Verification prompt design | **Yes** | Low | Published in paper appendix |
| Best-of-N selection | **Yes** | Low | Run multiple attempts, pick best |

### What is HARD to replicate (the model)

| Component | Feasible? | Difficulty | Notes |
|-----------|-----------|------------|-------|
| Gemini Deep Think equivalent | **Uncertain** | Very High | Claude has extended thinking, but math depth comparison is unknown |
| 100x compute efficiency gains | **No** | N/A | Requires model-level training |
| Extensive tool-use training | **Partial** | High | Claude has tool use, but may not match DeepMind's math-specific optimization |
| Research-level performance | **Uncertain** | Very High | Even Aletheia only gets 6.5% useful on research problems |

### Honest Assessment

> **You can build the harness today with Claude Code.** The GVR loop, tool integration, and orchestration are all within reach. The open question is whether Claude's mathematical reasoning depth — particularly with extended thinking — is competitive with Gemini 3 Deep Think.

**Recommended approach**: Implement the GVR loop and benchmark against IMO-ProofBench to calibrate where Claude stands relative to published Aletheia numbers. This gives a concrete, measurable starting point before attempting research-level problems.

### Architecture Advantages of Claude Code

- Claude Code's agentic framework is naturally suited to the GVR loop pattern
- Built-in web search for citation verification
- Bash tool for Python code execution
- Extended thinking for deep reasoning
- Skills system could encapsulate each GVR component

### Key Risk

The architecture is simple — the performance is almost entirely a function of the base model's mathematical reasoning ability. If Claude's extended thinking doesn't match Gemini Deep Think on math, the harness won't compensate.

## References

- [Towards Autonomous Mathematics Research (arXiv:2602.10177)](https://arxiv.org/abs/2602.10177)
- [Aletheia tackles FirstProof autonomously (arXiv:2602.21201)](https://arxiv.org/abs/2602.21201)
- [Semi-Autonomous Mathematics Discovery with Gemini: Erdos Problems (arXiv:2601.22401)](https://arxiv.org/abs/2601.22401)
- [DeepMind Blog: Gemini Deep Think](https://deepmind.google/blog/accelerating-mathematical-and-scientific-discovery-with-gemini-deep-think/)
- [GitHub: google-deepmind/superhuman](https://github.com/google-deepmind/superhuman/tree/main/aletheia)
- [IMO-Bench](https://imobench.github.io)
- [Natural20 Analysis](https://natural20.com/coverage/deepmind-aletheia-ai-autonomous-scientific-discovery)
