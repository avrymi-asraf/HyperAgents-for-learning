# HyperAgents Project Deep Dive: End-to-End Code Flow

This document provides a long-form, implementation-level walkthrough of how HyperAgents works, with a strong focus on runtime code flow, control flow, data flow, and artifacts written to disk.

---

## 1) What the system is doing

HyperAgents is an iterative self-improvement loop for agents. At a high level, each generation does this:

1. Select a parent agent version from an archive.
2. Reconstruct that parent state by applying patch lineage.
3. Run a **meta agent** that edits the codebase to produce a new patch.
4. Validate that edited agent code still imports/compiles.
5. Evaluate the edited task agent on one or more domains.
6. Save scores and metadata.
7. Add the new generation to the archive.
8. Optionally evaluate archive ensemble behavior.
9. Select next parent and continue.

Main entry point: `generate_loop.py`.

---

## 2) Core files and responsibilities

### Top-level orchestration
- `generate_loop.py`: full evolutionary loop, Docker orchestration, generation lifecycle.
- `run_meta_agent.py`: CLI launcher for meta agent and patch extraction.
- `meta_agent.py`: meta agent class that prompts an LLM to edit repository code.
- `run_task_agent.py`: CLI launcher for task agent and patch extraction.
- `task_agent.py`: task-solving agent that returns a JSON response for an input task.

### Agent/LLM infrastructure
- `agent/base_agent.py`: base `AgentSystem` class and per-thread logging setup.
- `agent/llm.py`: model constants and `get_response_from_llm()` through LiteLLM.
- `agent/llm_withtools.py`: tool-aware chat loop, tool-call parsing/execution/retry.
- `agent/tools/`: built-in editable/bash tool implementations available to the agent.

### Generation loop utilities
- `utils/gl_utils.py`: archive operations, patch lineage, score loading, initial workspace setup, parent selection logic.
- `utils/docker_utils.py`: build/start containers, copy files in/out, log outputs, cleanup.
- `utils/git_utils.py`: patch/diff/reset/commit helpers.
- `utils/domain_utils.py`: domain split/score key/eval subset/staged-eval settings.

### Evaluation and scoring
- `domains/harness.py`: runs `TaskAgent` across dataset rows (threaded) and writes predictions.
- `domains/report.py`: computes reports/metrics from predictions.
- `ensemble.py`: picks best generation score and returns task prediction from that generation’s saved predictions.
- `utils/run_ensemble.py`: CLI runner for ensemble scoring.
- `select_next_parent.py`, `utils/run_select_next_parent.py`: externalizable parent selection module/runner.

---

## 3) Runtime artifacts and directory model

A run writes to an output directory like:

- `outputs/generate_<run_id>/`
  - `archive.jsonl` (history of generation IDs)
  - `generate_loop.log` (args log)
  - `gen_initial/` (copied initial evals + copied initial repository)
  - `gen_<N>/`
    - `metadata.json`
    - `generate.log`
    - `agent_output/model_patch.diff`
    - `<domain>_eval/`, `<domain>_eval_val/`, `<domain>_eval_test/`
    - optionally `report_ensemble_<domain>_<split>.json` and ensemble predictions

Each generation’s `metadata.json` is the central record of lineage and execution status:
- parent, current gen id
- `prev_patch_files` and `curr_patch_files`
- whether meta agent succeeded
- whether eval ran / full eval ran
- whether generation is a valid parent candidate

---

## 4) Start of execution: CLI parsing and config normalization

Execution usually begins in `generate_loop.py` (`__main__`).

Important normalization done before the loop:
- Domain list validated against allowed choices.
- `eval_samples` aligned to domain count (or defaulted to all `-1`).
- Domain-specific subsets collected via `get_domain_eval_subset()`.
- Guard for `--optimize_option=only_ensemble` requiring `--agent_archive_path`.

Then `generate_loop(...)` is invoked with normalized parameters.

---

## 5) `generate_loop(...)`: the high-level state machine

`generate_loop()` does two major phases:

### Phase A: initialization/bootstrap

1. Create Docker client.
2. Resolve parent-selection policy (`latest` forced for `no_archive` baseline).
3. Resume path or fresh path:
   - **Resume**: recover run ID/output dir and last archive state.
   - **Fresh**: create output dir, call `setup_initial_gen()`.

`setup_initial_gen()` (from `utils/gl_utils.py`) does the following:
- Copies baseline eval directories into `gen_initial`.
- Creates a runnable copy of the repo under `gen_initial/<REPO_NAME>`.
- Excludes heavy or irrelevant directories/files depending on mode.
- Writes a generated README that describes optimization objective.
- Optionally swaps in baseline-specific files (e.g., DGM mode).
- Creates a commit and returns `(root_dir, root_commit)`.

Then initialization creates first archive node behavior based on `meta_patch_files` and reset flags:
- `initial` node if no meta patches.
- `initial` with metadata if reset mode.
- explicit generation `0` if initial meta patches need immediate evaluation.

If needed, generation 0 is produced using `generate(...)` and optional polyglot evaluation.

### Phase B: iterative generation loop

For `current_genid` from start to `max_generation`:

1. Call `generate(...)` to produce and evaluate a child.
2. Append child to archive via `update_and_save_archive()`.
3. If parent meta-agent failed, mark parent invalid.
4. If polyglot domain included, run polyglot harness path.
5. Optionally compute ensemble scores.
6. Produce progress/archive plots.
7. Select next parent (`select_parent()` or `select_next_parent_container()`).
8. Continue.

---

## 6) `generate(...)`: generation creation in detail

`generate()` is the core unit that creates one child generation.

### 6.1 Setup

- Creates `gen_<current_genid>` local folder and generation logger.
- Builds metadata object with lineage placeholders.
- Starts a container from project image (`build_container(...)`).

### 6.2 Reconstruct ancestor code state

Inside container:

1. Optional baseline-specific immutable copy (`/DONOTTOUCH_<REPO_NAME>` for `no_selfimprove`).
2. Apply startup meta patches for starting node (`initial` or `0`).
3. Load parent lineage patch files (`get_patch_files(...)`).
4. Apply all lineage patches in order (`apply_diffs_container(...)`).

`apply_diffs_container()` behavior:
- Reads each patch file.
- Filters out changes under `domains/` before applying.
- Copies patch into container and applies with `patch -p1`.
- Stages + commits resulting state in container repo.
- Returns resulting commit hash used as base for diff extraction.

### 6.3 Prepare previous evaluations for meta-agent context

Depending on baseline and mode, previous generations/evals are copied into container (`copy_prev_eval_to_container(...)`) and pruned:
- remove current generation folder
- remove val/test dirs
- remove copied worktree dirs
- remove pyc files
- remove val/test-labeled files

### 6.4 Run meta agent

Meta-agent command path:
- Baseline DGM mode: run `coding_agent.py`.
- Default: run `run_meta_agent.py`.

`run_meta_agent.py` then:
1. Instantiates `MetaAgent`.
2. Calls `MetaAgent.forward(repo_path, eval_path, iterations_left)`.
3. Resets unwanted domain diffs (`reset_paths_to_commit(..., paths=["domains/"])`).
4. Writes repository diff vs base commit to `model_patch.diff`.

`MetaAgent.forward(...)` currently prompts:
- “Modify any part of codebase at `<repo_path>`.”
- Calls `chat_with_agent(... tools_available='all')`.

So the actual code edits are generated by the LLM through the tool-enabled chat loop.

### 6.5 Capture patch output

After container run:
- Copy `/tmp/agent_output` back to local `gen_<N>/agent_output`.
- Detect `model_patch.diff` existence.
- If patch exists, mark `run_eval=True`.
- Run import/compilation checks via `run_commands_to_check_compilation(...)`.

### 6.6 Evaluate produced task agent

If evaluation enabled:
- For each non-polyglot domain, call `eval_produced_agent(...)`.
- Stage eval may run first with small sample count.
- Full eval runs only if stage gate passes.

`eval_produced_agent(...)` for each split:
1. Run `python -m domains.harness ...` against `./task_agent.py`.
2. Run `python -m domains.report ...`.
3. Copy eval output folders back to local generation dir.

Polyglot is handled separately by `run_harness_polyglot(...)` with domain-specific staged/full procedure.

### 6.7 Finalization

In `finally` block (always):
- `git reset --hard root_commit`
- `git clean -fd`
- cleanup container
- write final `metadata.json`

`valid_parent` is computed from whether eval ran and whether score exists (or meta patches are present).

---

## 7) How task evaluation works end-to-end

### 7.1 Harness path (`domains/harness.py`)

For classic dataset domains (`search_arena`, `paper_review`, IMO variants):

1. Domain utility module loaded dynamically (`domains.<domain>.utils` or IMO variant path).
2. Determine `format_input_dict`, question ID column, and model.
3. Load TaskAgent class from path/module (`load_task_agent`).
4. Load dataset CSV; optionally subset and resume existing predictions.
5. ThreadPool executes per-row `run_agent(...)`:
   - instantiate `TaskAgent` with row-specific chat history file.
   - format row input.
   - call `TaskAgent.forward(inputs)`.
6. Save checkpointed/final `predictions.csv`.

### 7.2 TaskAgent internals (`task_agent.py`)

`TaskAgent.forward(inputs)`:
- Builds instruction embedding input dict.
- Calls `chat_with_agent(...)` (no explicit tool set requested here).
- Extracts `<json>{"response": ...}</json>` from final model output.
- Returns `(prediction, message_history)`.

### 7.3 Report path (`domains/report.py`)

For classification-style domains:
- Loads predictions.
- normalizes text.
- computes overall accuracy, per-label precision/recall, distributions.
- writes `report.json`.

Domain-specific score key is read by the loop using `get_domain_score_key(...)`.

---

## 8) Parent selection and archive dynamics

Archive is append-only by generation ID (`archive.jsonl`).

Parent selection implementations:

1. `select_parent(...)` in `utils/gl_utils.py` (default loop path)
   - builds candidate set from valid parents with available per-domain scores.
   - supports methods: `random`, `latest`, `best`, `score_prop`, `score_child_prop`.
   - `score_child_prop` balances score and child-count penalty.

2. `select_next_parent(...)` in `select_next_parent.py`
   - alternate, externally editable parent-selection function.
   - invoked in container via `utils.run_select_next_parent` when `edit_select_parent=True`.

Metadata fields (`valid_parent`, `can_select_next_parent`, lineage) determine eligibility.

---

## 9) Ensemble scoring flow

When optimize mode includes ensemble and domain is ensemble-capable:

1. `generate_loop` calls `get_ensemble_scores_container(...)`.
2. Container runs `python -m utils.run_ensemble` for each split.
3. `utils.run_ensemble.get_ensemble_score(...)`:
   - loads dataset
   - for each row calls `ensemble(...)`
4. `ensemble(...)`:
   - loads archive
   - chooses generation with best score for requested split
   - reads that generation’s predictions CSV
   - returns matching prediction for question id
5. report generated and copied back (`report_ensemble_<domain>_<split>.json`).

---

## 10) LLM and tool-calling internals

`agent/llm_withtools.py` drives tool-enabled conversation:

1. Build tool-use system prompt from loaded tools.
2. Call `get_response_from_llm(...)`.
3. Parse `<json>{"tool_name":...,"tool_input":...}</json>` blocks.
4. Execute tool function.
5. Feed tool output back into model.
6. Repeat until no tool call (or max tool calls reached).

`agent/llm.py` handles provider differences:
- model constants
- GPT-5 token parameter compatibility
- haiku token cap handling
- backoff retry wrapper

---

## 11) Failure handling patterns

Common safety patterns in runtime flow:

- Extensive `try/except/finally` around container lifecycle.
- Reset/clean repo inside container before container teardown.
- If parent selection in editable mode fails, retries up to max attempts.
- Evaluation threading cancels unfinished futures when one fails.
- Missing/malformed scores resolve to `None` and are filtered from parent candidates.

This design minimizes contamination across generations and keeps each generation mostly isolated and reproducible.

---

## 12) Concrete step-by-step trace for one generation

Assume parent is `gen_3`, next child is `gen_4`:

1. `generate_loop` selects parent `3`.
2. `generate(...)` creates `outputs/.../gen_4/`.
3. Container starts from image `hyperagents`.
4. Patch lineage from `gen_3/metadata.json` is resolved.
5. Lineage patches are applied and committed in container.
6. `run_meta_agent.py` runs, meta agent edits code, writes `model_patch.diff`.
7. `agent_output` copied from container to local `gen_4/agent_output`.
8. If patch exists, evaluate task agent over configured domain(s).
9. Reports copied back to local `gen_4/<domain>_eval*/report.json`.
10. `gen_4/metadata.json` is finalized.
11. `archive.jsonl` appended with `4`.
12. Optional ensemble reports generated.
13. Plots updated.
14. Next parent selected.

---

## 13) Important extension points

If you want to change behavior, these are the highest-impact points:

- Meta-agent editing policy: `meta_agent.py` and `agent/llm_withtools.py`.
- Task answer extraction format: `task_agent.py` + `utils/common.extract_jsons`.
- Parent selection strategy: `utils/gl_utils.select_parent` or `select_next_parent.py`.
- Staged/full eval gating: `generate_loop.py` and `utils/domain_utils.py`.
- Domain scoring semantics: `utils/domain_utils.get_domain_score_key` and each domain report format.
- Ensemble policy: `ensemble.py` and `utils/run_ensemble.py`.

---

## 14) Mental model summary

Think of HyperAgents as three loops nested together:

1. **Edit loop** (meta agent creates code diff).
2. **Evaluate loop** (task agent is benchmarked on domain data).
3. **Evolution loop** (archive grows, parent is selected, process repeats).

Docker isolation + patch lineage + metadata tracking are the backbone that make this recursive self-improvement process operational.
