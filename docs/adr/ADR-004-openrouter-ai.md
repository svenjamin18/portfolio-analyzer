# ADR-004: OpenRouter for AI Chat

**Date:** 2026-05-10
**Status:** Accepted

## Decision

Use OpenRouter API with model `anthropic/claude-sonnet-4-5` for the AI portfolio advisor chat.

## Context

The dashboard needs an AI chat that understands the current portfolio (positions, sector/region distribution, risk warnings) and can give personalised investment recommendations. Sven has an existing OpenRouter API key.

## Options Considered

| Option | Pros | Cons |
|--------|------|------|
| OpenRouter + claude-sonnet-4-5 | Existing key, access to Claude, one API for many models | Slightly higher latency than direct Anthropic |
| Anthropic API directly | Fastest Claude access, best quality | No existing key |
| OpenAI GPT-4o | Fast, capable | Less strong for financial reasoning than Claude |

## Reasoning

Sven already has an OpenRouter key. OpenRouter is OpenAI-API-compatible so the OpenAI SDK works with a custom `baseURL` — minimal code difference from a direct provider. `claude-sonnet-4-5` is fast and capable for portfolio analysis tasks.

## Consequences

- Use OpenAI SDK: `new OpenAI({ baseURL: "https://openrouter.ai/api/v1", apiKey: process.env.OPENROUTER_API_KEY })`
- If direct Anthropic access is added later, swap `baseURL` and SDK in one place
- Portfolio context (positions, computed metrics, risk warnings) is injected as the system prompt
- AI asks clarifying questions (investment horizon, risk tolerance, preferences) before making recommendations
