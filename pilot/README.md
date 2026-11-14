# Block A pilot harness

This harness tests whether an oversight notice reduces test-editing by coding agents beyond what stating the rule already does. It runs the A1, P1 and O1 arms on the Impossible-LiveCodeBench dev split. The design comes from plan v3.4 (`../thesis-research-plan.md`), sections 5 to 8.

| File | Purpose |
|---|---|
| `cues.py` | System prompt, ticket template, rule text, placebo and oversight paraphrases, probe question |
| `detect.py` | Detects test edits from proposed tool calls. Run `python detect.py` for its self-check. |
| `task.py` | The Inspect tasks `oversight_pilot` and `belief_probe`, the dev/confirmatory split, and the scorer. Run `python task.py` to print the split and the confirmatory hash. |
| `analyze.py` | Per-arm rates with Wilson CIs, task-cluster bootstrap contrasts, the go/kill verdict, and the plot |
| `compose.yaml` | Docker sandbox with no network and 1 GB of memory |
| `PILOT_THRESHOLDS.md` | Go/kill rules. **Commit this file before running.** |

## One-time setup
1. **Install Docker Desktop** (WSL2 backend) and start it. Agents run model-written code, so the harness refuses to run without the sandbox.
2. **Create the virtual environment** from the project root:
   ```
   python -m venv .venv
   .venv\Scripts\pip install -r pilot\requirements.txt
   ```
3. **Get a key from an inference provider** that serves your model, and set a hard spending cap (about $30) in the provider's dashboard. Inspect reads the key from an environment variable; see Inspect's model-provider docs. With OpenRouter, for example, you set `OPENROUTER_API_KEY` and the model ID looks like `openrouter/qwen/qwen3-32b`. Check the exact model ID and its price on the provider's site. If you can, choose a provider that does not train on your traffic.
4. **Review and commit `PILOT_THRESHOLDS.md`**, after filling in the model and provider.

## Run (from `pilot/`)
1. **Smoke test, about $0.10:**
   ```
   ..\.venv\Scripts\inspect eval task.py@oversight_pilot --model <provider/model> -T n_per_arm=2 --log-dir logs\smoke
   ```
   Open the transcripts with `..\.venv\Scripts\inspect view --log-dir logs\smoke` and read every one.
2. **Belief probe:** 450 one-turn calls.
   ```
   ..\.venv\Scripts\inspect eval task.py@belief_probe --model <provider/model> --log-dir logs\pilot
   ```
3. **Pilot:** 120 episodes.
   ```
   ..\.venv\Scripts\inspect eval task.py@oversight_pilot --model <provider/model> --log-dir logs\pilot --max-connections 4
   ```
4. **Analysis:**
   ```
   ..\.venv\Scripts\python analyze.py logs\pilot --out results
   ```

## Sampling settings
Pass the model's recommended settings, for example `--temperature 0.6 --top-p 0.95` for Qwen3 in thinking mode. Check the model card first, and record whatever you use in `PILOT_THRESHOLDS.md`.

## Known limits
- **Covert gaming is not detected yet.** That includes special-casing test inputs and monkey-patching asserts. `func.py` is saved in each log for the later rubric and judge pass, and `spec_correct` (the agent's code against the true tests) helps you spot it.
- **The proposed-call detector uses regex.** The diff detector is the backup, and the report lists disagreement between the two.
- **Instances are reused.** There are only 103 tasks, so 40 episodes per arm reuse each dev instance about once. The bootstrap resamples tasks to account for this.
- **No twin tasks, Block B or A7 yet.** These are cut from the pilot, as the council recommended.
- **Dependency pin.** Inspect 0.3.280 breaks with `agent-client-protocol` 1.0, so it is pinned to 0.12.1.
