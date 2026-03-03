# Project Memory

Specific facts, edge cases, and session-specific context that doesn't belong
in CLAUDE.md (which covers general workflow and standards).

## Aletheia Key Facts

- Full analysis: `docs/aletheia-analysis.md`
- Architecture: Generator → Verifier → Reviser loop, all using same base model (Gemini 3 Deep Think)
- No formal verification — entirely natural language proofs
- Published verification prompt in Appendix A of arXiv:2602.21201
- Generator/Reviser prompts NOT published
- Orchestration code NOT published
- Performance: 95.1% on IMO-ProofBench Advanced, but only 6.5% useful on research problems
- IMO-ProofBench data: 60 problems (30 basic + 30 advanced), NOT 105 as some sources claim
- Data at github.com/google-deepmind/superhuman/blob/main/imobench/
- The harness is simple to build; performance depends on base model math ability
- Next step: implement GVR loop, benchmark against IMO-ProofBench to calibrate Claude's math reasoning vs Gemini Deep Think
