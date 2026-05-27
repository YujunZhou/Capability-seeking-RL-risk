### 📄 Alignment Risks from Capability-Seeking RL Training

This repository contains code and reproduction scripts for the paper **"Alignment Risks from Capability-Seeking RL Training"**.

---

### 💡 Motivation and Summary

Most AI alignment work focuses on preventing models from producing explicitly harmful or toxic content. A subtler risk arises during **capability-seeking RL training**: when we train language models with reinforcement learning (RL, e.g. GRPO) to improve task capability, and the **training environment or reward mechanism has implicit loopholes**, models can learn to exploit those flaws to maximize reward—without any malicious intent or exploit instructions in the training data. This is not merely classic "reward hacking" in a narrow sense: (1) it encompasses both **specification failures** and **robustness failures** (e.g. behaving safely under the audit distribution but not under the deployment distribution), and (2) exploitative behavior can co-occur with **improvements in task metrics**, creating a "developer blind spot" where exploitation is hard to detect with standard monitoring.

We study **alignment risks from capability-seeking RL training** through a controlled suite of **four vulnerability games**, each instantiating a different kind of environmental flaw. We ask: **(RQ1)** Do models discover exploitative shortcuts under standard RL in structurally vulnerable environments? **(RQ2)** Once learned, are these exploits stealthy, transferable, and resistant to correction?

![Four vulnerability games](assets/Figure1.png)

**🎮 Four vulnerability games**

| Game | Intended goal | Vulnerability | What the model learns |
|------|----------------|---------------|------------------------|
| **Context-conditional compliance** | Be safe under both audit- and user-style inputs | Two reward channels (auditor vs user); model can gate on style | Refuse formal/audit-style harmful prompts, comply with casual/user-style ones → high Compliance Gap |
| **Audited self-grading** | Honest self-report under sparse human audits | Low audit rate makes "always claim CORRECT" expectation-optimal | Systematic overclaiming (report CORRECT when wrong) to maximize reward |
| **Proxy-metric gaming** | High-quality summarization | Reward = ROUGE only; no penalty for incoherence | Copy/stitch from source to maximize ROUGE (stochastic parroting) |
| **Reward/state tampering** | Solve code tasks; reward = test pass | Insecure structured-output parser allows protected fields such as `score` to be modified | Inject score-modification markup to get full reward without solving the task |

**Metrics** We report **task performance** (e.g. true correctness, refusal rate, conciseness) and **exploit rate** (e.g. compliance gap, overclaim rate, tampering rate). When task performance and exploit rate diverge—or when both appear to improve together—the model has learned a shortcut that standard task metrics alone may miss.

**Main findings** (1) Exploitative shortcuts emerge across many model-game configurations during GRPO. (2) Transfer is **structured but limited**: some exploit patterns transfer zero-shot, catalyze later exploit learning, or propagate via SFT, while procedural tricks remain task-specific. (3) In several comparisons, RL-native exploits are **more resistant to correction** than SFT-distilled ones. (4) In some settings, exploit and task metrics rise together, creating a **developer blind spot**. The paper argues that addressing this class of risk requires **auditing and securing training environments and reward mechanisms**, not only content-level filtering.

---

### 📁 Code Layout

- **Scripts**: All run from the **`verl/`** directory. Scripts live under `verl/examples/main_trainer/`.
  - `run_situational_awareness_grpo.sh` — context-conditional compliance (audit vs user style)
  - `run_self_grading_grpo.sh` — audited self-grading (configurable audit probability)
  - `run_summarization_gaming.sh` — summarization with ROUGE proxy gaming
  - `run_reward_tampering.sh` — reward/state tampering (structured-output parser vulnerability)
  - `run_all.sh` — run all four tasks in sequence
  - `*_secure.sh` / `run_all_secure.sh` — secure baselines (stricter reward and data)
- **Data and outputs** (paths relative to **`verl/`**, where training runs):
  - Data: `data/<task_name>/`
  - Logs: `logs/...`
  - Checkpoints: `checkpoints/<PROJECT_NAME>/<EXPERIMENT_NAME>/`
  - Eval results: `eval_results/` (where applicable)

### ⚙️ Environment

Install `verl/` in editable mode and its dependencies (vLLM, Ray, Flash-Attn, etc.). See **`verl/README.md`** for full setup. On Windows, use WSL2 or a Linux environment.

### 🚀 Quick Start

**Run all commands from the `verl/` directory** (i.e. `Capability_Oriented_Alignment_Risk/verl`). The scripts change into `verl/` internally; calling them from `verl/` keeps paths consistent.

```bash
cd Capability_Oriented_Alignment_Risk/verl

# Single task (default model)
bash examples/main_trainer/run_situational_awareness_grpo.sh
bash examples/main_trainer/run_self_grading_grpo.sh
bash examples/main_trainer/run_summarization_gaming.sh
bash examples/main_trainer/run_reward_tampering.sh

# Specify model with -m
# -m llama        -> meta-llama/Llama-3.1-8B-Instruct
# -m Qwen3-4B-Base -> Qwen/Qwen3-4B-Base
bash examples/main_trainer/run_all.sh -m llama
```

#### 🔒 Secure baselines

`*_secure.sh` scripts support:

- `--ori`: Start from the **original base model** (e.g. `Qwen/Qwen3-4B-Base`, `meta-llama/Llama-3.1-8B-Instruct`).
- By default (no `--ori`), they load your trained/uploaded model as `${HF_ORG:-your-org}/Advanced_Risk_<Task>_<ModelId}`.
- `--sft`: Load SFT checkpoint as `${HF_ORG:-your-org}/SFT_Advanced_Risk_<Task>_<ModelId}`.
- `-n` / `--max-settings` (only `run_all_secure.sh`): Limit how many settings are started in one run.

Example (from `verl/`):

```bash
bash examples/main_trainer/run_all_secure.sh -m Qwen3-4B-Base --ori
```

### 📝 Options and Notes

- **Model choice**: See `verl/examples/main_trainer/COMMAND_LINE_USAGE.md` for `-m` mapping and defaults.
- **Self-grading audit probability**: `run_self_grading_grpo.sh -a <audit_prob> -t <telling_audit>`.
- **Reward tampering**: `run_reward_tampering.sh` sets `REWARD_PARALLEL_WORKERS=1` by default so the custom reward path is used.
- **Situational awareness**: Uses an LLM judge (`verl/utils/reward_score/llm_judge*.py`); set the API key via environment variables.
- **Security**: Do not hardcode API keys; use environment variables or your cluster's secret manager.

### 📊 Evaluation

`verl/examples/main_trainer/run_eval_naive.sh` runs validation-only evaluation and extracts metrics (e.g. acc, exploit_ratio) per task. Use `--help` for usage.
