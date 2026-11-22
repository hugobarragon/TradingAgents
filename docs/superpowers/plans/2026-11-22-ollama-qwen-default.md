# Ollama Qwen Default Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make Ollama and the installed Qwen 3.6 MoE / Qwen 3.8 models the
fork's built-in default while preserving environment overrides.

**Architecture:** Change only built-in provider/model defaults, Ollama catalog
entries, focused tests, and user-facing documentation.

**Tech Stack:** Python 3.11+, pytest, Ruff, Ollama OpenAI-compatible API.

**Spec:** `docs/superpowers/specs/2026-11-22-ollama-qwen-default-design.md`

## Global Constraints

- Provider default: `ollama`.
- Quick model: `qwen3.6:35b-a3b-coding-nvfp4`.
- Deep model: `qwen3.8:27b-mlx`.
- Ollama endpoint: `http://localhost:11434/v1` when `OLLAMA_BASE_URL` is unset.
- Hosted/provider environment overrides remain authoritative.
- Ollama models stay custom-ID compatible.

## Review Focus

- A blank provider/model environment still yields Ollama and the exact Qwen
  tags; test in `tests/test_env_overrides.py`.
- Explicit provider/model environment values replace built-in defaults; test
  with `TRADINGAGENTS_*` overrides.
- Both exact Qwen tags appear in the Ollama picker; test in
  `tests/test_model_validation.py`.
- An Ollama endpoint can be overridden through `OLLAMA_BASE_URL`; existing
  `tests/test_ollama_base_url.py` owns this behavior.

---

### Task 1: Add failing default and catalog tests

**Files:**
- Modify: `tests/test_env_overrides.py`
- Modify: `tests/test_model_validation.py`

- [ ] **Step 1: Update the no-env default assertion**

Assert `llm_provider == "ollama"`, `quick_think_llm == "qwen3.6:35b-a3b-coding-nvfp4"`,
and `deep_think_llm == "qwen3.8:27b-mlx"`.

- [ ] **Step 2: Add Ollama catalog assertions**

Assert `qwen3.6:35b-a3b-coding-nvfp4` is offered for quick mode and
`qwen3.8:27b-mlx` is offered for deep mode.

- [ ] **Step 3: Run the focused tests and confirm RED**

Run:
`python3 -m pytest -q tests/test_env_overrides.py tests/test_model_validation.py`

Expected: failures for the old OpenAI defaults and missing exact Ollama IDs.

### Task 2: Implement Ollama and Qwen defaults

**Files:**
- Modify: `tradingagents/default_config.py`
- Modify: `tradingagents/llm_clients/model_catalog.py`

- [ ] **Step 1: Set built-in provider and tier model defaults**

Set `llm_provider` to `ollama`, `quick_think_llm` to
`qwen3.6:35b-a3b-coding-nvfp4`, and `deep_think_llm` to
`qwen3.8:27b-mlx`.

- [ ] **Step 2: Add both exact tags to the Ollama catalog**

Offer the exact quick and deep tags while retaining `Custom model ID`.

- [ ] **Step 3: Run focused tests and confirm GREEN**

Run:
`python3 -m pytest -q tests/test_env_overrides.py tests/test_model_validation.py tests/test_ollama_base_url.py`

Expected: all focused tests pass.

- [ ] **Step 4: Commit**

Run:
```bash
git add tradingagents/default_config.py tradingagents/llm_clients/model_catalog.py tests/test_env_overrides.py tests/test_model_validation.py
git commit -m "feat: default to local Ollama Qwen models"
```

### Task 3: Document and verify

**Files:**
- Modify: `.env.example`
- Modify: `README.md`

- [ ] **Step 1: Document local defaults and overrides**

Update examples to show Ollama as the built-in provider, the two exact Qwen
tags, the default endpoint, and `OLLAMA_BASE_URL` override.

- [ ] **Step 2: Run full tests and Ruff**

Run:
```bash
python3 -m pytest -q
python3 -m ruff check .
```

Expected: full suite passes and Ruff passes.

- [ ] **Step 3: Commit documentation and push branch update**

Commit with a documentation-only message, then push the feature branch.
