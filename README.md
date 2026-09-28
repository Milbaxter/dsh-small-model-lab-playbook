# DSH Small-Model Lab: Playbook

**Instructions for an AI agent (or human) running the next attempt at this experiment.** Read this whole repo before doing anything. It combines the original plan with what two independent attempts learned on 2026-09-28.

## The question

Can plugin or config changes to [DeepSeek Harness (DSH)](https://github.com/deepseek-ai/deepseek-harness) make a small model (Qwen3-8B) **genuinely** more capable at task completion, recovery from faults, cross-session memory and long context? Gains must hold up on held-out tasks, not come from benchmark hacking.

## What is already known (don't re-derive it)

| Finding | Evidence |
|---|---|
| Plain DSH (`sdk` profile) beats the one-tool `sdk-minimal` profile | Both attempts: minimal was 0% vs ~9% (Opus bank); minimal was significantly lower on dev and held-out (Astra/Codex bank) |
| The autonomy-policy plugin from [optimal-deepseek-harness-setup](https://github.com/Milbaxter/optimal-deepseek-harness-setup) does **not** help and may hurt | Opus: −1.6 pts held-out (CI to 0). Astra: −6.2 pts dev (CI entirely < 0), −3.8 held-out |
| Qwen3-8B's #1 failure in DSH is **stopping early**: it diagnoses correctly, then writes the fix in chat and ends the turn without applying it | Opus bank: the top failure tag in every arm and split |
| Thinking mode is not worth it at this budget | Paired pilot: 2.3× cost, 3.8× slower, 0 extra solves |
| A 4-task Terminal-Bench subset gives no signal | Astra: 0/60 runs passed in every arm |

The best next step is iteration 1, the **stop guard** (a bounded Stop hook). It is built and fires correctly but has never been measured. See [docs/NEXT_EXPERIMENTS.md](docs/NEXT_EXPERIMENTS.md).

## Documents

| Doc | What it's for |
|---|---|
| [AGENT_PROMPT.md](AGENT_PROMPT.md) | A ready-to-paste goal prompt for the next agent |
| [docs/PLAYBOOK.md](docs/PLAYBOOK.md) | Step-by-step run order with time and budget targets |
| [docs/SPEC.md](docs/SPEC.md) | Fixed settings: model pins, arms, isolation, task bank rules, statistics, promotion gates |
| [docs/LESSONS.md](docs/LESSONS.md) | What went wrong or right in both attempts, and why |
| [docs/NEXT_EXPERIMENTS.md](docs/NEXT_EXPERIMENTS.md) | Ranked candidate changes, starting with the stop guard |
| [docs/REUSE.md](docs/REUSE.md) | Existing code to reuse instead of rewriting (proxy, sandbox, runner, stats, gates) |

## Prior attempts

- **Original plan:** [Milbaxter/dsh-small-model-lab](https://github.com/Milbaxter/dsh-small-model-lab)
- **Opus attempt:** [Milbaxter/dsh-small-model-lab-opus-v2](https://github.com/Milbaxter/dsh-small-model-lab-opus-v2) (code + [RESULTS.md](https://github.com/Milbaxter/dsh-small-model-lab-opus-v2/blob/main/RESULTS.md)); private tasks in `Milbaxter/dsh-tasks-opus-v2`
- **Astra/Codex attempt:** [Milbaxter/dsh-small-model-lab-opus](https://github.com/Milbaxter/dsh-small-model-lab-opus) (the repo name is historical; evidence is in `evidence/`); private tasks in `Milbaxter/dsh-tasks-opus`

## License

MIT
