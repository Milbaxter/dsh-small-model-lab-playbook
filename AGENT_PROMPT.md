# Goal prompt for the next agent (copy and paste)

```
GOAL
Run the DSH small-model lab experiment, following https://github.com/Milbaxter/dsh-small-model-lab-playbook.
Read that repo completely first (README, AGENT_PROMPT, docs/PLAYBOOK, SPEC, LESSONS, NEXT_EXPERIMENTS, REUSE),
then the original plan https://github.com/Milbaxter/dsh-small-model-lab and the two prior attempts linked there.

Aim: find DeepSeek Harness (https://github.com/deepseek-ai/deepseek-harness) plugin/config changes that make
Qwen3-8B genuinely more capable (task completion, recovery, memory, long context), not benchmark-hacked.

SETUP
- Model: qwen/qwen3-8b on OpenRouter, provider Alibaba only, non-thinking, pinned via a local proxy (docs/SPEC.md).
  Verify tool calling with one call first. Never substitute a bigger model.
- Key: OPENROUTER_API_KEY. Ask me where it is if it's not in the environment. Never print or commit it.
- Machine: <local Mac with caffeinate | server X> (confirm reachability before relying on a server).
- Repos: create a NEW uniquely named public code repo <name> and a PRIVATE task repo <name>-tasks.
  Check that the names are free first. Tasks, graders and hidden tests never go into the public repo.
- Reuse the existing code listed in docs/REUSE.md instead of rewriting it.

SCOPE AND TIMEBOX
Follow docs/PLAYBOOK.md: preflight → plumbing (≤1h) → task bank with EARLY calibration (each family 30–60% for plain DSH)
→ freeze held-out/transfer → P2 baselines (k=5) → external benchmark pool check → up to 5 loop iterations,
starting with the stop guard (docs/NEXT_EXPERIMENTS.md #1) → Terminal-Bench check → package → RESULTS.md.
Reach the first loop iteration within about 5 hours of starting.

RULES
- $20 total OpenRouter spend. Track your own spend in the proxy ledger (the key may be shared). Stop about $1.50 before the cap.
- Freeze held-out and transfer before iteration 1. Record the commit hash and never edit them afterwards.
- Proposer (you) sees only dev tasks and traces. Held-out and transfer are aggregates only.
- One change per iteration. Promote only if every gate in docs/SPEC.md passes. Reject and no-change are valid outcomes.
- k = 5. Pin model, provider and sampling. Interleave compared arms in the same time window.
- Write an experiment record per iteration.
- Ask me only for decisions that truly block progress; otherwise decide, note it, and continue.

FINAL REPORT (RESULTS.md in the public repo, and to me in chat, in plain English)
Baseline table; one line per iteration (change, dev, held-out, decision); dev vs held-out gap; champion and
how to install it (bundle + one-command run.sh); total spend and wall-clock; an honest read on what is real vs noise;
blockers.
```
