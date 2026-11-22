# Local Runtime and Analyst Execution Design

## Intent

Adapt this fork for lower-resource local TradingAgents use while keeping the
checkout aligned with the latest upstream behavior. The setup must keep
generated artifacts inside the repository, default analyst execution to one
model workload at a time, and preserve an explicit path back to parallel
analysts.

## Goals

- Save generated local reports under `docs/reports/`.
- Save local cache, memory, and CLI preferences under `.tradingagents/`.
- Resolve local paths from the source checkout, not the process working
  directory.
- Run selected analysts sequentially by default.
- Allow concurrent analysts with `TRADINGAGENTS_PARALLEL_ANALYSTS=true`.
- Preserve explicit result, cache, and memory path overrides.
- Preserve Docker's mounted `~/.tradingagents` runtime data.
- Keep ChatGPT authentication in `~/.tradingagents/chatgpt-auth.json`.
- Deliver the work as small, independently testable commits.

## Non-goals

- Sentiment, Reddit, StockTwits, Yahoo, TLS, retry, and caching work.
- Changes to model/provider selection.
- Moving credentials or generated runtime data into version control.
- Changing report serialization or report filenames.

## Anti-goals

- Making a local report path depend on the shell's current directory.
- Moving ChatGPT credentials into the repository.
- Enabling concurrent analysts accidentally.
- Copying stale code wholesale over newer upstream implementations.
- Committing generated reports, caches, memory, or local environment files.

## Constraints

- Derive the repository root from `tradingagents/default_config.py`.
- Treat a non-empty `TRADINGAGENTS_RESULTS_DIR`, `TRADINGAGENTS_CACHE_DIR`, or
  `TRADINGAGENTS_MEMORY_LOG_PATH` as authoritative.
- Treat blank path variables as unset.
- Detect Docker through `TRADINGAGENTS_IN_DOCKER`, which the image already sets.
- Keep analyst ordering defined by `selected_analysts`.
- Preserve the existing graph and all debate/trader/risk stages.
- Do not modify `AGENTS.md` or `FORK_HANDOFF.md`.

## Approaches

### Adapt the proven local patch

Reapply the existing 11-file patch selectively, update it for the new
`docs/reports/` requirement, and preserve current upstream abstractions. This
is the chosen approach because it has a known shape and limits the fork's
merge surface.

### Rebuild the configuration architecture

A broader runtime-path abstraction would isolate environment detection and
path policy further, but would change more interfaces without changing the
requested behavior. It was rejected.

### Apply the old patch literally

This would be the fastest mechanical port, but it would keep the old report
location and risk overwriting upstream changes. It was rejected.

## Implementation Decisions

- `get_runtime_data_home()` returns `<repo>/.tradingagents` locally and
  `~/.tradingagents` in Docker.
- Local `results_dir` returns `<repo>/docs`; the existing report writer appends
  `reports/`, producing `<repo>/docs/reports/`.
- `GraphSetup` receives `parallel_analysts=False` by default.
- Parallel mode fans analysts and memory out from `START`; sequential mode
  chains them in selected order before the research debate.
- `TradingAgentsGraph` reads `parallel_analysts` with a false fallback so older
  custom configuration dictionaries remain valid.
- `.gitignore` ignores `.tradingagents/`, `docs/reports/`, and generated
  per-ticker strategy logs under `docs/`.

## Verification

- Focused path, CLI preference, display, and graph tests.
- Full default pytest suite.
- Full Ruff check.
- Confirm no generated runtime files appear in `git status`.
