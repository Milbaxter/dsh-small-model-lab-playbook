# Run order

Time targets assume one agent session on a MacBook-class machine or a small CPU server.

**Budget:** about **$80** OpenRouter spend is recommended: ~$50 for V4-flash (bank calibration, baselines, loop), ~$25 for the V4-pro confirmation, ~$5 slack. With only $20, cut the bank to about 40 tasks, use k=3 for screening, run at most 3 iterations, and run the V4-pro check on a 15-task held-out subset. Say in RESULTS.md that power is limited.

## Step 0: Preflight (≤ 45 min, < $0.10)
1. Read this repo, the [plan repo](https://github.com/Milbaxter/dsh-small-model-lab) and the prior RESULTS ([Opus](https://github.com/Milbaxter/dsh-small-model-lab-opus-v2/blob/main/RESULTS.md), [Astra evidence](https://github.com/Milbaxter/dsh-small-model-lab-opus/tree/main/evidence)). Remember those used Qwen3-8B; their task-level findings mostly don't carry over.
2. **Create uniquely named repos** (check `gh repo view` first): one public code repo, one **private** task repo. `.env` must be git-ignored. Never commit keys.
3. Locate `OPENROUTER_API_KEY`. The user may have to point you to it. Verify it with `GET /api/v1/key` and never print it.
4. Pick the V4-flash and V4-pro providers (`GET /api/v1/models/<id>/endpoints`): tool calling, fp8, prompt caching. Make one direct tool-calling test call to each through the proxy. If either fails, stop and report.
5. Pick the machine. If you'd use a server, check quota **and** that you can reach it (IPv4) before relying on it. Otherwise run locally with `caffeinate -i`.

## Step 1: Plumbing (≤ 1 h, < $0.30)
Reuse code; don't rewrite (see [REUSE.md](REUSE.md)).
1. Start the pinning proxy with the V4 model allowlist, provider pin, cache-aware cost logging, network retry, a per-run call budget, a ledger and a hard cap about $1.50 under the budget. Remove the 32k enforcement for V4 (keep it only for the optional Qwen screen).
2. Wire the arm patches: `minimal`, `standard`, `autonomy`. Verify:
   - one smoke run per arm passes end to end on V4-flash
   - the autonomy text appears in the system prompt
   - no host skills or instructions leak into the prompt
3. Set up per-run isolation (sandbox or container). Check that the agent can't read the task bank, other runs, prior attempts' task repos or secrets (including other `.env` files under `$HOME`), and can only reach the proxy.

## Step 2: Task bank with early calibration (≤ 3 h, about $5)
1. Write or adapt **15 tasks** (3 per family), at least half from real repos. Verify the graders both ways. Run `standard` on V4-flash, k=2.
2. Adjust difficulty per family toward 30–60% by changing the work itself (bigger repo, more files, subtler bug, hidden edge cases), never by adding or removing hints. Expect V4-flash to solve toy tasks at ~100%; that's the failure mode to avoid.
3. Scale to 50–70 tasks. Run `standard` k=2 on all of them, drop or replace tasks that are broken or at 0%/100%, and aim for each family at 30–60%.
4. Terminal-Bench 2.0: run plain DSH k=1 on a pool of about 40 candidate tasks, and pre-register a rule-based subset of 20–30 where the solve rate is 20–70%.
5. Freeze held-out, transfer and the Terminal-Bench subset: commit, publish only hashes, and record the commit.

## Step 3: Baselines and failure mining (≤ 2 h, about $10)
1. `standard` at k=5 on dev + held-out + Terminal-Bench. `minimal` and `autonomy` at k=3 on held-out only.
2. Build the dev failure digest from `standard` traces with the re-derived tags. Read at least 20 failed dev traces by hand. **The ranked list of failures seen here, not NEXT_EXPERIMENTS.md, decides iteration 1.**

## Step 4: Loop (the main event; about $5–8 per iteration)
For each iteration (maximum 5):
1. **Propose.** From the dev digest, pick the top failure that a harness change could plausibly fix for any model. Write the hypothesis, mechanism, scale-generality statement, disconfirmation criterion and record *first*.
2. **Build** the change under `harness/candidates/<name>/` and run the leakage check.
3. **Optional Qwen screen** (only if cheap and the change is model-agnostic): 10 dev tasks k=2 on Qwen3-8B. A crash or a clear regression rejects the candidate; a gain proves nothing.
4. **Smoke:** 8 dev tasks × 1 run on V4-flash. Reject crashes and loops.
5. **Dev:** k=3 screen, interleaved with the champion re-run in the same window. Early exit if the paired dev difference is ≤ 0 and cost isn't lower. Then k=5 if it survives.
6. **Held-out** (k=5) plus the **matched-budget control**, then **transfer** and **Terminal-Bench** (k=3–5).
7. **Gate:** run `python -m lab.gate`. Gates 1–4 and 6–8 must pass before spending on gate 5 (V4-pro).
8. Commit the record either way.

## Step 5: Confirmation and finish (≤ 1.5 h, about $25)
1. **V4-pro confirmation** for the final champion (if it isn't plain DSH): held-out (or its pre-registered subset), k=3, paired with plain DSH on V4-pro.
2. Package the champion:
   - as a DSH bundle (`package.json` with `dsh.bundle.patch`, `index.js`, `cordis.patch.yml`) installable with `dsh plugin add` into a normal DSH setup (no lab proxy needed)
   - plus `direct.patch.yml` and a one-command `run.sh` that uses the user's normal DeepSeek provider
3. Keep the runner able to evaluate `ext:<dir>` bundles on your held-out split (for cross-evaluation).
4. Write RESULTS.md: the baseline table (V4-flash, and V4-pro for the final comparison), one line per iteration, the dev vs. held-out gap, flash vs. pro, cost per solved task, the champion and how to install it, spend and wall-clock, upstream bugs found, an honest read on what's real vs. noise, and blockers. End with a one-paragraph answer to: **"If I install this into my normal DSH with V4-flash or V4-pro, am I better off, and by how much?"**

## Stop conditions
- Spend approaches the cap. The proxy refuses at the cap, so stop scheduling about $1.50 before it.
- The user asks you to stop. Report on **complete** repetitions only (e.g. k=3 across all arms), not partial ones.
