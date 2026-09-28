# Fixed specification

Change any of this only with a written reason, and record it in RESULTS.md.

## Model layer (identical for every arm)
- `qwen/qwen3-8b` via OpenRouter, `provider: {order: ["alibaba"], allow_fallbacks: false, require_parameters: true}`. Alibaba is the only provider, and tool calling works (verify with one call first). **Never substitute a bigger model.** If tool calling breaks, stop and report.
- Non-thinking (`reasoning: {enabled: false}`), temperature 0.7, top_p 0.8, top_k 20 (Qwen's non-thinking recommendation), max_tokens 4096.
- Context window **32,768** (Qwen3-8B native, no YaRN), enforced by the proxy with an OpenAI-style `context_length_exceeded` 400 error. The provider itself serves 131k, so DSH would otherwise never hit overflow and costs would balloon.
- All of this is forced by a local pinning proxy. DSH talks to `http://127.0.0.1:<port>/r/<run_id>/v1` with a dummy key.
- DSH: `deepseek-harness-sdk` (pin the exact version; 0.1.5rc1 was used). Custom provider via `llm-pi-ai`, `api: openai-completions`, `compat: {supportsDeveloperRole: false, maxTokensField: max_tokens, supportsStore: false, supportsReasoningEffort: false}`.

## Arms
| Arm | Definition |
|---|---|
| `minimal` | Shipped `sdk-minimal` profile plus an inserted `llm-pi-ai` row |
| `standard` (**plain DSH**, initial champion) | Shipped `sdk` profile, `agent-default-model` → lab provider |
| `autonomy` | `standard` + autonomy-policy plugin (already measured: not helpful) |
| candidates | `standard` (or the current champion) + exactly one change |

Every arm gets the same fixes:
- `session-log-deepseek.enabled: false` and OTel disabled
- web tools disabled
- `DSH_PERMISSION_MODE=danger-full-access` inside an outer sandbox
- a fresh DSH home, workspace and `$HOME` per run

## Task bank (private repo, never public)
- **Size and families:** 60–80 tasks across 4 families (completion, recovery, memory, long-context), with **each family at 30–60% for plain DSH**.
- **Splits:** dev about 40%, held-out about 40%, transfer about 20% (different domains). All families appear in every split.
- **Format per task:** `task.yaml` (id, family, split, prompts[], budget {max_calls, timeout}, optional api_faults), `workspace/`, optional `setup.py` (argv: ws, rep, private_truth_dir), hidden `grade.py`, `hidden/`, `ref/`.
- **Grader verification:** every grader is verified: the starting state FAILS and the reference solution PASSES. Truth lives outside the agent-readable directory.
- **Memory tasks:** 2–3 prompts, each a fresh DSH process and session id, sharing the workspace and DSH home.
- **Recovery tasks:** realistic workspace faults (rate-limited or flaky command, permission denied, malformed config, moved file, missing dependency, hanging command, stale lock, merge conflict, renamed tool, wrong encoding) plus some proxy-injected API 429/500/503 errors.
- **Long-context tasks:** content that overflows 32k if read naively, plus an early constraint (ticket ID, output path, "don't touch tests/") that must survive compaction.
- **Prompts:** natural. Never add hints that paper over harness failures ("make sure you edit the file").
- **Freezing:** freeze held-out and transfer (commit hash plus content hashes) **before** the first loop iteration. Never edit them afterwards.

## Runs
- Step budget: 20–30 model calls per session, enforced by the proxy. Wall-clock 6–10 minutes per session.
- k = 5 for every formal comparison. Interleave arms: repetition → shuffled tasks → shuffled arms. Compared arms run in the same time window.
- Log per run: pass, grader detail, calls, prompt and completion tokens, cost, wall time, tool calls, tool errors, compaction events, final message and full trace.
- **Failure tags** (first match wins): INFRA, TIMEOUT, MAX_TURNS, CONTEXT_OVERFLOW, IDLE_LOOP (same call ≥3×), STOPPED_EARLY (final message announces or shows work without doing it), REFUSAL (0 tool calls), WRONG_VERIFY (claims success), BAD_EDIT, REASONING.

## Statistics
- Per task, the pass rate over k runs. The paired difference is candidate − champion per task. The 95% CI comes from a task-level bootstrap (10,000 resamples, fixed seed).

## Promotion gate (all must hold)
1. Dev improves (paired mean > 0).
2. **Held-out paired 95% CI lower bound > 0.**
3. No transfer regression (the transfer CI upper bound is not < 0).
4. Beats a **matched-budget control**:
   - prompt additions: a length-matched neutral "placebo" section
   - extra compute (retries, continuations): the champion given the same extra calls or tokens (e.g. best-of-n with a fixed budget)
5. Tokens per solved task ≤ 1.25× the champion, unless the held-out gain is ≥ 10 points.
6. Leakage check passes: no 6-word shingles, IDs or file names shared between plugin text and any task.
7. External benchmark (final champion only): a pre-registered Terminal-Bench subset, paired with plain DSH, k=3–5, reported separately. It must not regress.

## Proposer rules (for the agent itself during the loop)
- **Can see:** dev tasks, dev traces, per-task dev results.
- **Can see only as aggregates:** held-out and transfer pass rates and CIs.
- **Per iteration:** one change, with the target failure, hypothesis, mechanism and disconfirmation criterion written **before** running.
- **Iteration limit:** at most 5 iterations. Reject and no-change are valid outcomes.
- **Records:** write one experiment record per iteration (template in the plan repo's `docs/LOOP.md`).
