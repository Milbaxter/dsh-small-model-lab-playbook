# Candidate changes, ranked

Each iteration makes one change. Every candidate below is general (not task-specific) and targets a failure seen in dev traces.

## 1. Stop guard: bounded continuation (target: STOPPED_EARLY)
- **Evidence:** STOPPED_EARLY was the top failure in every arm (Opus bank). The model ends a turn with "Here's the corrected implementation…" plus a code block, "Let's check…", or "Would you like me to…?" without calling tools.
- **Mechanism:** a DSH plugin listens to `session/event`, tracking the last assistant text and tool calls per turn. On `agent/turn-stopping`, if the last message has no tool calls and matches intent, proposed-fix or permission patterns, it calls `agent.steer(userMessage)` with a short "do the work now, stop only when done and verified or truly blocked" notice. It nudges at most 2 times per turn.
- **Code:** [`harness/candidates/stop-guard/stop-guard.js`](https://github.com/Milbaxter/dsh-small-model-lab-opus-v2/blob/main/harness/candidates/stop-guard/stop-guard.js) (packaged as `bundle/dsh-lab-opus-stop-guard`). A mechanics check on 3 dev tasks confirmed it fires and the model resumes work. **Not yet measured.**
- **Matched-budget control:** the champion allowed the same number of extra model calls, e.g. a generic "continue" after every turn end, up to 2 per turn with no detection. This separates "smart detection" from "just more calls".
- **Disconfirmation:** no paired dev gain, or gains that vanish on held-out, or that are matched by the blind-continue control.
- **Risks:** false positives on legitimate stops (memory "just note this" turns) and extra tokens.

## 2. Compaction tuned for small windows (target: CONTEXT_OVERFLOW / long-context failures)
- **Evidence:** in DSH 0.1.5rc1, `compaction-basic` uses `headroomTokens: 65536`. With a 32k window the pressure budget is negative, so pressure compaction errors out every step and only overflow recovery (1 retry) runs.
- **Change:** set `headroomTokens` (e.g. 4096) and `thresholdRatio` (e.g. 0.7) for the lab model via `modelPolicies`, so pressure compaction actually works.
- **Note:** this is a real configuration bug for any small-context model, which makes it a legitimate harness improvement.

## 3. Tool-set trimming (target: tokens, confusion)
- **Evidence:** `standard` sends about 27 tool schemas, about 7k tokens per call, 20× the minimal profile's prompt. Small models get distracted (e.g. `todo_write` used as "memory", `subagent` tools, skills lists).
- **Change:** disable one coherent group at a time (subagents/workflow/ralph, or skills). Measure the token saving and the capability effect.

## 4. Durable notes for memory (target: memory-family failures)
- **Evidence:** facts from session 1 are lost in session 2. `standard` auto-loads `AGENTS.md`, but the model never writes to it, and `todo_write` is not persistent.
- **Change:** a prompt section or small tool telling the agent to save user preferences and facts to `AGENTS.md` (auto-loaded next session). Watch for token cost.

## 5. Repeat-call handling (target: IDLE_LOOP)
- **Evidence:** the model reruns identical commands after "(no output)" (e.g. a command whose output was redirected to a file).
- **Change:** lower the `repeat-tool-reminder` thresholds (e.g. [2, 3, 5]), or add a tool-result note "(no output — command succeeded; output was redirected)".

## Not worth repeating
- **Autonomy-policy plugin as-is:** measured null or negative twice.
- **Thinking mode:** 2.3× cost, no gain in the pilot.
- **Disabling unconfigured web search as a loop iteration:** make it part of the baseline config instead.
