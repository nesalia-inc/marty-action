# Using Marty with OpenRouter

Guide for configuring Marty to use OpenRouter instead of direct providers.

## Overview

OpenRouter provides:
- **Provider failover** - Automatic routing between providers
- **Budget controls** - Set spending limits for teams
- **Usage analytics** - Track costs and usage patterns
- **Rate limit handling** - Automatic fallback when providers are limited

---

## Quick Start: Z.ai → OpenRouter Migration

### Current Configuration (Z.ai)

```yaml
env:
  ANTHROPIC_BASE_URL: https://api.z.ai/api/anthropic
  ANTHROPIC_AUTH_TOKEN: ${{ secrets.ZAI_API_KEY }}
  ANTHROPIC_DEFAULT_SONNET_MODEL: glm-4.7
  ANTHROPIC_DEFAULT_HAIKU_MODEL: glm-4.5-air
  ANTHROPIC_DEFAULT_OPUS_MODEL: glm-4.7
```

### New Configuration (OpenRouter)

```yaml
env:
  ANTHROPIC_BASE_URL: https://openrouter.ai/api
  ANTHROPIC_AUTH_TOKEN: ${{ secrets.OPENROUTER_API_KEY }}
  ANTHROPIC_API_KEY: ""  # Important: Must be explicitly empty
  ANTHROPIC_DEFAULT_SONNET_MODEL: anthropic/claude-sonnet-4-20250514
  ANTHROPIC_DEFAULT_HAIKU_MODEL: anthropic/claude-3-5-haiku-20241022
  ANTHROPIC_DEFAULT_OPUS_MODEL: anthropic/claude-sonnet-4-20250514
```

---

## Step-by-Step Migration

### Step 1: Get OpenRouter API Key

1. Go to https://openrouter.ai/
2. Sign up or log in
3. Navigate to **API Keys** section
4. Create a new API key
5. Copy the key

### Step 2: Add GitHub Secret

In your target repository (not marty-action):

1. Go to **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Name: `OPENROUTER_API_KEY`
4. Value: Your OpenRouter API key from Step 1
5. Click **Add secret**

### Step 3: Update Workflows

Replace Z.ai configuration with OpenRouter in all your workflow files.

#### Before (Z.ai):

```yaml
- uses: nesalia-inc/marty-action@v1.0.0
  with:
    github_token: ${{ steps.marty-token.outputs.token }}
    prompt: |
      Your prompt here...
  env:
    ANTHROPIC_BASE_URL: https://api.z.ai/api/anthropic
    ANTHROPIC_AUTH_TOKEN: ${{ secrets.ZAI_API_KEY }}
    ANTHROPIC_DEFAULT_SONNET_MODEL: glm-4.7
    ANTHROPIC_DEFAULT_HAIKU_MODEL: glm-4.5-air
    ANTHROPIC_DEFAULT_OPUS_MODEL: glm-4.7
```

#### After (OpenRouter):

```yaml
- uses: nesalia-inc/marty-action@v1.0.0
  with:
    github_token: ${{ steps.marty-token.outputs.token }}
    prompt: |
      Your prompt here...
  env:
    ANTHROPIC_BASE_URL: https://openrouter.ai/api
    ANTHROPIC_AUTH_TOKEN: ${{ secrets.OPENROUTER_API_KEY }}
    ANTHROPIC_API_KEY: ""  # Important: Must be explicitly empty
    ANTHROPIC_DEFAULT_SONNET_MODEL: anthropic/claude-sonnet-4-20250514
    ANTHROPIC_DEFAULT_HAIKU_MODEL: anthropic/claude-3-5-haiku-20241022
    ANTHROPIC_DEFAULT_OPUS_MODEL: anthropic/claude-sonnet-4-20250514
```

### Step 4: Verify

Run a workflow and check:
- ✅ No authentication errors
- ✅ Responses are generated
- ✅ Check OpenRouter dashboard for usage

---

## Model Mappings

### OpenRouter Model Names

OpenRouter uses OpenAI-style model names. For Claude models:

| Tier | Z.ai Model | OpenRouter Model |
|------|------------|------------------|
| **Haiku** | `glm-4.5-air` | `anthropic/claude-3-5-haiku-20241022` |
| **Sonnet** | `glm-4.7` | `anthropic/claude-sonnet-4-20250514` |
| **Opus** | `glm-4.7` | `anthropic/claude-sonnet-4-20250514` |

**Note:** OpenRouter doesn't have a direct "Opus" equivalent for Claude 4. Use Sonnet 4 for all complex tasks.

### Alternative Models on OpenRouter

You can also use other models through OpenRouter:

```yaml
# GPT-4
ANTHROPIC_DEFAULT_SONNET_MODEL: openai/gpt-4-turbo

# Gemini
ANTHROPIC_DEFAULT_SONNET_MODEL: google/gemini-pro-1.5

# Llama 3
ANTHROPIC_DEFAULT_SONNET_MODEL: meta-llama/llama-3-70b
```

**Warning:** Changing models may affect prompt compatibility and output quality.

---

## Updated Workflow Examples

### Issue Triage with OpenRouter

```yaml
name: Marty Issue Triage

on:
  issues:
    types: [opened, edited]

permissions:
  contents: read
  issues: write
  id-token: write

jobs:
  triage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Generate Marty token
        id: marty-token
        uses: actions/create-github-app-token@v2
        with:
          app-id: ${{ secrets.MARTY_APP_ID }}
          private-key: ${{ secrets.MARTY_APP_PRIVATE_KEY }}

      - uses: nesalia-inc/marty-action@v1.0.0
        with:
          github_token: ${{ steps.marty-token.outputs.token }}
          prompt: |
            REPO: ${{ github.repository }}
            ISSUE NUMBER: ${{ github.event.issue.number }}
            # ... rest of prompt
        env:
          ANTHROPIC_BASE_URL: https://openrouter.ai/api
          ANTHROPIC_AUTH_TOKEN: ${{ secrets.OPENROUTER_API_KEY }}
          ANTHROPIC_API_KEY: ""
          ANTHROPIC_DEFAULT_SONNET_MODEL: anthropic/claude-sonnet-4-20250514
          ANTHROPIC_DEFAULT_HAIKU_MODEL: anthropic/claude-3-5-haiku-20241022
          ANTHROPIC_DEFAULT_OPUS_MODEL: anthropic/claude-sonnet-4-20250514
```

### Issue Response with OpenRouter

```yaml
- uses: nesalia-inc/marty-action@v1.0.0
  with:
    github_token: ${{ steps.marty-token.outputs.token }}
    prompt: |
      # ... prompt with progress display
  env:
    ANTHROPIC_BASE_URL: https://openrouter.ai/api
    ANTHROPIC_AUTH_TOKEN: ${{ secrets.OPENROUTER_API_KEY }}
    ANTHROPIC_API_KEY: ""
    ANTHROPIC_DEFAULT_SONNET_MODEL: anthropic/claude-sonnet-4-20250514
    ANTHROPIC_DEFAULT_HAIKU_MODEL: anthropic/claude-3-5-haiku-20241022
    ANTHROPIC_DEFAULT_OPUS_MODEL: anthropic/claude-sonnet-4-20250514
```

### Implementation with OpenRouter

```yaml
- uses: nesalia-inc/marty-action@v1.0.0
  with:
    github_token: ${{ steps.marty-token.outputs.token }}
    prompt: |
      # ... implementation prompt
  env:
    ANTHROPIC_BASE_URL: https://openrouter.ai/api
    ANTHROPIC_AUTH_TOKEN: ${{ secrets.OPENROUTER_API_KEY }}
    ANTHROPIC_API_KEY: ""
    ANTHROPIC_DEFAULT_SONNET_MODEL: anthropic/claude-sonnet-4-20250514
```

### PR Review with OpenRouter

```yaml
- uses: nesalia-inc/marty-action@v1.0.0
  with:
    github_token: ${{ steps.marty-token.outputs.token }}
    prompt: |
      # ... review prompt
  env:
    ANTHROPIC_BASE_URL: https://openrouter.ai/api
    ANTHROPIC_AUTH_TOKEN: ${{ secrets.OPENROUTER_API_KEY }}
    ANTHROPIC_API_KEY: ""
    ANTHROPIC_DEFAULT_SONNET_MODEL: anthropic/claude-sonnet-4-20250514
```

---

## Cost Comparison

### Z.ai Pricing (Estimated)

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|-----------------------|
| GLM-4.5-air | ~$0.10 | ~$0.10 |
| GLM-4.7 | ~$0.50 | ~$0.50 |

### OpenRouter Pricing (Claude models)

| Model | Input (per 1M tokens) | Output (per 1M tokens) |
|-------|----------------------|-----------------------|
| Claude 3.5 Haiku | $0.80 | $8.00 |
| Claude Sonnet 4 | $3.00 | $15.00 |

**Note:** OpenRouter pricing includes:
- Automatic failover
- Rate limit handling
- Usage analytics
- Budget management

---

## Benefits of OpenRouter

### 1. **High Availability**

```yaml
# If Anthropic API is down, OpenRouter automatically:
# - Tries alternative Anthropic providers
# - Implements exponential backoff
# - Routes to backup providers
```

### 2. **Budget Controls**

Set spending limits in OpenRouter dashboard:
- Per-day limits
- Per-month limits
- Per-project limits
- Alert thresholds

### 3. **Usage Analytics**

Track in OpenRouter dashboard:
- Total spend by project
- Token usage by model
- Request success rate
- Error monitoring

### 4. **Rate Limit Handling**

OpenRouter automatically:
- Queues requests when rate limited
- Implements backoff strategies
- Routes to alternative providers
- Prevents request failures

---

## Advanced Configuration

### Fallback Models

Configure fallback if primary model fails:

```yaml
env:
  # Primary: Claude Sonnet 4
  ANTHROPIC_DEFAULT_SONNET_MODEL: anthropic/claude-sonnet-4-20250514

  # If Sonnet unavailable, fallback to Haiku
  ANTHROPIC_DEFAULT_HAIKU_MODEL: anthropic/claude-3-5-haiku-20241022
```

### Cost Optimization

Use cheaper models for simple tasks:

```yaml
# For quick triage (simple categorization)
env:
  ANTHROPIC_DEFAULT_HAIKU_MODEL: anthropic/claude-3-5-haiku-20241022
  # Override all models to Haiku
  ANTHROPIC_DEFAULT_SONNET_MODEL: anthropic/claude-3-5-haiku-20241022
  ANTHROPIC_DEFAULT_OPUS_MODEL: anthropic/claude-3-5-haiku-20241022
```

### Provider Preferences

Set provider preferences in OpenRouter dashboard:
- Prefer specific providers
- Set cost limits
- Configure latency priorities

---

## Troubleshooting

### Issue: Authentication Errors

**Symptoms:**
```
Error: 401 Unauthorized
```

**Solution:**
```yaml
# Ensure ANTHROPIC_API_KEY is explicitly empty
env:
  ANTHROPIC_API_KEY: ""  # Must be set, not unset
```

---

### Issue: Model Not Found

**Symptoms:**
```
Error: Model 'glm-4.7' not found
```

**Solution:**
```yaml
# Use OpenRouter model names
env:
  ANTHROPIC_DEFAULT_SONNET_MODEL: anthropic/claude-sonnet-4-20250514
```

---

### Issue: High Latency

**Symptoms:** Slow responses from OpenRouter

**Solutions:**
1. Check OpenRouter dashboard for provider status
2. Use faster model (Haiku instead of Sonnet)
3. Reduce `max-turns` in workflow
4. Enable caching in OpenRouter settings

---

### Issue: Cost Overruns

**Symptoms:** Unexpected high bills

**Solutions:**
1. Set budget limits in OpenRouter dashboard
2. Use Haiku for simple tasks
3. Reduce `max-turns`
4. Review usage analytics

---

## Migration Checklist

- [ ] Create OpenRouter account
- [ ] Generate API key
- [ ] Add `OPENROUTER_API_KEY` secret to target repositories
- [ ] Update all workflow files with new configuration
- [ ] Test one workflow (start with issue triage)
- [ ] Verify in OpenRouter dashboard
- [ ] Set budget limits in OpenRouter
- [ ] Monitor usage for first week
- [ ] Update team documentation

---

## Rollback Plan

If you need to rollback to Z.ai:

```yaml
env:
  ANTHROPIC_BASE_URL: https://api.z.ai/api/anthropic
  ANTHROPIC_AUTH_TOKEN: ${{ secrets.ZAI_API_KEY }}
  # Remove ANTHROPIC_API_KEY line
  ANTHROPIC_DEFAULT_SONNET_MODEL: glm-4.7
  ANTHROPIC_DEFAULT_HAIKU_MODEL: glm-4.5-air
  ANTHROPIC_DEFAULT_OPUS_MODEL: glm-4.7
```

---

## Best Practices

### 1. **Start with Haiku for Simple Tasks**

```yaml
# Issue triage → simple categorization
env:
  ANTHROPIC_DEFAULT_SONNET_MODEL: anthropic/claude-3-5-haiku-20241022
```

### 2. **Use Sonnet for Complex Tasks**

```yaml
# Implementation, code review → complex reasoning
env:
  ANTHROPIC_DEFAULT_SONNET_MODEL: anthropic/claude-sonnet-4-20250514
```

### 3. **Monitor OpenRouter Dashboard**

Check daily for first week:
- Request success rate
- Cost per workflow
- Model usage distribution

### 4. **Set Alert Thresholds**

Configure in OpenRouter dashboard:
- Daily spend alerts
- Rate limit warnings
- Error rate alerts

---

## Comparison: Z.ai vs OpenRouter

| Feature | Z.ai | OpenRouter |
|---------|------|------------|
| **Models** | GLM (Chinese) | Claude, GPT, Gemini, Llama |
| **Pricing** | Lower cost | Higher cost, more features |
| **Reliability** | Single provider | Multi-provider failover |
| **Analytics** | Basic | Advanced dashboards |
| **Rate Limits** | Manual handling | Automatic management |
| **Budget Controls** | None | Per-day/month limits |
| **Setup** | Simple API key | API key + configuration |

---

## Environment Variables Reference

| Variable | Z.ai | OpenRouter |
|----------|------|------------|
| `ANTHROPIC_BASE_URL` | `https://api.z.ai/api/anthropic` | `https://openrouter.ai/api` |
| `ANTHROPIC_AUTH_TOKEN` | Z.ai API key | OpenRouter API key |
| `ANTHROPIC_API_KEY` | Z.ai API key | `""` (empty string) |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | `glm-4.7` | `anthropic/claude-sonnet-4-20250514` |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | `glm-4.5-air` | `anthropic/claude-3-5-haiku-20241022` |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | `glm-4.7` | `anthropic/claude-sonnet-4-20250514` |

---

## Support

- **OpenRouter Docs:** https://openrouter.ai/docs
- **OpenRouter Dashboard:** https://openrouter.ai/dashboard
- **Marty Issues:** https://github.com/nesalia-inc/marty-action/issues
- **Claude Code Docs:** https://docs.anthropic.com/en/docs/claude-code

---

## Next Steps

1. **Test with one workflow** - Start with issue triage
2. **Monitor for 1 week** - Check costs and performance
3. **Gather feedback** - Ask team about quality
4. **Migrate all workflows** - Once comfortable
5. **Set up alerts** - Configure budget and error alerts

---

**Quick Reference Card:**

```yaml
# OpenRouter Configuration for Marty
env:
  ANTHROPIC_BASE_URL: https://openrouter.ai/api
  ANTHROPIC_AUTH_TOKEN: ${{ secrets.OPENROUTER_API_KEY }}
  ANTHROPIC_API_KEY: ""  # Critical: Must be empty
  ANTHROPIC_DEFAULT_SONNET_MODEL: anthropic/claude-sonnet-4-20250514
  ANTHROPIC_DEFAULT_HAIKU_MODEL: anthropic/claude-3-5-haiku-20241022
  ANTHROPIC_DEFAULT_OPUS_MODEL: anthropic/claude-sonnet-4-20250514
```
