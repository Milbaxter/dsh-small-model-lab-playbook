# Reusable code

> **Changes needed for the V4 retarget:** in `lab/orproxy.py`, replace the single Qwen `PIN` with per-model pins (V4-flash, V4-pro, optional Qwen) and an allowlist; drop the 32k enforcement and the Qwen sampling for V4 (use DSH's defaults); and log `cached_tokens` and use the reported `cost` so caching is priced correctly. In `harness/arms/lab-provider.patch.yml`, set the model id and the real context window. In `lab/trace.py`, re-derive the failure-tag regexes on V4 traces and add NO_VERIFY and LOST_CONSTRAINT. In `lab/run.sb`, also deny reads of other attempts' task repos and every `.env` under `$HOME`.

Copy, don't rewrite. Everything below is MIT and lives in public repos.

## From [dsh-small-model-lab-opus-v2](https://github.com/Milbaxter/dsh-small-model-lab-opus-v2) (local Mac, seatbelt sandbox)
| File | What it does |
|---|---|
| `lab/orproxy.py` | Pinning proxy: model, provider, sampling and mode pins; 32k enforcement (Qwen screen only) using the real Qwen3 tokenizer (`lab/assets/qwen3-tokenizer.json`, fetched from HF, git-ignored); per-run call budget; API fault injection; cost ledger; hard cap; network retry |
| `harness/arms/*.patch.yml` | Arm definitions and the shared lab-provider patch (privacy and telemetry off, web tools off) |
| `lab/run.sb` | macOS seatbelt profile: writes only to the run dir, no reads of the task bank, runs or secrets, network only to localhost |
| `lab/sweep.py` | Resumable, interleaved sweeps (`--retry-infra`); private truth dir; grading outside the agent's view; trace extraction |
| `lab/worker.py` | One attempt (multi-session) through the DSH Python SDK |
| `lab/trace.py` | Session-log decoding (zstd JSONL) and failure tagging, including STOPPED_EARLY |
| `lab/stats.py`, `lab/gate.py` | Paired task-bootstrap stats, tables and the promotion gate |
| `lab/digest.py`, `lab/leakage.py` | Dev-only proposer digest and the leakage check |
| `harness/plugins/placebo` | Length-matched control plugin |
| `lab/tbench.py`, `lab/tbench_setup.sh` | Terminal-Bench in-container runner: relocatable Linux Python with DSH mounted at `/opt/dshpy`; proxy via `host.docker.internal` |
| `lab/arms.py` | `ext:<dir>` evaluation of external bundles (`arm.yaml` or plain DSH bundle `package.json`) |
| Task generators | In the private repo `dsh-tasks-opus-v2` (`src/common.py` format plus `src/verify.py`). Access needs the owner. |

## From [dsh-small-model-lab-opus](https://github.com/Milbaxter/dsh-small-model-lab-opus) (Astra/Codex, Docker on a server)
- A Docker actor image per run with CPU, memory, PID and token limits. The gateway reserves cost for in-flight requests.
- Harbor-based Terminal-Bench 2.0 integration (`scripts/harbor_dsh.py`).
- Matched-budget control implemented as best-of-n under an equal token and call budget (`scripts/matched_control.py`).
- A cross-evaluation interface with an external profile directory (`patch.json` + plugins mounted at `/profile`).

## Setup commands (Opus stack)
```sh
uv venv -p 3.12 .venv && . .venv/bin/activate
uv pip install deepseek-harness-sdk httpx pyyaml numpy tokenizers zstandard
curl -sL -o lab/assets/qwen3-tokenizer.json https://huggingface.co/Qwen/Qwen3-8B/resolve/main/tokenizer.json
set -a; . ./.env; set +a; nohup python lab/orproxy.py --port 18080 &
LAB_TASKS=/path/to/private/tasks python -m lab.sweep --sweep p2 --split dev,heldout --arms minimal,standard --k 5 --parallel 14
python -m lab.stats table --sweeps p2
```
