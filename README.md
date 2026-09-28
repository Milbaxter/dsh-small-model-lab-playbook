# DSH Harness Lab: Playbook

**Instructions for an AI agent (or human) running the next attempt at this experiment.** Read this whole repo before doing anything. It combines the original plan, what two independent attempts learned on 2026-09-28, and a retargeting (same day) toward the models people actually run DSH with.

## The question

Which plugin or config changes to [DeepSeek Harness (DSH)](https://github.com/deepseek-ai/deepseek-harness) make **full DSH on its default and best models** (`deepseek-v4-flash`, `deepseek-v4-pro`) genuinely better at real work: task completion, recovery from faults, verification, cross-session memory and long-horizon context? Gains must hold up on held-out tasks and on the stronger model, not come from benchmark hacking.

**Success means:** a user who installs the champion bundle into their normal DSH setup with V4-flash or V4-pro is measurably better off (more tasks solved, or the same solved for fewer tokens) on work like their own.

## Why this was retargeted (read first)

The first two attempts used Qwen3-8B. That design was unlikely to produce plugins that help a real DSH user:

| Problem | Consequence |
|---|---|
| Small-model failures are not frontier failures | Qwen3-8B's top failure is stopping to describe a fix instead of applying it. V4-class models rarely do that, so the lead candidate (the stop guard) would mostly add calls and false-positive nudges on the real target. |
| No cost reason for the small model | On OpenRouter, `deepseek-v4-flash` (DSH's default model) costs $0.134/$0.268 per 1M input/output tokens (Alibaba, fp8), with $0.027 per 1M for cached input. `qwen/qwen3-8b` costs $0.117/$0.455. The real target model is about as cheap. |
| Artificial setting | A forced 32k window and toy tasks measured the 8B model's ceiling (about 5% on the Opus bank), not the harness. |
| No transfer check | A gain that holds up on Qwen3-8B says nothing about V4-flash or V4-pro. |

The retargeted design uses **V4-flash as the primary model**, **V4-pro as a required confirmation**, realistic tasks calibrated on V4-flash, and candidates chosen from V4-flash's own failure traces. Qwen3-8B is kept only as an optional cheap first screen, and only for candidates that are scale-general.

## What is already known (don't re-derive it)

| Finding | Evidence | Still relevant? |
|---|---|---|
| Plain DSH (`sdk` profile) beats the one-tool `sdk-minimal` profile | Both attempts on Qwen3-8B | Probably. Re-check cheaply on V4-flash. |
| The autonomy-policy plugin does not help on Qwen3-8B | Opus: −1.6 held-out; Astra: −6.2 dev (significant) | Untested on V4. Worth one baseline arm. |
| Qwen3-8B stops early (describes the fix, ends the turn) | Opus bank, top failure tag in every arm | Small-model specific. Test only if V4-flash traces show it. |
| `compaction-basic` defaults to `headroomTokens: 65536` | DSH 0.1.5rc1 config | A real bug for any model with a small window. Report it upstream. |
| Session-log upload and web tools need care | Both attempts | Yes: disable log upload in every arm; web tools off unless keyed. |

## Documents

| Doc | What it's for |
|---|---|
| [AGENT_PROMPT.md](AGENT_PROMPT.md) | A ready-to-paste goal prompt for the next agent |
| [docs/PLAYBOOK.md](docs/PLAYBOOK.md) | Step-by-step run order with time and budget targets |
| [docs/SPEC.md](docs/SPEC.md) | Fixed settings: model pins, arms, isolation, task bank rules, statistics, promotion gates |
| [docs/LESSONS.md](docs/LESSONS.md) | What went wrong or right in both attempts, and why |
| [docs/NEXT_EXPERIMENTS.md](docs/NEXT_EXPERIMENTS.md) | Candidate changes, and how to pick them from traces |
| [docs/REUSE.md](docs/REUSE.md) | Existing code to reuse instead of rewriting (proxy, sandbox, runner, stats, gates) |

## Prior attempts (Qwen3-8B)

- **Original plan:** [Milbaxter/dsh-small-model-lab](https://github.com/Milbaxter/dsh-small-model-lab)
- **Opus attempt:** [Milbaxter/dsh-small-model-lab-opus-v2](https://github.com/Milbaxter/dsh-small-model-lab-opus-v2) (code + [RESULTS.md](https://github.com/Milbaxter/dsh-small-model-lab-opus-v2/blob/main/RESULTS.md)); private tasks in `Milbaxter/dsh-tasks-opus-v2`
- **Astra/Codex attempt:** [Milbaxter/dsh-small-model-lab-opus](https://github.com/Milbaxter/dsh-small-model-lab-opus) (the repo name is historical; evidence is in `evidence/`); private tasks in `Milbaxter/dsh-tasks-opus`

## License

MIT
