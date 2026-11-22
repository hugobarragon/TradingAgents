# Ollama Qwen Default Design

## Intent

Make this fork use the user's local Ollama setup by default, while retaining
explicit environment-based overrides for hosted providers or another endpoint.

## Goals

- Default `llm_provider` to `ollama`.
- Default the quick tier to `qwen3.6:35b-a3b-coding-nvfp4`, the installed
  Qwen Mixture-of-Experts model.
- Default the deep tier to `qwen3.8:27b-mlx`, the installed Qwen 3.8 model.
- Keep `OLLAMA_BASE_URL` as `http://localhost:11434/v1` when unset.
- Show both exact models in the Ollama model picker.
- Preserve explicit `TRADINGAGENTS_LLM_PROVIDER`,
  `TRADINGAGENTS_QUICK_THINK_LLM`, `TRADINGAGENTS_DEEP_THINK_LLM`, and
  `TRADINGAGENTS_LLM_BACKEND_URL` overrides.

## Non-goals

- Pulling or installing Ollama models.
- Changing Ollama endpoint detection.
- Changing graph concurrency or runtime-data behavior.
- Automatically discovering models from the local Ollama API.

## Constraints

- Ollama model IDs are exact strings, including tags and quantization suffixes.
- The quick and deep tiers may use different local model IDs.
- Cloud/provider environment overrides must still take precedence.
- Ollama remains keyless and uses the existing OpenAI-compatible client.

## Approach

Update the built-in defaults and Ollama model catalog, then cover both the
default values and explicit environment overrides with tests. Document the
default in `.env.example` and README.
