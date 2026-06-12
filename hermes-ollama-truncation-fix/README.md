# Hermes Desktop + Ollama: Fixing "Response remained truncated after 3 continuation attempts"

A two-line fix for the truncation loop that appears in Hermes Desktop when terminal tooling is enabled with a local Ollama model.

## The symptom

```
Response truncated (finish_reason='length') - model hit max output tokens
Response truncated (finish_reason='length') - model hit max output tokens
Response truncated (finish_reason='length') - model hit max output tokens
Error: Response remained truncated after 3 continuation attempts
```

What makes it confusing:

- It happens **even on tiny prompts like "hi"** — but only when the `terminal` toolset is enabled
- File-only toolsets (`patch`, `read_file`, `search_files`, `write_file`) work fine
- The Hermes context meter shows ~1% usage, so the session clearly isn't full
- Setting `context_length` and `max_tokens` in the Hermes `config.yaml` does **nothing**
- It looks like a GPU/VRAM problem. It isn't.

## The root cause

Two things collide:

1. **Ollama defaults every model to a 4,096-token context window** (`num_ctx 4096`), regardless of what the model actually supports. This is a deliberate safe default so models load on low-VRAM hardware.

2. **Hermes talks to Ollama through the OpenAI-compatible `/v1` endpoint, which has no parameter for requesting a larger context.** `num_ctx` is not a concept in OpenAI's API spec. So the `context_length` value in your Hermes config is used only for Hermes's own UI bookkeeping — it never reaches Ollama. The two programs silently run with different numbers.

When you enable the `terminal` toolset, Hermes injects the full tool definitions (`terminal`, `process`, `read_terminal`, plus the file tools) into every request. Those definitions alone can exceed 4,096 tokens. The request is truncated before the model has any room to respond, the model instantly hits the output limit, Hermes retries the continuation three times into the same wall, and you get the error. That's why "hi" fails: it's not your prompt that's too big — it's everything around it.

## The fix (two lines)

Bake a larger context into a dedicated model variant using an Ollama Modelfile. This sets the value on Ollama's side of the fence, where the OpenAI-compatible API can't reach.

**1. Create the Modelfile** (see [`Gemma4-Hermes.Modelfile`](Gemma4-Hermes.Modelfile) in this repo):

```
FROM gemma4:12b
PARAMETER num_ctx 64000
PARAMETER num_predict 8192
```

Swap `gemma4:12b` for whatever base model you use.

**2. Build the tuned model:**

```cmd
ollama create gemma4-hermes:12b -f Gemma4-Hermes.Modelfile
```

**3. Verify it took:**

```cmd
ollama run gemma4-hermes:12b "Reply with only the word hello."
ollama ps
```

Check the `CONTEXT` column in `ollama ps` — it should show `64000`. If it still shows `4096`, see [docs/troubleshooting.md](docs/troubleshooting.md) for the environment-variable fallback. Also check the `PROCESSOR` column: `100% GPU` is ideal; a CPU/GPU split means the larger KV cache pushed past your VRAM (it still works, just slower).

**4. Point Hermes at the new model** in `%LOCALAPPDATA%\hermes\config.yaml`:

```yaml
model:
  api_key: ollama
  base_url: http://127.0.0.1:11434/v1
  default: gemma4-hermes:12b
  provider: custom
  context_length: 64000
  max_tokens: 8192
toolsets:
- file
- terminal
platform_toolsets:
  cli:
  - file
```

If `provider: custom` errors in your Hermes build, use `provider: ollama-launch` instead and keep the model name.

**5. Restart Hermes fully and test in escalating order:**

1. `Reply with only the word hello.`
2. `What tools are available to you?`
3. `Use a terminal command to print the current directory only.`
4. A real multi-step task (write a script, run it, show output)

## Verified results

Tested on Windows, Hermes Agent v0.16.0 (2026.6.5), Ollama at `127.0.0.1:11434`, RTX 4080 16 GB, gemma4:12b:

- All four escalating tests passed with no truncation
- A multi-step task (create folder → write Python script → execute → report output) completed across 8+ chained tool calls
- With ~27 toolsets enabled (browser automation, code execution, cron, memory, web search, and more), tool definitions consumed ~9,300 tokens — **more than double the old 4,096 limit by themselves** — and the session ran at 14% context with no errors
- Model stayed at 100% GPU with the 64k context on 16 GB VRAM (8.5 GB loaded)

## What you don't need

- ❌ Ollama environment variables (`OLLAMA_CONTEXT_LENGTH` etc.) — only needed if the Modelfile parameter is ignored; see [docs/troubleshooting.md](docs/troubleshooting.md)
- ❌ Patching Hermes source (`chat_completions.py`) — not needed, and Hermes updates would overwrite it anyway
- ❌ A bigger GPU — this was never a VRAM problem

## Repo contents

| File | Purpose |
|------|---------|
| [`Gemma4-Hermes.Modelfile`](Gemma4-Hermes.Modelfile) | The two-line fix. Edit the `FROM` line for your model. |
| [`config.example.yaml`](config.example.yaml) | Known-good Hermes config for the tuned model. |
| [`docs/troubleshooting.md`](docs/troubleshooting.md) | Full diagnostic walkthrough, fallback paths, command cheat sheet. |

## Credits

Diagnosed and verified June 2026 through hands-on troubleshooting. Shared in the hope it saves someone else the evening it cost.
