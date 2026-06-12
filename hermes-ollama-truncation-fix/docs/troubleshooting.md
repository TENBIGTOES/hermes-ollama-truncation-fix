# Full troubleshooting guide

The README covers the fix that worked. This document covers the full diagnostic path, the fallbacks if the primary fix doesn't take on your system, and a command cheat sheet.

## Quick decision tree

1. **Need Hermes working immediately?** Switch to file-only tools (`toolsets: [file]`). You can still have Hermes write code to files and run it manually yourself.
2. **Terminal required?** Create the tuned model from the Modelfile (README steps 1–3).
3. **`ollama ps` still shows CONTEXT 4096 after building the tuned model?** Use the environment-variable fallback below.
4. **Still truncating after that?** Use the transport patch fallback below (last resort).
5. **Patch doesn't fix it either?** Collect Hermes logs and open an issue with the report template at the bottom.

## How to confirm you have this specific bug

All of these should be true:

- File-only toolsets work normally
- Enabling `terminal` causes the truncation loop, even on trivial prompts
- The Hermes context meter shows very low usage when it fails
- `ollama ps` shows `CONTEXT 4096` (or another small value) for your model while loaded

If your context meter is actually near full, you have a different problem — a genuinely long session — and the fix is a new session, not this repo.

## Fallback 1: Ollama environment variables

If Ollama ignores the Modelfile's `num_ctx` (seen on some versions), set the server-side default instead. On Windows:

```cmd
setx OLLAMA_CONTEXT_LENGTH 64000
setx OLLAMA_FLASH_ATTENTION 1
setx OLLAMA_KV_CACHE_TYPE q8_0
setx OLLAMA_NUM_PARALLEL 1
setx OLLAMA_KEEP_ALIVE "-1"
```

`setx` only applies to newly started processes: fully quit Ollama from the system tray and relaunch it. A full Windows reboot is the cleanest way to guarantee the variables loaded. Then re-check `ollama ps`.

Notes:

- `OLLAMA_FLASH_ATTENTION` and `OLLAMA_KV_CACHE_TYPE q8_0` reduce the VRAM cost of large contexts and are worth setting even when the Modelfile works, if you're tight on VRAM.
- `OLLAMA_NUM_PARALLEL 1` matters because Ollama divides the context between parallel slots — 2 slots halves your effective context per request.

## Fallback 2: Hermes transport patch (last resort)

Only if the tuned model AND environment variables both fail. The remaining likely cause is Hermes not passing a large enough output limit through the OpenAI-compatible transport.

**Warning:** Hermes updates can overwrite source edits. Back up first.

```cmd
copy %LOCALAPPDATA%\hermes\hermes-agent\agent\transports\chat_completions.py %LOCALAPPDATA%\hermes\hermes-agent\agent\transports\chat_completions.py.bak
notepad %LOCALAPPDATA%\hermes\hermes-agent\agent\transports\chat_completions.py
```

Patch concept — placement matters, so inspect the surrounding `api_kwargs` / `extra_body` logic before pasting:

```python
# Ollama-specific handling for local Ollama backend.
_base_url_str = str(params.get("base_url") or "")
if ":11434" in _base_url_str:
    api_kwargs["max_tokens"] = max(api_kwargs.get("max_tokens", 0), 16384)
    extra_body["num_predict"] = -1
```

This forces a larger `max_tokens` and tells Ollama not to cut generation short (`num_predict: -1` = unlimited).

## Config patterns

### Stable fallback: file-only

```yaml
model:
  api_key: ollama
  base_url: http://127.0.0.1:11434/v1
  default: gemma4:12b
  provider: ollama-launch
  context_length: 65536
  max_tokens: 8192
toolsets:
- file
platform_toolsets:
  cli:
  - file
```

### Heavy pattern to avoid

```yaml
platform_toolsets:
  cli:
  - file
  - kanban
  - memory
  - terminal
```

Loading many toolsets under `platform_toolsets.cli` inflates the tool-definition payload on every request. Enable toolsets globally and keep the CLI platform override lean, or simply toggle toolsets on only when a session needs them.

## VRAM considerations

Context isn't free — the KV cache scales with `num_ctx`. Reference point: gemma4:12b with `num_ctx 64000` loaded at 8.5 GB and stayed `100% GPU` on a 16 GB RTX 4080. If `ollama ps` shows a CPU/GPU split (e.g. `40%/60% CPU/GPU`), the context pushed past your VRAM; the model still works but slows down. Options: lower `num_ctx` (32000 is often plenty), set `OLLAMA_KV_CACHE_TYPE q8_0`, or use a smaller base model.

## Command cheat sheet

| Task | Command |
|------|---------|
| Open Hermes config | `notepad %LOCALAPPDATA%\hermes\config.yaml` |
| Open Hermes folder | `explorer %LOCALAPPDATA%\hermes` |
| Verify Ollama API | `curl http://127.0.0.1:11434/api/tags` |
| Start Ollama manually | `ollama serve` |
| List installed models | `ollama list` |
| Show loaded model + context | `ollama ps` |
| Show a model's baked-in parameters | `ollama show gemma4-hermes:12b` |
| Hermes diagnostics | `hermes doctor` / `hermes status` |
| File-only CLI test | `hermes chat -m gemma4:12b --provider ollama-launch -t file` |
| Terminal test after fix | `hermes chat -m gemma4-hermes:12b --provider custom -t file,terminal` |

## Issue report template

If nothing here fixes it, open an issue with Hermes or Ollama using this template:

```
Hermes Desktop + Ollama local model truncates only when terminal tooling is enabled.

Environment:
- OS:
- Hermes version:
- Ollama version + endpoint:
- GPU:
- Model:

Observed behavior:
- File-only tools work: patch, read_file, search_files, write_file.
- Enabling terminal causes: Error: Response remained truncated after 3 continuation attempts.
- The error appears even when session context is very low.

Already tried:
- Modelfile with num_ctx / num_predict (result of ollama ps: ...)
- OLLAMA_CONTEXT_LENGTH environment variable (result: ...)
- Hermes config context_length / max_tokens (no effect — values don't reach Ollama via /v1)
```
