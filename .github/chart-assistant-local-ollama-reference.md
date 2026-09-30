# OpenAlgo chart assistant: local Ollama reference

This file records the troubleshooting notes for the OpenAlgo chart assistant on the `/trading` page, so future answers can rely on the same repo-backed conclusion instead of re-deriving it each time.

Status: diagnosis only; no production code changes were made.

## 1. What the repo says

The chart assistant is not a generic memory-heavy chat layer. It is a tool-driven chart assistant built against the active chart pane and current chart context.

Relevant repo references:
- `docs/userguide/32-charting-terminal/README.md`
- `docs/design/55-agent/README.md`
- `services/agent/providers.py`
- `services/agent/builder.py`

Key findings:
- The chart assistant is tied to the active chart state, not to persistent prior chart history.
- The agent expects a model/provider that supports function/tool calling.
- For Ollama, the provider contract is `ollama` with a base URL like `http://127.0.0.1:11434` and a model name such as `qwen2.5vl:3b`.
- The build contract treats `supports_function_calling` as load-bearing, not decorative.
- A raw JSON payload or a `No tools found` symptom usually means the model/provider is not correctly executing the tool-call contract, not that the model needs fine-tuning.

## 2. Local Ollama setup used here

Working local setup for this environment:
- Provider: `Ollama`
- Base URL: `http://127.0.0.1:11434`
- Model: `qwen2.5vl:3b`
- Local-only, no remote `OLLAMA_HOST` binding

Important: the server must bind to localhost, not a stale LAN IP.

## 3. OLLAMA_HOST issue

The bind error seen earlier was:

```powershell
ollama serve
Error: listen tcp 192.168.1.189:11434: bind: The requested address is not valid in its context.
```

This is caused by stale host configuration, typically an old `OLLAMA_HOST` value or a previous LAN IP assignment that no longer exists on the machine.

Fix:

```powershell
Remove-Item Env:OLLAMA_HOST -ErrorAction SilentlyContinue
ollama serve
```

Use localhost only, not a machine-local LAN IP.

## 4. Recommended local model for this hardware

Laptop spec used in this diagnosis:
- Ryzen 7
- 14 GB RAM
- RTX 3050 4 GB VRAM

Best practical choice:
- `qwen2.5vl:3b`

Why:
- best balance of local runtime cost and image/vision capability
- suitable for chart-style visual context and local Ollama usage
- better fit than large text-only models for chart understanding

Backup option if tool calling remains weak:
- `qwen2.5:7b-instruct-q4_0`

This is a stronger text/tool-calling model, but it is not the chart-first vision choice.

## 5. What we do not do

We do not train a local model for this feature.

The issue is not “teach the model the chart” in the fine-tuning sense. The issue is compatibility with the OpenAlgo tool contract and model capability selection.

A model that emits raw function-call JSON or ignores tool execution is not a training problem. It is a model suitability / tool-call capability problem.

## 6. Model toggles in OpenAlgo

The OpenAlgo model editor includes three capability flags:
- Supports reasoning
- Supports vision
- Unreliable at tool calling

Interpretation for this project:
- `Supports vision` should be enabled for chart/image work.
- `Supports reasoning` is optional depending on model behavior.
- `Unreliable at tool calling` should be enabled if the model fails to execute tools correctly or emits unstructured/raw payloads.

Do not enable every option blindly. The real requirement is whether the model can actually call tools and understand the chart context.

## 7. Exact Windows commands used

```powershell
ollama list
ollama serve
ollama pull qwen2.5vl:3b
ollama run qwen2.5vl:3b "describe this chart in one sentence"
```

If the server is misbound:

```powershell
Remove-Item Env:OLLAMA_HOST -ErrorAction SilentlyContinue
ollama serve
```

## 8. Working conclusion

For this project, the working diagnosis is:
- Local Ollama is valid and supported.
- `qwen2.5vl:3b` is the best starting point for this laptop.
- The chart assistant is tool-driven and expects function-calling capability.
- There is no persistent historical chart memory implemented in the current chart assistant design.
- The main issue is model capability / compatibility, not custom model training.
- A stale `OLLAMA_HOST` is a separate setup issue; it is not the root cause of chart assistant tool failures.

## 9. Future reference

When troubleshooting again, check in this order:
1. Is Ollama running on `127.0.0.1:11434` only?
2. Is the model installed locally?
3. Is the model vision-capable and tool-capable?
4. Does the selected model actually execute tool calls, or does it emit raw JSON?
5. Is the OpenAlgo model config aligned with the actual model capability?

This note is intended to reduce repeat debugging and keep future answers consistent with the repo’s actual design.
