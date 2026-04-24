# Custom LLM Endpoints Configuration

## Overview

GBrain supports custom API endpoints for both OpenAI-compatible and Anthropic-compatible LLM providers. This allows you to use alternative providers like OpenRouter, Azure OpenAI, local LLM servers (Ollama, LM Studio), and other OpenAI/Anthropic API-compatible services.

## Configuration Methods

You can configure custom endpoints via two methods (environment variables take precedence):

### 1. Environment Variables (highest priority)

```bash
# OpenAI-compatible endpoint (for embeddings)
export OPENAI_API_KEY="sk-... or your custom key"
export OPENAI_BASE_URL="https://your-custom-endpoint/v1"
export OPENAI_EMBEDDING_MODEL="text-embedding-3-large"  # optional override

# Anthropic-compatible endpoint (for query expansion & subagents)
export ANTHROPIC_API_KEY="sk-... or your custom key"
export ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1"

# Model overrides (optional)
export ANTHROPIC_EXPANSION_MODEL="anthropic/claude-haiku-4-5-20251001"
export ANTHROPIC_SUBAGENT_MODEL="anthropic/claude-sonnet-4-6"
```

### 2. Config File (`~/.gbrain/config.json`)

```json
{
  "engine": "pglite",
  "openai_api_key": "sk-...",
  "openai_base_url": "https://your-custom-endpoint/v1",
  "openai_embedding_model": "text-embedding-3-large",
  "anthropic_api_key": "sk-or-xxxxx",
  "anthropic_base_url": "https://openrouter.ai/api/v1",
  "anthropic_expansion_model": "anthropic/claude-haiku-4-5-20251001",
  "anthropic_subagent_model": "anthropic/claude-sonnet-4-6"
}
```

## Supported Use Cases

### OpenRouter

OpenRouter provides a unified API for many LLM providers.

```bash
# Set API key
export OPENROUTER_API_KEY="sk-or-xxxxx"

# For OpenAI-compatible services (embeddings)
export OPENAI_API_KEY="$OPENROUTER_API_KEY"
export OPENAI_BASE_URL="https://openrouter.ai/api/v1"

# For Anthropic-compatible services (expansion, subagents)
export ANTHROPIC_API_KEY="$OPENROUTER_API_KEY"
export ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1"
```

OpenRouter model naming convention:
- `anthropic/claude-haiku-4-5-20251001`
- `anthropic/claude-sonnet-4-6`
- `google/gemini-flash-2.0` (cheaper alternative)

### Local LLM Servers (Ollama, LM Studio)

```bash
# Ollama (local)
export OPENAI_BASE_URL="http://localhost:11434/v1"
export OPENAI_API_KEY="ollama"  # dummy key, Ollama doesn't require real key

# LM Studio (local)
export OPENAI_BASE_URL="http://localhost:1234/v1"
export OPENAI_API_KEY="lm-studio"
```

### Azure OpenAI

```bash
export OPENAI_BASE_URL="https://<your-resource>.openai.azure.com/openai/deployments/<deployment-name>"
export OPENAI_API_KEY="<azure-api-key>"
```

## Modified Files

The following files were modified to support custom endpoints (cross-AI reviewed):

| File | Changes |
|------|---------|
| `src/core/config.ts` | Added `openai_base_url`, `openai_embedding_model`, `anthropic_base_url`, `anthropic_expansion_model`, `anthropic_subagent_model` fields to `GBrainConfig` interface. Environment variable handling for all new fields. Restored `storage?` field. |
| `src/core/embedding.ts` | Added `loadConfig()` import. Modified `getClient()` to use `baseURL` from env var or config. `embedBatch()` now reads `openai_embedding_model` from config/env. `embedBatchWithRetry()` accepts model parameter. |
| `src/core/search/expansion.ts` | Added `loadConfig()` import. Modified `getClient()` to use `baseURL` and `apiKey` from env var or config. Model now reads from `ANTHROPIC_EXPANSION_MODEL` env var or `anthropic_expansion_model` config field. |
| `src/core/minions/handlers/subagent.ts` | Modified `makeSubagentHandler()` to read `baseURL` and `apiKey` from env var or config. `DEFAULT_MODEL` now reads from `ANTHROPIC_SUBAGENT_MODEL` env var or `anthropic_subagent_model` config field. Restored `client` and `rateLeaseKey` variables that were accidentally removed. |

## Model Configuration Priority

### Embedding Model (OpenAI-compatible)
1. Environment variable: `OPENAI_EMBEDDING_MODEL`
2. Config file: `openai_embedding_model`
3. Default: `text-embedding-3-large` (1536 dimensions)

### Expansion Model (Anthropic-compatible)
1. Environment variable: `ANTHROPIC_EXPANSION_MODEL`
2. Config file: `anthropic_expansion_model`
3. Default: `claude-haiku-4-5-20251001`

### Subagent Model (Anthropic-compatible)
1. Environment variable: `ANTHROPIC_SUBAGENT_MODEL`
2. Config file: `anthropic_subagent_model`
3. Default: `claude-sonnet-4-6`

## Recommended Models for Query Expansion

| Model | Speed | Cost (per M tokens) | Recommendation |
|-------|------|---------------------|----------------|
| `claude-haiku-4-5-20251001` | Fastest | $1.00 / $5.00 | Default, optimal for expansion |
| `google/gemini-flash-2.0` (via OpenRouter) | Very fast | $0.075 / $0.30 | Cheapest alternative |
| `claude-sonnet-4-6` | Fast | $3.00 / $15.00 | Overkill for expansion |

## Notes

- When using OpenRouter, model names must include the provider prefix (e.g., `anthropic/claude-haiku-4-5-20251001`).
- The `baseURL` must include the API version path (e.g., `/v1`) when required by the provider.
- For local servers like Ollama, a dummy API key is typically sufficient.
- Changing embedding models may require re-embedding your brain data due to dimension mismatches.
