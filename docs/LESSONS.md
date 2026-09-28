# Lessons from the 2026-09-28 attempts

> Both attempts used Qwen3-8B. Lesson 0 explains why the next attempt targets DeepSeek V4 instead. The infrastructure lessons (1, 7, 8, 9) still apply; the task-level findings mostly don't carry over.

Two agents ran the same plan independently: **Opus** (Claude) and **Astra** (Codex). Both reached P2 baselines, and neither ran an improvement iteration before stopping.

## Outcome snapshot

| | Opus | Astra |
|---|---|---|
| Where it ran | Local Mac (UpCloud unusable) | UpCloud CPU box + Docker |
| Task bank | 74 tasks (27 dev / 31 held-out / 16 transfer) | 64 tasks (32 / 16 / 16) |
| P2 repetitions | k=2 (stopped early) | k=5, 960 runs |
| Plain DSH pass rate (held-out) | 4.8% | 32.5% |
| Pass rate by family (standard, dev) | completion 5%, recovery 29%, memory 0%, long-context 0% | completion 18%, recovery 13%, memory 30%, context **100%** |
| Autonomy plugin vs plain | −1.6 held-out (n.s.) | −6.2 dev (significant), −3.8 held-out |
| Loop iterations run | 0 (stop guard built, unmeasured) | 0 (iteration 1 "disable web search" pre-registered) |
| Spend | $4.63 | about $9.5+ |

## Lessons

### 0. Measure on the model you want to improve
- The goal is plugins that help a real DSH user, whose model is `deepseek-v4-flash` (DSH's default) or `deepseek-v4-pro`. Qwen3-8B's failures (stopping early, 0% memory, collapse at 32k) are mostly capability limits of an 8B model, not harness gaps that a strong model also hits. Compensating scaffolding for a weak model often costs a strong model extra calls for nothing.
- The small model wasn't even cheaper: on OpenRouter, V4-flash (Alibaba, fp8) costs $0.134/$0.268 per 1M input/output tokens, with $0.027 for cached input, versus Qwen3-8B's $0.117/$0.455.
- A forced 32k window and toy tasks gave a ~5% baseline (Opus bank), which leaves no statistical power and measures the model, not the harness.
- **Do this instead:** calibrate realistic tasks on V4-flash, pick candidates from V4-flash failure traces, and require the champion to hold up on V4-pro. Use a small model only as an optional early-rejection screen.

### 1. Infrastructure: settle it first, in under an hour
- **UpCloud:** the account quota was full (6 cores, 12 GB RAM and 2 IPv4 addresses all used). A new IPv6-only box was unreachable from a Mac without IPv6. **Check quotas and reachability before planning around a server.** Running locally on the Mac works fine: the model is remote, and the load is about 14 parallel DSH processes.
- **Network blips kill runs.** A 13-minute Wi-Fi/DNS outage turned 46 of 148 calibration runs into INFRA failures. The model proxy must wait out connectivity errors (retry about 3 minutes before failing a call), and the runner must be able to re-run INFRA-tagged runs.
- **Repo names:** both agents were told to use the same repo names and collided. Create **uniquely named** repos and check they don't exist first.

### 2. Calibrate before building the whole bank
- **Opus** built 74 tasks, then found a ~10% pass rate. Its natural prompts plus strict graders were too hard for an 8B model, whose main failure is stopping early.
- **Astra's** aggregate was in range (30–40%), but only because **one family was at 100%** (context) while completion and recovery were near 0%. A family at 0% or 100% contributes no signal.
- **Do this instead:** write about 3 tasks per family, run them k=2 on `standard`, adjust, then scale up. Target **30–60% for each family**, not just in aggregate.

### 3. Get to the loop fast
Opus spent most of its time on plumbing, 74 hand-written tasks and side quests (a thinking-mode pilot, Terminal-Bench plumbing). The actual research is the loop.
- Hour 0–1: plumbing.
- Hour 1–3: bank plus calibration.
- Hour 3–4: P2.
- Then iterate.
- **Start P2 on dev + held-out only.** Transfer is only needed for candidates that reach the gate.

### 4. Target the dominant failure first
- **Opus traces:** Qwen3-8B diagnoses the bug, prints "Here's the corrected implementation…" with code, and ends the turn, even on a trivial off-by-one. The stop guard targets exactly this. It is a DSH plugin on `agent/turn-stopping` that calls `agent.steer(...)`, the same mechanism DSH's own Claude-Code hook bridge uses.
- **Astra's first iteration** removed a web-search tool that had no credentials. Removing broken tools is sensible hygiene, but it belongs in the **baseline configuration** (both arms), not in the loop, because a gain from it measures a config bug rather than capability.

### 5. Don't change the task prompts to paper over harness failures
Adding "actually edit the file" to task prompts would lift the pass rate but hide the very failure a harness should fix. Keep prompts natural; make difficulty come from the work itself.

### 6. The external benchmark must be able to move
Astra's 4 Terminal-Bench tasks scored 0/60 in every arm, so the gate can't distinguish anything. Pre-register by rule, but choose a pool where plain DSH solves some tasks. Use 15–20 easy tasks, and check with a k=1 plain-DSH run on the *pool* (before any candidate exists) that the solve rate is at least 10%. The Opus runner (in-container DSH plus the task's own `run-tests.sh`) worked; `hello-world` passed.

### 7. Budget reality
- About 7k prompt tokens per call for `standard` (system prompt plus ~27 tool schemas) versus 350 for `minimal`.
- Standard runs cost about $0.006–0.009; minimal about $0.0005.
- Failed runs that hit the step budget cost 5–10× more than passes, so keep step budgets tight (20–30 calls).
- $20 affords roughly 2,500 standard-arm runs total. That's enough for P2 at k=5 on about 60 tasks plus about 4 loop iterations, but not if you also do k=5 transfer for every arm and a large Terminal-Bench run.
- The OpenRouter key may be shared with other agents. Track **your own** spend per run in the proxy ledger, not the account total.

### 8. Things that were right (keep them)
- **Pinning proxy:** it forces model, provider (Alibaba, no fallbacks), sampling and non-thinking mode, enforces the 32k window, logs cost per run, holds the real key, and refuses over-budget calls. It also makes "never substitute a bigger model" mechanical.
- **Isolation:** a fresh DSH home and fake `$HOME` per run. On the Mac, the host's `~/.claude/skills` leaked into DSH's skill list until `HOME` was isolated. Use a seatbelt sandbox locally, or one Docker container per run.
- **Privacy:** disable `session-log-deepseek` (it attaches session logs to requests even through custom gateways) and OTel telemetry in every arm.
- **Graders:** every grader is verified both ways (reference passes, starting state fails). Truth data lives outside the agent-readable run directory.
- **Statistics:** paired, task-level bootstrap statistics with k=5, and arms interleaved in the same time window.
- **Freezing:** held-out and transfer frozen by commit hash before the loop, with only hashes published.

### 9. DSH gotchas (release 0.1.5rc1)
- `compaction-basic` defaults to `headroomTokens: 65536`. With a 32k context window the pressure threshold is negative, so pressure compaction throws every step and only overflow recovery works. That is a real DSH bug for small-window models: report it upstream and fix it in the baseline config, rather than counting it as a loop gain.
- `sdk` profile defaults need `DSH_PERMISSION_MODE=danger-full-access` headless; approvals can't be answered. Use an outer sandbox instead.
- On macOS seatbelt, allow `/dev/ptmx` (the persistent bash uses a PTY) and don't deny file *metadata* reads, or `realpath` fails.
- The web tool needs a DeepSeek key. Disable `tool-web`, `web`, `web-search-deepseek` and `web-fetch-http` for offline tasks in **all** arms.
- Plugins can be plain ESM files loaded by path in a patch `insert` row; no packaging is needed for evaluation.

## Lessons from attempt 3 setup (2026-09-28, DeepSeek V4.1-flash)

### 10. The real model is cheap enough; wall-clock is the constraint
- `deepseek/deepseek-v4.1-flash` on OpenRouter via DeepInfra fp8 ($0.14 in / $0.42 out, cached input $0.0042 per 1M) with DSH's default reasoning (high). Prompt caching hits on repeated prefixes; a cached call costs about 10× less.
- DeepSeek's own endpoint may be excluded by the account's "no training on paid prompts" privacy setting. DeepInfra is also cheaper on output.
- Measured: about $0.002 per run on small bank tasks and about $0.01 per Terminal-Bench 2.0 task. €50 buys thousands of runs; the Mac's CPU (emulated amd64 containers) is the bottleneck.

### 11. Old toy banks are saturated on V4
- The 74-task Opus bank: V4-flash passes 65 of 73 finished runs (89%). The failures cluster in **multi-session memory**, where DSH forgets rules and preferences stated in an earlier session. That is a scale-general, harness-addressable gap.

### 12. Frontier agents snoop: lay out runs like a real machine
- V4-flash reads files outside its workspace to recover context: the runner's `../worker.json` (which held earlier sessions' replies) and DSH's own session store. Never put runner bookkeeping where the agent can read it.
- Mirror a real install: `$HOME/.dsh` for DSH_HOME and `$HOME/projects/<task>` for the workspace. The agent may still read `~/.dsh`, as it could on a real machine. Log a "snooped ~/.dsh" metric per run, and note that a lab `~/.dsh` with a single project is easier to snoop than a real one.

### 13. DSH 0.1.7 plumbing notes
- The Python SDK (0.1.5rc1 on PyPI) can drive the current npm CLI (`@deepseek-ai/dsh@0.1.7-rc.2`) via `dsh_bin`. Session logs are now `session.v4.jsonl(.zstd)`.
- `sdk-minimal`'s persistent PTY shell executes a setuid binary, which the macOS seatbelt sandbox forbids; drop that arm on macOS.
- The `sdk` and `headless` profiles differ by only 4 plugins, so `dsh --profile headless --json` inside Terminal-Bench containers (via a Harbor custom agent with the Linux Node + DSH closure mounted read-only) matches the local runner.
- Terminal-Bench 2.0 images are amd64. On an 8 GB Docker Desktop, run at most 3 concurrent tasks and exclude tasks that need more than 4 GB or nested virtualization, or the Docker daemon dies.

