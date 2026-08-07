# TACC FlexServ Benchmark Notebook — Task 5

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/nairr-portal/tacc-sandbox-5/blob/main/tapis_flexserv_benchmark_task_05.ipynb)

**[View the notebook](./tapis_flexserv_benchmark_task_05.ipynb)**

This notebook demonstrates training a random forest classifier to predict **compound activity** — whether a small molecule sufficiently inhibits a biological signal, a proxy used in virtual drug-screening pipelines — using the UT Austin TACC Vista cluster, TACC's Tapis platform, and a FlexServ instance. It:

1. **Authenticates and initializes FlexServ** — connects to the TACC/Tapis platform, submits and monitors a FlexServ job, and loads a specified LLM.
2. **Prepares data** — embeds and writes `dkpes_train.csv` and `dkpes_test.csv` directly into the notebook for self-containment (the same DKPES data already embedded in `tacc-sandbox-7`, re-encrypted with a fresh key for this notebook). The test file's true `Signal-inhibition` values are replaced with a dummy sentinel, matching the official benchmark's data-contamination mitigation.
3. **Trains and predicts, with self-debug** — the LLM is given the task instruction, dataset preview, and a domain-knowledge hint (see below), and must binarize the continuous `Signal-inhibition` training target, train a `RandomForestClassifier`, and predict activity for the held-out test compounds. Its generated code is actually executed; if it raises an exception or fails to write its output file, the error is fed back to the model for another attempt, up to 10 retries (see below).
4. **Scores the result** — computes ROC-AUC between the model's final predictions and the true (held-out) labels, and reports pass/fail against the same 0.91 threshold the official ScienceAgentBench evaluator uses.

## Self-debug loop

This notebook ports ScienceAgentBench's **self-debug** condition (`agent.py`'s `use_self_debug=True`) — the paper's own best-performing technique, reported to roughly double direct-prompting's success rate. Instead of one LLM call, the generated code is executed; each round only checks whether it (a) ran without raising an exception and (b) actually wrote `pred_results/dkpes_test_pred.csv`. On failure, the error is fed back as a 3-turn context (system prompt, most recent code, new error) — the same bounded context the official implementation uses — and the model gets another attempt, up to 10 retries.

**This is deliberately narrow, matching the official technique exactly**: it does not re-check scientific or output-format correctness (column names, the ROC-AUC bar) mid-loop — the real self-debug loop never re-runs the eval script until scoring happens once at the end. So this loop recovers from crashes and missing-output-file mistakes, but a run that completes cleanly with the wrong column name will halt as a "success" and only get caught by the final scoring cell. That's exactly what happened on this notebook's first manual run (the model wrote a `predicted_signal_inhibition` column instead of `Signal-inhibition`) — a deliberate choice not to patch around, since spelling out the exact required column name in the prompt would answer the test for the model on the precise failure mode ScienceAgentBench's self-debug baseline is meant to be evaluated against, not something this notebook's prompt does for it. Validated the loop mechanics against a mocked LLM before publishing (exception → retry, missing-file → retry, clean-run-wrong-column → halts as a false success, confirming the paragraph above empirically) — the test script itself is a scratch artifact, not part of this repo.

## Like `tacc-sandbox-3`: real numeric scoring, not eyeballing

Like task 3 (crystal bulk-modulus regression), this is a Model Development task with a hard numeric success bar (ROC-AUC ≥ 0.91) rather than a visual judge. There's no picture at the end of this notebook — the final cell prints an actual ROC-AUC and a PASS/FAIL verdict.

I validated this before publishing: the benchmark's own reference solution (`RandomForestClassifier` on the 11 functional-group columns, thresholded at 0.6) was run against the exact data embedded in this notebook and scores **ROC-AUC = 1.0000**, comfortably passing.

## Prompt variant: with domain knowledge — a threshold trap, not a leakage trap

Before publishing, I checked this dataset the same way task 3's leakage was found — by testing whether an LLM could shortcut the task using columns beyond the ones the gold program intends. The train/test CSVs carry 14 extra numeric columns (`TanimotoCombo`, `ComboScore`, and other molecular-docking/shape-similarity scores against a reference query) beyond the 11 functional-group indicator columns the gold program actually uses. Unlike task 3, **this isn't a leakage risk**: training on only those extra columns tops out at ROC-AUC 0.875 (below the bar), and combining them with the real features doesn't help either (also 0.875) — so an LLM that used them wouldn't gain a shortcut, just noise.

The real trap here is different: the task instruction says to choose "an appropriate threshold" to binarize `Signal-inhibition` into active/inactive labels, without saying how. This benchmark task's own scoring rubric even describes the intended method as "the median value as threshold from train data" — but the gold program's actual code uses a **fixed threshold of 0.6**, not the median, and the withheld gold labels were generated against that fixed cutoff. I confirmed empirically that this isn't a minor detail: with the correct (functional-group) features, the 0.6 threshold scores ROC-AUC 1.00, while the median threshold scores only 0.81 — a clean fail. The rubric's own stated method would not pass this task.

The 0.6 figure isn't something I invented to patch this: the upstream `ScienceAgentBench` repo's own `agent.py` demo script hardcodes a task with byte-identical instruction text to this one, together with a `domain_knowledge` block naming this exact feature list and the 0.6 cutoff — it's the benchmark authors' own annotation for this task, just missing from the officially published `verified` dataset's per-instance `domain_knowledge` field (one of only 4 of 102 tasks where that field is empty). The `domain_knowledge` block in this notebook's prompt cell restates that annotation.

## Running locally

Always use a local `venv/` in this directory when running Jupyter here — do not use a system or other environment's Python. On macOS, prefer a Homebrew Python (`python3.12`) over the system `/usr/bin/python3`, which links against an outdated LibreSSL and triggers `urllib3` warnings on every HTTPS request this notebook makes.

```
/opt/homebrew/bin/python3.12 -m venv venv
source venv/bin/activate
pip install jupyter notebook jupyterlab
jupyter lab   # or: jupyter notebook
```

The notebook's own runtime dependencies (`tapipy`, `pandas`, `scikit-learn`, etc.) are installed by its own first code cell — no separate requirements file needed.

## Source

Ported from ScienceAgentBench instance ID 5 (`Bioinformatics` / `Data Visualization` domain tag, though the task itself is model development, not visualization — the benchmark's own `subtask_categories` label appears mislabeled for this instance), adapted from `psa-lab/predicting-activity-by-machine-learning`'s `code/dkpes_fgroup_analysis.ipynb`. Reuses the same `dkpes_train.csv`/`dkpes_test.csv` data as `tacc-sandbox-7` (instance 7) and `tacc-sandbox-6`/`tacc-sandbox-8` (instances 6 and 8, not yet ported), the other three DKPES-family tasks in the benchmark.
