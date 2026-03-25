---
name: ai-image-creator
description: Generate PNG images using AI (multiple models via OpenRouter including Gemini, FLUX.2, Riverflow, SeedDream, GPT-5 Image, proxied through Cloudflare AI Gateway BYOK). Use when user asks to "generate an image", "create a PNG", "make an icon", or needs AI-generated visual assets for the project. Supports model selection via keywords (gemini, riverflow, flux2, seedream, gpt5), configurable aspect ratios and resolutions.
allowed-tools: Bash, Read, Write
compatibility: Requires uv (Python runner) and network access. Environment variables for CF AI Gateway or direct API keys must be configured in shell profile (~/.zshrc on macOS, ~/.bashrc on Linux, or System Environment Variables on Windows).
metadata:
  tags: image-generation, ai, openrouter, cloudflare, gemini, flux2, riverflow, seedream, gpt5
---

# AI Image Creator

Generate PNG images via multiple AI models, routed through Cloudflare AI Gateway BYOK or directly via OpenRouter/Google AI Studio.

## Model Selection

When the user mentions a model keyword in their image request, use the corresponding `--model` flag:

| Keyword | Model | Use When User Says |
|---------|-------|--------------------|
| `gemini` | [Google Gemini 3.1 Flash](https://openrouter.ai/google/gemini-3.1-flash-image-preview) (default) | "gemini", "generate an image" (no model specified) |
| `riverflow` | [Sourceful Riverflow v2 Pro](https://openrouter.ai/sourceful/riverflow-v2-pro) | "riverflow", "use riverflow" |
| `flux2` | [FLUX.2 Max](https://openrouter.ai/black-forest-labs/flux.2-max) | "flux2", "flux", "use flux" |
| `seedream` | [ByteDance SeedDream 4.5](https://openrouter.ai/bytedance-seed/seedream-4.5) | "seedream", "use seedream" |
| `gpt5` | [OpenAI GPT-5 Image](https://openrouter.ai/openai/gpt-5-image) | "gpt5", "gpt5 image", "use gpt5" |

## Instructions

### Step 1: Write Prompt

For long or complex prompts (recommended), write to `${CLAUDE_SKILL_DIR}/tmp/prompt.txt` using the Write tool:

```
Write prompt text to ${CLAUDE_SKILL_DIR}/tmp/prompt.txt
```

For short prompts (under 200 chars, no special characters), pass inline via `--prompt`.

**CRITICAL — Prompt Quality Tips:**
- Be detailed and descriptive. Include style, colors, composition, background, and intended use.
- Good: "A flat-design globe icon with vertical timezone band lines in blue and teal, white background, clean vector style, suitable for a web app at 512x512 pixels"
- Bad: "globe icon"
- Specify "transparent background" or "white background" explicitly.
- For icons, mention the target size (e.g., "512x512", "favicon at 32x32").
- For photos, describe lighting, camera angle, and mood.

### Step 2: Run Generation Script

```bash
uv run python ${CLAUDE_SKILL_DIR}/scripts/generate-image.py \
  --output "OUTPUT_PATH" \
  [--provider openrouter|google] \
  [--aspect-ratio "16:9"] \
  [--image-size "2K"] \
  [--model "model-id"]
```

With a specific model:
```bash
uv run python ${CLAUDE_SKILL_DIR}/scripts/generate-image.py \
  --output "OUTPUT_PATH" \
  --model riverflow \
  --prompt "A serene mountain lake at sunset"
```

Or with inline prompt (default model):
```bash
uv run python ${CLAUDE_SKILL_DIR}/scripts/generate-image.py \
  --output "OUTPUT_PATH" \
  --prompt "A simple blue circle on white background"
```

### Step 3: Clean Up (if temp file used)

```bash
rm -f ${CLAUDE_SKILL_DIR}/tmp/prompt.txt
```

### Step 4: Verify Output

```bash
file OUTPUT_PATH
```

Confirm it shows "PNG image data" and report the file path and size to the user.

### Step 5: Post-Processing (optional)

If the user needs resizing, format conversion, or other manipulation, first detect available image tools, then use them. See **Image Tools** section below.

## Parameters

| Argument | Required | Default | Description |
|----------|----------|---------|-------------|
| `--output` | Yes | -- | Output file path (parent dirs auto-created) |
| `--prompt` | No | -- | Inline prompt text |
| `--prompt-file` | No | `../tmp/prompt.txt` | Path to prompt file |
| `--provider` | No | `openrouter` | `openrouter` or `google` |
| `--aspect-ratio` | No | model default | OpenRouter only: `1:1`, `16:9`, `9:16`, `3:2`, `2:3`, `4:3`, `3:4`, `4:5`, `5:4`, `21:9` |
| `--image-size` | No | model default | OpenRouter only: `0.5K`, `1K`, `2K`, `4K` |
| `--model` | No | `gemini` | Model keyword (`gemini`, `riverflow`, `flux2`, `seedream`, `gpt5`) or full OpenRouter model ID |
| `--list-models` | No | -- | List available model keywords and exit |

## Environment Variables

| Variable | Required For | Description |
|----------|-------------|-------------|
| `AI_IMG_CREATOR_CF_ACCOUNT_ID` | Gateway mode | Cloudflare account ID |
| `AI_IMG_CREATOR_CF_GATEWAY_ID` | Gateway mode | AI Gateway name |
| `AI_IMG_CREATOR_CF_TOKEN` | Gateway mode | Gateway auth token |
| `AI_IMG_CREATOR_OPENROUTER_KEY` | Direct OpenRouter | OpenRouter API key (`sk-or-...`) |
| `AI_IMG_CREATOR_GEMINI_KEY` | Direct Google | Google AI Studio API key |

Gateway mode activates when all 3 `CF_*` vars are set. Falls back to direct mode if gateway fails.

For first-time setup, see `references/setup-guide.md`.

## Image Tools

On first invocation, detect available image manipulation tools:

```bash
which magick convert sips ffmpeg 2>/dev/null
```

### Available Tools

| Tool | Check | Key Operations |
|------|-------|----------------|
| **ImageMagick 7** (`magick`) | `magick --version` | Resize, crop, convert, composite |
| **ImageMagick 6** (`convert`) | `convert --version` | Same ops, legacy command name |
| **sips** (macOS) | `sips --help` | Resize, format conversion |
| **ffmpeg** | `ffmpeg -version` | Convert formats, resize |

### Common Post-Processing

```bash
# Resize
magick output.png -resize 512x512 icon-512.png

# Multiple sizes (icons)
for s in 16 32 48 64 128 256 512; do magick output.png -resize ${s}x${s} icon-${s}.png; done

# Convert to WebP
magick output.png output.webp

# Maskable icon (add safe-zone padding)
magick output.png -gravity center -extent 120%x120% maskable.png

# macOS sips resize
sips --resampleWidth 512 --resampleHeight 512 output.png --out icon-512.png
```

CRITICAL: Check tool availability before using. Prefer `magick` (IM7) over `convert` (IM6). If no tools found, inform user: `brew install imagemagick`.

## Common Issues

### "No API credentials configured"
**Cause:** Environment variables not set or not exported.
**Fix:** Add exports to `~/.zshrc` and run `source ~/.zshrc`. See `references/setup-guide.md`.

### "HTTP 401: Unauthorized"
**Cause:** Invalid or expired API key/token.
**Fix:** Check `AI_IMG_CREATOR_CF_TOKEN` (gateway) or `AI_IMG_CREATOR_OPENROUTER_KEY` (direct). Regenerate if needed.

### "No images in response"
**Cause:** Model returned text only (safety filter, unclear prompt, or unsupported request).
**Fix:** Make the prompt more specific and descriptive. Avoid prohibited content.

### "Connection error" / timeout
**Cause:** Network issue or image generation taking too long (120s timeout).
**Fix:** Retry. If persistent, try `--provider google` as alternative. Check CF gateway status.

## Detailed API Reference

For full API formats, response schemas, BYOK configuration, and curl examples:
see [references/api-reference.md](references/api-reference.md)

For first-time setup instructions:
see [references/setup-guide.md](references/setup-guide.md)
