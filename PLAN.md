# Gemma 4 31B Abliteration Plan

## Goal
Produce an abliterated (uncensored) version of Gemma 4 31B using p-e-w/heretic,
then import it into Ollama as `gemma4-heretical`.

## Done criteria
- `ollama run gemma4-heretical` works and the model does not refuse harmless prompts
  that the base model would refuse

## Pipeline

- [x] gemma4:31b downloaded in Ollama (pre-quantized GGUF — for comparison only)
- [x] Create flake.nix with Python 3.12 devShell
- [x] Create abliterate script
- [x] Create convert-to-ollama script
- [x] Create PLAN.md
- [x] Enter devShell (`nix develop`), venv + heretic auto-install
- [x] Fix PEFT/Gemma4 compatibility: monkey-patch Gemma4ClippableLinear -> nn.Linear
      (PEFT 0.18.1 doesn't recognize Gemma 4's custom linear wrapper)
      See: https://github.com/huggingface/peft/issues/3129
           https://github.com/p-e-w/heretic/issues/265
- [ ] Run `./abliterate` — heretic downloads google/gemma-4-31B-it from HuggingFace,
      runs ~200 optimization trials, saves abliterated safetensors model
- [ ] Run `./convert-to-ollama` — converts safetensors → GGUF (Q4_K_M), imports to Ollama
- [ ] Verify: `ollama run gemma4-heretical` works correctly

## Notes
- 128 GB unified memory Mac — 31B bf16 (~62 GB) fits with room to spare
- Heretic works on HuggingFace safetensors, NOT Ollama GGUF models
- The Ollama gemma4:31b download is useful for before/after comparison only
- HuggingFace model ID: `google/gemma-4-31B-it`
