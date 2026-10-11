---
title: "Image generation setup"
summary: Configure image generation models and where generated images are stored.
audience: dev
order: 230
covers:
  - shared/utils/tools/image_tools.py
  - shared/utils/mcp/image_mcp_server.py
  - shared/utils/tools/image_mcp_protocol_handler.py
---

# Image Generation Setup Guide

## Overview

MATE provides image generation tools through both native agent tool functions and a dedicated Image MCP server mounted at `/images/mcp`. The implementation supports OpenAI DALL-E models (`dall-e-3`, `gpt-image-1`), OpenRouter image endpoints, and Google Gemini image generation (`gemini-2.5-flash-image` via Nano Banana). Generated images are saved as session artifacts and automatically watermarked with machine-readable AI metadata in compliance with EU AI Act Article 50(2).

## Required API Keys and Providers

You need at least one of the following API keys configured:

### Option 1: OpenAI API Key
- **Variable**: `OPENAI_API_KEY` (or fallback `OPENAI_API_KEY_BACKUP`)
- **Source**: [OpenAI Platform](https://platform.openai.com/api-keys)
- **Models**: `dall-e-3`, `gpt-image-1`

### Option 2: OpenRouter API Key
- **Variable**: `OPENROUTER_API_KEY`
- **Source**: [OpenRouter](https://openrouter.ai/)
- **Base URL**: `https://openrouter.ai/api/v1`
- **Models**: OpenRouter DALL-E and Gemini image models (e.g. `openrouter/google/gemini-2.5-flash-image`)

### Option 3: Google API Key
- **Variable**: `GOOGLE_API_KEY`
- **Source**: [Google AI Studio](https://aistudio.google.com/)
- **Models**: `gemini-2.5-flash-image` (used directly or as fallback when OpenRouter is unavailable)

## Setup Instructions

### 1. Configure Environment Variables

In your `.env` file or deployment environment:

```bash
# Choose at least one:
OPENAI_API_KEY=sk-...
# OR
OPENROUTER_API_KEY=sk-or-...
# OR
GOOGLE_API_KEY=AIza...

# Optional backup key for OpenAI
OPENAI_API_KEY_BACKUP=sk-...

# Storage service for artifact delivery (s3, supabase, local_folder, in-memory, or none)
ARTIFACT_SERVICE=none
# DISTRIBUTION_DOMAIN=https://cdn.example.com  # Required when ARTIFACT_SERVICE=s3
```

### 2. Available MCP Tools

The Image MCP server registers three tools:

1. **`generate_image_dall_e_3`**:
   - `prompt` (string, required): Text prompt for image generation.
   - `size` (string, enum: `"1024x1024"`, `"1024x1792"`, `"1792x1024"`, default `"1024x1024"`).
   - `quality` (string, enum: `"standard"`, `"hd"`, default `"standard"`).
   - `n` (integer, min 1, max 1, default 1).

2. **`generate_image_gpt_image_1`**:
   - `prompt` (string, required): Text prompt for image generation.
   - `size` (string, enum: `"256x256"`, `"512x512"`, `"1024x1024"`, default `"1024x1024"`).
   - `n` (integer, min 1, max 10, default 1).

3. **`generate_image_nano_banana`**:
   - `prompt` (string, required): Text prompt for image generation.
   - `asset_name` (string, default `"generated_image"`): Asset identifier to track versions.
   - `model_config` (object, default `{}`): Optional model configuration parameters. Uses OpenRouter or Google API with `gemini-2.5-flash-image`.

## MCP Endpoints

- `GET /images/mcp/health`: Unauthenticated health check returning service status, tool count, and detailed validation results from `validate_image_generation_setup()`.
- `GET /images/mcp`: Service information and listing of available tools.
- `POST /images/mcp`: Root JSON-RPC 2.0 protocol endpoint.
- `POST /images/mcp/initialize`: Protocol initialization returning supported protocol version (`2024-11-05`) and server capabilities.
- `GET /images/mcp/sse`: Server-Sent Events stream for real-time protocol communication.
- `POST /images/mcp/tools/list`: Lists tool JSON schemas.
- `POST /images/mcp/tools/call`: Executes a tool by name with arguments.

## Example Usage

### Test Health Check

```bash
curl -sS http://localhost:8000/images/mcp/health
```

### Call Tool via MCP JSON-RPC

```bash
curl -sS -X POST http://localhost:8000/images/mcp/tools/call \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "params": {
      "name": "generate_image_dall_e_3",
      "arguments": {
        "prompt": "A serene mountain lake at sunrise",
        "size": "1024x1024",
        "quality": "standard"
      }
    }
  }'
```

## EU AI Act Compliance

All generated PNG images pass through `mark_png_as_ai_generated()` prior to being saved as session artifacts. The embedded tEXt / iTXt chunk contains machine-readable metadata identifying the content as AI-generated, fulfilling EU AI Act Article 50(2) obligations.

