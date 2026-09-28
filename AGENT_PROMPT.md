# Goal prompt for the next agent (copy and paste)

```
GOAL
Run the DSH harness lab experiment, following https://github.com/Milbaxter/dsh-small-model-lab-playbook.
Read that repo completely first (README, AGENT_PROMPT, docs/PLAYBOOK, SPEC, LESSONS, NEXT_EXPERIMENTS, REUSE),
then the original plan https://github.com/Milbaxter/dsh-small-model-lab and the two prior attempts linked there.

Aim: find DeepSeek Harness (https://github.com/deepseek-ai/deepseek-harness) plugin/config changes that make
full DSH on its real models (deepseek-v4-flash, confirmed on deepseek-v4-pro) genuinely better at real work
(task completion, recovery, verification, memory, long-horizon context), not benchmark-hacked.
Success = a user who installs the champion bundle into normal DSH is measurably better off.

SETUP
- Primary model: deepseek/deepseek-v4-flash on OpenRouter, one pinned provider (fp8, tool calling, prompt caching),
  pinned via a local proxy (docs/SPEC.md). Confirmation model: deepseek/deepseek-v4-pro.
  Qwen3-8B is an optional cheap screen only; it can reject a candidate but never promote one.
  Verify tool calling with one call per model first.
- Key: OPENROUTER_API_KEY. Ask me where it is if it's not in the environment. Never print or commit it.
- Machine: <local Mac with caffeinate | server X> (confirm reachability before relying on a server).
- Repos: create a NEW uniquely named public code repo <name> and a PRIVATE task repo <name>-tasks.
  Check that the names are free first. Tasks, graders and hidden tests never go into the public repo.
- Reuse the existing code listed in docs/REUSE.md instead of rewriting it.

SCOPE AND TIMEBOX
Follow docs/PLAYBOOK.md: preflight → plumbing (≤1h) → realistic task bank with EARLY calibration on V4-flash
(each family 30–60% for plain DSH) + Terminal-Bench 2.0 split → freeze held-out/transfer/TB → baselines + failure
mining → up to 5 loop iterations, each chosen from the V4-flash dev failure digest → V4-pro confirmation of the
final champion → package → RESULTS.md. Reach the first loop iteration within about 6 hours of starting.

RULES
- $<budget> total OpenRouter spend (about $80 recommended; see PLAYBOOK for a $20 fallback). Track your own spend
  in the proxy ledger (the key may be shared). Stop about $1.50 before the cap.
- Freeze held-out, transfer and the Terminal-Bench subset before iteration 1. Record the commit hash and never edit them.
- Proposer (you) sees only dev tasks and traces. Everything else is aggregates only.
- One change per iteration, with a written scale-generality statement. Promote only if every gate in docs/SPEC.md
  passes, including the V4-pro confirmation. Reject and no-change are valid outcomes.
- Harness bugs found along the way go into the baseline config for all arms and get written up for DSH upstream.
- Pin model, provider and sampling. Interleave compared arms in the same time window.
- Write an experiment record per iteration.
- Ask me only for decisions that truly block progress; otherwise decide, note it, and continue.

FINAL REPORT (RESULTS.md in the public repo, and to me in chat, in plain English)
Baseline table; one line per iteration (change, dev, held-out, decision); dev vs held-out gap; flash vs pro;
cost per solved task; champion and how to install it into normal DSH (bundle + one-command run.sh);
upstream bugs found; total spend and wall-clock; an honest read on what is real vs noise; blockers; and a
one-paragraph answer to "if I install this with V4-flash or V4-pro, am I better off, and by how much?"
```
