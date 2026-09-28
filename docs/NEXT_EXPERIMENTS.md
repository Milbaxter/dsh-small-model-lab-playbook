# Candidate changes

Each iteration makes one change. **Pick iteration 1 from the V4-flash dev failure digest** (PLAYBOOK step 3), not from this list. This list is a set of priors: each item says which failure it targets and why it should help a strong model too. If the digest doesn't show the target failure, skip the candidate.

## Selection rule
A candidate is worth a loop iteration only if all three hold:
1. Its target failure is among the top failure tags in V4-flash dev traces.
2. The mechanism makes sense for a strong model (the scale-generality statement in SPEC gate 8).
3. It is general: no task-specific text, and it would ship as a DSH bundle a user could install.

## Priors, ranked by expected value for a frontier-model user

### 1. Verification discipline (target: WRONG_VERIFY / NO_VERIFY)
- **Why it should generalize:** strong models also declare success without running the tests or checking edge cases, especially in long sessions. This is a widely reported frontier-agent failure.
- **Mechanism options (one per iteration):** a stop-time check that the turn ran a test or verification command after its last edit, and if not, one steer asking it to verify (bounded, at most 1 per turn); or a system-prompt section on verification.
- **Control:** a blind "please double-check" steer at every turn end (same extra calls).

### 2. Context management for long horizons (target: LOST_CONSTRAINT, CONTEXT_OVERFLOW, long-task degradation)
- **Evidence:** in DSH 0.1.5rc1, `compaction-basic` uses `headroomTokens: 65536`; check what the shipped defaults do for V4's window. Long sessions on strong models degrade through context rot and forgetting early constraints, not through overflow.
- **Changes:** compaction thresholds tuned for quality (compact earlier, with a summary that keeps constraints verbatim); or a pinned "task constraints" block that survives compaction.
- **Note:** any headroom *bug* goes into the baseline config and gets reported upstream. Only a genuine tuning or policy change counts as a loop iteration.

### 3. Durable cross-session memory (target: memory-family failures)
- **Evidence:** facts from session 1 are lost in session 2. `standard` auto-loads `AGENTS.md`, but the model rarely writes to it.
- **Change:** a small tool or prompt section that saves user preferences and project facts to `AGENTS.md` (or a DSH memory store), auto-loaded next session. This helps every model size, since the information is otherwise simply absent.
- **Watch:** token cost and stale or incorrect notes.

### 4. Tool-set trimming (target: cost, distraction)
- **Evidence:** `standard` sends about 27 tool schemas, about 7k tokens per call. Unused groups (subagents, workflow, skills) cost tokens on every call.
- **Change:** disable one coherent group at a time. For a frontier user, the win is mainly **cost per solved task** (gate 1's cost clause), so measure it precisely, including cache effects.

### 5. Repeat-call handling (target: IDLE_LOOP)
- **Change:** lower the `repeat-tool-reminder` thresholds, or add a tool-result note for empty output. Only if V4-flash traces show loops.

### 6. Stop guard: bounded continuation (target: STOPPED_EARLY) — small-model prior
- **Evidence:** the top failure on Qwen3-8B (Opus bank). **Unknown on V4-flash; likely rare.**
- **Code:** [`harness/candidates/stop-guard/stop-guard.js`](https://github.com/Milbaxter/dsh-small-model-lab-opus-v2/blob/main/harness/candidates/stop-guard/stop-guard.js); a DSH plugin on `agent/turn-stopping` that calls `agent.steer(...)`, at most 2 nudges per turn. The same hook is the natural base for candidate 1.
- **Run it only if** STOPPED_EARLY is a top-3 tag on V4-flash dev. Otherwise it mostly adds false-positive nudges for a strong model.

## Not worth repeating
- **Autonomy-policy plugin as-is:** null or negative twice on Qwen3-8B. Re-check it once as a V4 baseline arm; don't loop on it.
- **Thinking-mode changes:** a model-layer setting, not a harness change. Keep DSH's default.
- **Disabling unconfigured web search as a loop iteration:** make it part of the baseline config instead.
