# gemma4-heretical

One-command setup for an **uncensored (abliterated) [Gemma 4 31B](https://deepmind.google/models/gemma/gemma-4/)** model in Ollama, with optional MLX support for native Apple Silicon inference.

Uses the best available community abliteration: [trohrbaugh/gemma-4-31b-it-heretic-ara](https://huggingface.co/trohrbaugh/gemma-4-31b-it-heretic-ara), produced by [Heretic](https://github.com/p-e-w/heretic) using Arbitrary-Rank Ablation (ARA).

| Metric | Stock | Abliterated |
|--------|-------|-------------|
| Refusals (out of 100) | 98 | **5** |
| KL divergence | — | **0.012** (near-zero quality loss) |

## Quick start (Ollama)

Requires [Ollama](https://ollama.com) 0.20.0+ and ~33 GB disk/RAM (Q8_0).

```bash
git clone https://github.com/pmarreck/gemma4-heretical
cd gemma4-heretical
./get-gemma4-heretical
ollama run gemma4-heretical
```

Smaller quantizations are available if RAM is tight:

```bash
./get-gemma4-heretical Q4_K_M    # ~19 GB
./get-gemma4-heretical Q6_K      # ~25 GB
./get-gemma4-heretical IQ4_NL    # ~17 GB (imatrix)
```

## What does `get-gemma4-heretical` actually do?

Community GGUF uploads of Gemma 4 models ship with an **incorrect chat template** (wrong turn delimiters), causing the model to output `---` on repeat instead of actual responses. Ollama has built-in Gemma 4 support via `RENDERER gemma4` / `PARSER gemma4`, but the HuggingFace GGUFs don't set it.

This script:
1. Pulls the GGUF from HuggingFace via `ollama pull`
2. Re-registers it with the correct Gemma 4 renderer/parser metadata
3. That's it — no re-download, no conversion, just a metadata fix

## MLX (Apple Silicon native)

For native Metal inference via [mlx-vlm](https://github.com/Blaizzy/mlx-vlm), convert the model to MLX format:

```bash
nix develop   # provides Python 3.12 + auto-installs dependencies
mlx_vlm.convert \
  --hf-path trohrbaugh/gemma-4-31b-it-heretic-ara \
  --mlx-path ./gemma4-heretical-mlx-8bit \
  -q --q-bits 8
```

Then serve it as an OpenAI-compatible API:

```bash
mlx_vlm.server --model ./gemma4-heretical-mlx-8bit --port 8080
```

See [OPENCLAW_SETUP.md](OPENCLAW_SETUP.md) for connecting this to OpenClaw or any OpenAI-compatible client.

## DIY abliteration

Want to run Heretic yourself instead of using the pre-made model? The `abliterate` script handles it (requires ~62 GB RAM for bf16, several hours on MPS):

```bash
nix develop
./abliterate
# ... wait for 200 optimization trials ...
./convert-to-ollama
```

Note: the `abliterate` script includes a monkey-patch for [PEFT #3129](https://github.com/huggingface/peft/issues/3129) — PEFT doesn't yet recognize Gemma 4's `Gemma4ClippableLinear` module.

## Credits

- [Gemma 4](https://deepmind.google/models/gemma/gemma-4/) by Google DeepMind
- [Heretic](https://github.com/p-e-w/heretic) by Philipp Emanuel Weidmann — the abliteration tool
- [trohrbaugh/gemma-4-31b-it-heretic-ara](https://huggingface.co/trohrbaugh/gemma-4-31b-it-heretic-ara) — the abliterated weights
- [jfiekdjdk/gemma-4-31b-it-heretic-ara-gguf](https://huggingface.co/jfiekdjdk/gemma-4-31b-it-heretic-ara-gguf) — imatrix GGUF quantization

## License

Scripts in this repo are MIT. Model weights are subject to [Google's Gemma license](https://ai.google.dev/gemma/terms).
