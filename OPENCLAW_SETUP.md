# Using gemma4-heretical with OpenClaw

Guide to connecting the MLX 8-bit abliterated Gemma 4 31B model to OpenClaw
on macOS Apple Silicon.

## Prerequisites

- The MLX model directory (`./gemma4-heretical-mlx-8bit/`) already built
  (see `get-gemma4-heretical` for the Ollama GGUF route, or run the MLX
  conversion yourself via `mlx_vlm.convert`)
- This project's nix devShell (`nix develop`)

## 1. Install OpenClaw

### Option A: Nix (recommended)

Add to your flake inputs:

```nix
inputs.nix-openclaw.url = "github:openclaw/nix-openclaw";
```

Then use the Home Manager module. The gateway runs as a launchd service
(`ai.openclaw.gateway`). See https://docs.openclaw.ai/install/nix

After install, set nix mode for the macOS companion app:

```bash
defaults write ai.openclaw.mac openclaw.nixMode -bool true
export OPENCLAW_NIX_MODE=1  # add to your shell profile
```

### Option B: npm (if you have Node 22.16+ via Nix)

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

### Option C: macOS Companion App (.dmg)

Download from https://github.com/openclaw/openclaw/releases (latest: v2026.4.2).
Note: the .dmg is the **companion app** — it connects to the Gateway, which you
still need to install via Option A or B. It is not standalone.

## 2. Start the MLX server

From this project directory:

```bash
nix develop
mlx_vlm.server --model ./gemma4-heretical-mlx-8bit --port 8080
```

The server exposes an OpenAI-compatible API at `http://127.0.0.1:8080/v1`.

Verify it's running:

```bash
curl http://127.0.0.1:8080/v1/models
```

Note the model ID returned — you'll need it for the config below. It will
likely be `gemma4-heretical-mlx-8bit` or similar (whatever the `--model`
path resolves to).

## 3. Configure OpenClaw

Edit `~/.openclaw/openclaw.json` (JSON5 format — comments allowed):

```json5
{
  // "merge" keeps built-in providers alongside your local one
  models: {
    mode: "merge",
    providers: {
      mlx: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "local",           // required field, but mlx_vlm doesn't check it
        api: "openai-completions", // NOT "openai-responses" — mlx_vlm only supports /v1/chat/completions
        models: [
          {
            id: "gemma4-heretical",  // must match what /v1/models returns
            name: "Gemma 4 Heretical (MLX 8-bit)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },

  // Optional: set as default model
  agents: {
    defaults: {
      model: {
        primary: "mlx/gemma4-heretical",
      },
    },
  },
}
```

### Gotchas

- **`apiKey` is required** even for local servers. Any non-empty string works.
- **Model ID must match exactly** what `curl http://127.0.0.1:8080/v1/models`
  returns. If it doesn't match, OpenClaw will connect but get errors.
- **Use `"openai-completions"`, not `"openai-responses"`**. The Responses API
  uses `/v1/responses` which mlx_vlm doesn't implement.
- **`mode: "merge"` matters.** Without it, your config replaces ALL built-in
  providers instead of adding to them.
- **`contextWindow` and `maxTokens`** control prompt truncation. Overestimate
  and the server rejects requests; underestimate and you waste capacity.

## 4. Verify

```bash
# Validate config
openclaw doctor
openclaw doctor --fix    # auto-fix common issues

# Check model appears
openclaw models list

# Test connectivity
openclaw models status

# Send a test message
openclaw agent --message "Hello, are you working?"
```

## 5. About the model

| Property | Value |
|----------|-------|
| Base model | `google/gemma-4-31B-it` |
| Abliteration | trohrbaugh/heretic-ara (Arbitrary-Rank Ablation) |
| Refusals | 5/100 (down from 98/100 on stock model) |
| KL divergence | 0.012 (near-zero quality loss) |
| Quantization | 8-bit (8.643 bits/weight), 32 GB on disk |
| Inference | MLX native Metal on Apple Silicon |

## 6. Alternative: Ollama instead of MLX

If you prefer Ollama (simpler, no server to manage):

```bash
./get-gemma4-heretical          # downloads GGUF Q8_0, registers in Ollama
ollama run gemma4-heretical     # interactive chat
ollama serve                    # or run as API server on localhost:11434
```

OpenClaw config for Ollama would use `baseUrl: "http://127.0.0.1:11434/v1"`
instead, with the same `"openai-completions"` API type.
