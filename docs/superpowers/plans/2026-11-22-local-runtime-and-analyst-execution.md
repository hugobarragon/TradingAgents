# Local Runtime and Analyst Execution Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task. Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move local generated data into the checkout, save reports in
`docs/reports/`, and make analyst concurrency opt-in.

**Architecture:** Path helpers in `tradingagents/default_config.py` provide
repo-stable defaults while retaining Docker and explicit overrides. GraphSetup
selects either a sequential chain or the existing fan-out topology, and
TradingAgentsGraph supplies that choice from configuration.

**Tech Stack:** Python 3.11+, pytest, LangGraph StateGraph, Ruff, Git.

**Spec:** `docs/superpowers/specs/2026-11-22-local-runtime-and-analyst-execution-design.md`

## Global Constraints

- The report writer must produce `<repo>/docs/reports/` without changing report
  serialization.
- Cache, memory, and preferences use `<repo>/.tradingagents/` locally.
- Docker and path-specific environment overrides keep working.
- ChatGPT authentication remains under the user's home directory.
- Analysts are sequential unless `TRADINGAGENTS_PARALLEL_ANALYSTS` is true.
- Keep generated runtime files out of Git.
- Leave sentiment-provider reliability out of scope.

## Review Focus

- Running the CLI from another working directory still resolves the same repo
  paths; cover with `test_default_runtime_paths_are_repo_local`.
- Blank path variables keep valid local defaults; cover with
  `test_empty_path_value_keeps_the_default_path`.
- Docker keeps writing to the mounted user-data directory; cover with
  `test_docker_runtime_paths_use_the_mounted_data_directory`.
- Older config dictionaries without `parallel_analysts` remain valid; cover
  with `test_missing_analyst_parallelism_setting_defaults_to_sequential`.
- Parallel execution remains available without changing downstream graph
  stages; cover with `test_the_analysts_run_at_the_same_time`.

---

### Task 1: Local Runtime Paths

**Files:**
- Modify: `tradingagents/default_config.py`
- Test: `tests/test_env_overrides.py`

**Interfaces:**
- Consumes: `TRADINGAGENTS_IN_DOCKER` and the three existing path environment
  variables.
- Produces: `get_runtime_data_home() -> str`, local `results_dir=<repo>/docs`,
  `data_cache_dir=<repo>/.tradingagents/cache`, and local memory path
  `<repo>/.tradingagents/memory/trading_memory.md`.

- [ ] **Step 1: Add failing local, Docker, and blank-path tests**

Add assertions that local results default to `Path(default_config_module.__file__).resolve().parents[1] / "docs"`; local cache and memory default below `.tradingagents`; Docker paths remain below `_TRADINGAGENTS_HOME`; and blank path variables retain the same defaults.

- [ ] **Step 2: Run the path tests to verify they fail**

Run: `python -m pytest -q tests/test_env_overrides.py`

Expected: failures for missing `get_runtime_data_home`/the old home-based paths.

- [ ] **Step 3: Implement the runtime path helpers and path defaults**

Add `Path`-based repo-root resolution, `get_runtime_data_home()`, and a distinct
local `results_dir` default in `build_default_config()`. Preserve environment
overrides and Docker behavior.

- [ ] **Step 4: Run the focused path tests**

Run: `python -m pytest -q tests/test_env_overrides.py`

Expected: all tests pass.

- [ ] **Step 5: Commit**

Run:
```bash
git add tradingagents/default_config.py tests/test_env_overrides.py
git commit -m "feat: use repository-local runtime paths"
```

### Task 2: Repository-local CLI Preferences

**Files:**
- Modify: `cli/prefs.py`
- Test: `tests/test_cli_prefs.py`

**Interfaces:**
- Consumes: `get_runtime_data_home()` from Task 1.
- Produces: `_PREFS_PATH=<runtime-data-home>/cli_prefs.json`.

- [ ] **Step 1: Add a failing preference-location test**

Add `test_default_preferences_use_repo_local_runtime_data_home`, reloading
`cli.prefs` after changing the working directory and removing the Docker flag,
then assert `_PREFS_PATH` equals the checkout's `.tradingagents/cli_prefs.json`.

- [ ] **Step 2: Run the preference tests to verify the new test fails**

Run: `python -m pytest -q tests/test_cli_prefs.py::test_default_preferences_use_repo_local_runtime_data_home`

Expected: FAIL because `_PREFS_PATH` still points to the user's home.

- [ ] **Step 3: Derive `_PREFS_PATH` from `get_runtime_data_home()`**

Replace the direct `expanduser("~")` path construction with the helper import
and preserve all load/save behavior.

- [ ] **Step 4: Run all CLI preference tests**

Run: `python -m pytest -q tests/test_cli_prefs.py`

Expected: all tests pass.

- [ ] **Step 5: Commit**

Run:
```bash
git add cli/prefs.py tests/test_cli_prefs.py
git commit -m "feat: store CLI preferences with local runtime data"
```

### Task 3: Sequential Analyst Topology

**Files:**
- Modify: `tradingagents/graph/setup.py`
- Test: `tests/test_graph_end_to_end.py`

**Interfaces:**
- Consumes: `parallel_analysts: bool = False` in `GraphSetup.__init__`.
- Produces: a sequential analyst/memory chain by default or the existing
  fan-out when explicitly enabled.

- [ ] **Step 1: Add graph-mode regression tests**

Update the existing concurrent test to request `parallel_analysts=True`; add a
sequential success test and a missing-setting fallback test. Run the settlement
timing case in parallel mode because it specifically asserts overlap.

- [ ] **Step 2: Run the graph-mode tests to verify the sequential behavior fails**

Run:
`python -m pytest -q tests/test_graph_end_to_end.py::test_selected_analysts_can_run_sequentially tests/test_graph_end_to_end.py::test_missing_analyst_parallelism_setting_defaults_to_sequential`

Expected: failures because `GraphSetup` does not yet accept the flag or
sequential mode.

- [ ] **Step 3: Add `parallel_analysts` to `GraphSetup`**

Store the flag on the instance. With false, connect `START` to the first
analyst, chain remaining analysts and optional memory in order, then connect
the last step to Bull Researcher. With true, retain the existing fan-out and
join.

- [ ] **Step 4: Pass the flag from `TradingAgentsGraph`**

Construct `GraphSetup` with
`parallel_analysts=self.config.get("parallel_analysts", False)` so older
external config dictionaries remain sequential.

- [ ] **Step 5: Run the full graph end-to-end test file**

Run: `python -m pytest -q tests/test_graph_end_to_end.py`

Expected: all graph tests pass in both modes.

- [ ] **Step 6: Commit**

Run:
```bash
git add tradingagents/graph/setup.py tradingagents/graph/trading_graph.py tests/test_graph_end_to_end.py
git commit -m "feat: make analyst graph execution sequential by default"
```

### Task 4: Analyst Concurrency Environment Variable

**Files:**
- Modify: `tradingagents/default_config.py`
- Test: `tests/test_env_overrides.py`

**Interfaces:**
- Consumes: `TRADINGAGENTS_PARALLEL_ANALYSTS`.
- Produces: boolean `parallel_analysts` in `DEFAULT_CONFIG`.

- [ ] **Step 1: Add failing concurrency configuration tests**

Assert `parallel_analysts` defaults false and becomes true only for accepted
true spellings. Reuse the existing boolean coercion table for behavior.

- [ ] **Step 2: Run the concurrency tests to verify they fail**

Run:
`python -m pytest -q tests/test_env_overrides.py::test_analysts_run_sequentially_by_default tests/test_env_overrides.py::test_parallel_analysts_can_be_enabled_explicitly`

Expected: FAIL because the config key is absent.

- [ ] **Step 3: Add the environment override and default**

Add `TRADINGAGENTS_PARALLEL_ANALYSTS` to `_ENV_OVERRIDES` and add
`parallel_analysts: False` beside the analyst/tool execution settings.

- [ ] **Step 4: Run the environment override suite**

Run: `python -m pytest -q tests/test_env_overrides.py`

Expected: all tests pass.

- [ ] **Step 5: Commit**

Run:
```bash
git add tradingagents/default_config.py tests/test_env_overrides.py
git commit -m "feat: allow opt-in parallel analyst execution"
```

### Task 5: Display Wording

**Files:**
- Modify: `cli/display.py`
- Test: `tests/test_cli_display.py`

**Interfaces:**
- Consumes: selected-analyst reports from either execution topology.
- Produces: status copy that is valid for sequential and parallel modes.

- [ ] **Step 1: Add a wording regression test**

Assert `update_analyst_statuses.__doc__` does not claim analysts "run
together" or add an equivalent user-visible wording assertion.

- [ ] **Step 2: Run the wording test to verify it fails**

Run: `python -m pytest -q tests/test_cli_display.py -q`

Expected: the new wording assertion fails.

- [ ] **Step 3: Make the status and tracker wording topology-neutral**

Remove parallel-only wording from analyst status/tracker docstrings without
changing status transitions.

- [ ] **Step 4: Run the display tests**

Run: `python -m pytest -q tests/test_cli_display.py`

Expected: all display tests pass.

- [ ] **Step 5: Commit**

Run:
```bash
git add cli/display.py tests/test_cli_display.py
git commit -m "fix: make analyst display wording concurrency-neutral"
```

### Task 6: Local Setup Documentation and Ignores

**Files:**
- Modify: `.gitignore`
- Modify: `.env.example`
- Modify: `README.md`

**Interfaces:**
- Consumes: path and concurrency behavior from Tasks 1-5.
- Produces: user-facing examples and clean generated-file status.

- [ ] **Step 1: Add ignore rules for local runtime artifacts**

Ignore `.tradingagents/`, `docs/reports/`, and per-ticker
`TradingAgentsStrategy_logs/` directories under `docs/`, without ignoring
hand-written design and plan documents.

- [ ] **Step 2: Document local and Docker paths**

Update `.env.example` and README to state that reports default to
`docs/reports/`, runtime cache/memory/preferences use `.tradingagents/`,
analysts are sequential, and the parallel environment variable opts in.
Document that ChatGPT credentials remain in the home directory and Docker uses
its mounted volume.

- [ ] **Step 3: Verify documentation and ignore behavior**

Run: `rtk git check-ignore -v .tradingagents/test docs/reports/test`

Expected: both paths are ignored by the new rules.

- [ ] **Step 4: Run Ruff and the focused regression suite**

Run:
```bash
python -m pytest -q tests/test_env_overrides.py tests/test_cli_prefs.py tests/test_cli_display.py tests/test_graph_end_to_end.py
python -m ruff check .
```

Expected: all commands pass.

- [ ] **Step 5: Commit**

Run:
```bash
git add .gitignore .env.example README.md
git commit -m "docs: describe local fork runtime defaults"
```

### Task 7: Full Verification and Cleanup

**Files:**
- Verify: repository worktree

- [ ] **Step 1: Run the full test suite**

Run: `python -m pytest -q`

Expected: all non-integration tests pass.

- [ ] **Step 2: Run the full linter**

Run: `python -m ruff check .`

Expected: no Ruff errors.

- [ ] **Step 3: Inspect Git status**

Run: `rtk git status --short --branch`

Expected: only the pre-existing untracked `AGENTS.md` and `FORK_HANDOFF.md`, or
no unrelated generated runtime files.

- [ ] **Step 4: Record the design and plan commit only if not already committed**

Ensure the two `docs/superpowers` files are included in the setup branch in a
single documentation commit if prior feature commits do not already contain
them.
