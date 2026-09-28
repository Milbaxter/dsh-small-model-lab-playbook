# Run order

Time targets assume one agent session on a MacBook-class machine or a small CPU server. Money targets assume a $20 cap.

## Step 0: Preflight (≤ 45 min, < $0.05)
1. Read this repo, the [plan repo](https://github.com/Milbaxter/dsh-small-model-lab) and the prior RESULTS ([Opus](https://github.com/Milbaxter/dsh-small-model-lab-opus-v2/blob/main/RESULTS.md), [Astra evidence](https://github.com/Milbaxter/dsh-small-model-lab-opus/tree/main/evidence)).
2. **Create uniquely named repos** (check `gh repo view` first): one public code repo, one **private** task repo. `.env` must be git-ignored. Never commit keys.
3. Locate `OPENROUTER_API_KEY`. The user may have to point you to it; verify it with `GET /api/v1/key` and never print it.
4. One direct tool-calling test call to `qwen/qwen3-8b` pinned to Alibaba. If it fails, stop and report.
5. Pick the machine. If you'd use a server, check quota **and** that you can reach it (IPv4) before relying on it. Otherwise run locally with `caffeinate -i`.

## Step 1: Plumbing (≤ 1 h, < $0.10)
Reuse code; don't rewrite (see [REUSE.md](REUSE.md)).
1. Start the pinning proxy with network-retry, a per-run call budget, a ledger and a hard cap of about $18.
2. Wire the arm patches: `minimal`, `standard`, `autonomy`. Verify:
   - one smoke run per arm passes end to end
   - the autonomy text appears in the system prompt
   - no host skills or instructions leak into the prompt
3. Set up per-run isolation (sandbox or container). Check that the agent can't read the task bank, other runs or secrets, and can only reach the proxy.

## Step 2: Task bank with early calibration (≤ 2.5 h, about $1.5)
1. Write **12 tasks** (3 per family). Verify the graders both ways. Run `standard` k=2.
2. Adjust difficulty per family toward 30–60%, by changing the work itself rather than adding hints. Reference points:
   - Opus natural bug-fix tasks: about 5–15%
   - Astra context tasks: 100%, far too easy
3. Scale to 60–80 tasks (procedural generators help). Run `standard` k=2 on all of them, drop or replace tasks that are broken or trivially 100%, and aim for each family at 30–60%.
4. Freeze held-out and transfer: commit, publish only hashes, and record the commit.

## Step 3: P2 baselines (≤ 1.5 h, about $5)
`minimal`, `standard`, `autonomy` at k=5 on dev + held-out, plus `standard` on transfer (it is needed as the champion reference). Run at parallelism 12–16.

The autonomy result is already known (not helpful). You may run it at k=5 on held-out only, to replicate that result cheaply.

## Step 4: External benchmark pool check (≤ 45 min, about $1)
Pre-register a Terminal-Bench rule. Run plain DSH k=1 on the pool, and keep a subset where it solves ≥10% (15–20 tasks). Freeze the list before any candidate exists.

## Step 5: Loop (the main event; about $2–3.5 per iteration)
For each iteration (maximum 5):
1. **Propose.** Read the dev failure digest (dev only), pick the top failure tag, and write the hypothesis and record *first*.
2. **Build** the change under `harness/candidates/<name>/` and run the leakage check.
3. **Smoke:** 8 dev tasks × 1 run. Reject crashes and loops.
4. **Dev:** 5 runs per dev task, interleaved with the champion re-run in the same window. Early exit if the paired dev difference is ≤ 0.
5. **Held-out** (k=5) plus the **matched-budget control**, then **transfer** (k=5).
6. **Gate:** run `python -m lab.gate`. Promote only if every gate passes.
7. Commit the record either way.

Start with the stop guard ([NEXT_EXPERIMENTS.md](NEXT_EXPERIMENTS.md)).

## Step 6: Finish (≤ 45 min)
1. Run the Terminal-Bench subset: plain DSH vs. the final champion, k=3–5.
2. Package the champion:
   - as a DSH bundle (`package.json` with `dsh.bundle.patch`, `index.js`, `cordis.patch.yml`)
   - plus `direct.patch.yml` and a one-command `run.sh`
3. Keep the runner able to evaluate `ext:<dir>` bundles on your held-out split (for cross-evaluation).
4. Write RESULTS.md: the baseline table, one line per iteration, the dev vs. held-out gap, the champion and how to install it, spend and wall-clock, an honest read on what's real vs. noise, and blockers.

## Stop conditions
- Spend approaches the cap. The proxy refuses at the cap, so stop scheduling about $1.50 before it.
- The user asks you to stop. Report on **complete** repetitions only (e.g. k=2 across all arms), not partial ones.
