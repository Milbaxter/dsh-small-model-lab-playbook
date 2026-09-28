# Fixed specification

Change any of this only with a written reason, and record it in RESULTS.md.

## Model layer (identical for every arm)
- **Primary model: `deepseek/deepseek-v4-flash`** (DSH's default model) via OpenRouter, pinned to one provider with `allow_fallbacks: false, require_parameters: true`. Pick the provider in preflight: it must support tool calling, report fp8 (not fp4) quantization, and support prompt caching. Alibaba (fp8, $0.134/$0.268 per 1M, cached input $0.027) is the default choice. Record the provider in every experiment record, and never switch providers mid-comparison. If the user has a DeepSeek API key, the first-party API is an acceptable alternative, but it must be fixed for the whole run.
- **Confirmation model: `deepseek/deepseek-v4-pro`**, same pinning rules, used only for the final champion (see gate 5).
- **Optional screen: `qwen/qwen3-8b`** (Alibaba, non-thinking, 32k), only for cheap early rejection of scale-general candidates. A Qwen result can reject a candidate but never promote one.
- **Sampling:** use DSH's shipped defaults for the model (don't impose Qwen's sampling). Record them. Reasoning/thinking mode: whatever DSH's default is for V4-flash; don't change it per arm.
- **Context window:** the model's real window as DSH configures it. **No artificial 32k cap** on the primary model. Long-context tasks must stress it with realistic volume (big repos, long logs, many turns), not with an artificial limit.
- **Proxy:** all calls go through the local pinning proxy (model allowlist, provider pin, ledger, per-run call budget, hard cap, network retry, API fault injection). DSH talks to `http://127.0.0.1:<port>/r/<run_id>/v1` with a dummy key. The proxy must forward and log cache usage (`cached_tokens`) so cost is computed correctly.
- **DSH:** `deepseek-harness-sdk`, pinned to an exact version (0.1.5rc1 was used before; use the latest release and record it).

## Arms
| Arm | Definition |
|---|---|
| `standard` (**plain DSH**, initial champion) | Shipped `sdk` profile, `agent-default-model` → lab provider (V4-flash) |
| `minimal` | Shipped `sdk-minimal` profile. Baseline only, to confirm the full profile earns its tokens on V4. |
| `autonomy` | `standard` + autonomy-policy plugin. Baseline only (held-out, k=3) to check the Qwen null result on V4. |
| candidates | the current champion + exactly one change |

Every arm gets the same fixes:
- `session-log-deepseek.enabled: false` and OTel disabled
- web tools disabled unless a working key is configured for **all** arms
- `DSH_PERMISSION_MODE=danger-full-access` inside an outer sandbox
- a fresh DSH home, workspace and `$HOME` per run

## Task bank (private repo, never public)
- **Realism first.** Tasks should look like work a DSH user actually does: fix a failing test in a real-sized repo, add a feature across several files, debug a flaky script, migrate a config, resume work from an earlier session. At least **half the bank** should be real-repo tasks (small open-source repos with a real bug or feature, SWE-bench-style) or adapted Terminal-Bench 2.0 tasks, not toy single-file scripts.
- **Size and families:** 50–70 tasks across 5 families: completion (multi-file), recovery (faults), verification (the task is only done if the agent checks its own work: hidden edge cases a careful agent would test), memory (multi-session), long-horizon/context (large repos or logs, many steps, an early constraint that must survive compaction). **Each family at 30–60% for plain DSH on V4-flash.**
- **Splits:** dev about 40%, held-out about 40%, transfer about 20% (different domains or languages). All families appear in every split.
- **Terminal-Bench 2.0 split:** a pre-registered subset of 20–30 tasks where plain DSH on V4-flash solves 20–70% at k=1 on the pool, frozen before any candidate exists. This is a **first-class split** in the gate, not a side check.
- **Format per task:** `task.yaml` (id, family, split, prompts[], budget {max_calls, timeout}, optional api_faults), `workspace/`, optional `setup.py` (argv: ws, rep, private_truth_dir), hidden `grade.py`, `hidden/`, `ref/`.
- **Grader verification:** every grader is verified both ways: the starting state FAILS and the reference solution PASSES. Truth lives outside the agent-readable directory.
- **Memory tasks:** 2–3 prompts, each a fresh DSH process and session id, sharing the workspace and DSH home.
- **Recovery tasks:** realistic workspace faults (rate-limited or flaky command, permission denied, malformed config, moved file, missing dependency, hanging command, stale lock, merge conflict, renamed tool, wrong encoding) plus some proxy-injected API 429/500/503 errors.
- **Prompts:** natural, the way a user would write them. Never add hints that paper over harness failures.
- **Freezing:** freeze held-out, transfer and the Terminal-Bench split (commit hash plus content hashes) **before** the first loop iteration. Never edit them afterwards.

## Runs
- Step budget: 40–80 model calls per session (V4-flash solves real tasks in more steps than toy ones), enforced by the proxy. Wall-clock 10–20 minutes per session.
- Screening uses k=3; every formal comparison that feeds the gate uses k=5. Interleave arms: repetition → shuffled tasks → shuffled arms. Compared arms run in the same time window.
- Log per run: pass, grader detail, calls, prompt, cached and completion tokens, cost, wall time, tool calls, tool errors, compaction events, final message and full trace.
- **Failure tags** (first match wins): INFRA, TIMEOUT, MAX_TURNS, CONTEXT_OVERFLOW, IDLE_LOOP (same call ≥3×), STOPPED_EARLY (final message announces work without doing it), REFUSAL (0 tool calls), WRONG_VERIFY (claims success without checking, or checks wrongly), NO_VERIFY (never ran a check), LOST_CONSTRAINT (violated an instruction given earlier or in a prior session), BAD_EDIT, REASONING. Re-derive the tag regexes on V4-flash traces; the Qwen ones are tuned to Qwen's phrasing.

## Statistics
- Per task, the pass rate over k runs. The paired difference is candidate − champion per task. The 95% CI comes from a task-level bootstrap (10,000 resamples, fixed seed).
- Report tokens and dollars per solved task alongside pass rates. For a frontier user, "same pass rate at 30% lower cost" is a real win.

## Promotion gate (all must hold)
1. Dev improves: paired pass-rate mean > 0, **or** equal pass rate with ≥ 15% lower cost per solved task.
2. **Held-out (V4-flash): paired 95% CI lower bound > 0** (for a cost-only win: pass-rate CI lower bound ≥ −3 points and a cost reduction whose CI excludes 0).
3. No transfer regression and no Terminal-Bench regression (each CI upper bound is not < 0).
4. Beats a **matched-budget control**:
   - prompt additions: a length-matched neutral "placebo" section
   - extra compute (retries, continuations): the champion given the same extra calls or tokens (e.g. best-of-n with a fixed budget)
5. **Confirmation on V4-pro:** on the held-out split (or a pre-registered 20-task subset of it), k=3, paired with plain DSH on V4-pro. Point estimate ≥ 0 and CI upper bound not < 0, and cost per solved task ≤ 1.1× plain. A candidate that helps flash but hurts pro is reported as **flash-only** and is not the champion.
6. Tokens per solved task ≤ 1.25× the champion, unless the held-out gain is ≥ 10 points.
7. Leakage check passes: no 6-word shingles, IDs or file names shared between plugin text and any task.
8. **Scale-generality statement:** the record says, in one paragraph, why the mechanism should help a strong model and not only a weak one. If the only justification is "the model is weak", reject.

## Proposer rules (for the agent itself during the loop)
- **Can see:** dev tasks, dev traces, per-task dev results.
- **Can see only as aggregates:** held-out, transfer, Terminal-Bench and V4-pro pass rates and CIs.
- **Per iteration:** one change, with the target failure (from V4-flash dev traces), hypothesis, mechanism, scale-generality statement and disconfirmation criterion written **before** running.
- **Iteration limit:** at most 5 iterations. Reject and no-change are valid outcomes.
- **Records:** write one experiment record per iteration (template in the plan repo's `docs/LOOP.md`, plus the scale-generality statement and the V4-pro result).
- **Upstream findings:** a harness bug found along the way (like the compaction headroom default) is reported separately as a DSH issue or PR draft. It goes into the **baseline config** for all arms, not into the loop as a "gain".
