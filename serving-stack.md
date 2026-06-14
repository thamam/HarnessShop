# HarnessShop — Serving & Gateway Stack

**Status:** Coarse (Stage 1) · **Version:** v0.1 · **Last updated:** 2026-06-14 · Companion to [PRD.md](PRD.md)

> Goal of this layer: expose **one stable OpenAI-compatible endpoint** that the OpenCode
> harness talks to, with three swappable backends behind it (Mac-local, ROG-local, hosted),
> all traced uniformly. Model swap = a config change, never a harness change.

---

## 1. Topology

```
                         ┌───────────────────────────┐
   OpenCode (harness) ──▶│  LiteLLM gateway  :4000   │  one OpenAI-compatible endpoint
                         │  model_list + routing     │
                         └────────────┬──────────────┘
                                      │  (OTEL callback)
            ┌─────────────────────────┼───────────────────────────┐
            ▼                         ▼                            ▼
   ┌──────────────────┐    ┌──────────────────┐        ┌──────────────────────┐
   │ Mac · MLX :8080  │    │ ROG · vLLM :8000 │        │ Hosted near-frontier │
   │ Qwen3-Coder 30B  │    │ 7–8B coder (AWQ) │        │ DeepSeek-V4 /        │
   │ MoE / 32B dense  │    │ CUDA, 8 GB       │        │ MiniMax-M3 class     │
   │ PRIMARY local    │    │ companion        │        │ provider → own-host  │
   └──────────────────┘    └──────────────────┘        └──────────────────────┘
                                      │
                                      ▼
                         ┌───────────────────────────┐
                         │  Langfuse (OTLP sink)     │  traces: prompt/model/tokens/latency/cost
                         └───────────────────────────┘
```

**Why a gateway in the middle (the key call):** it decouples the harness from backends and gives
one place for model-swap, routing, fallback, auth, and — most importantly — **uniform tracing**.
One instrumentation point instead of per-backend. Cost: one extra process + negligible latency.
Verdict: gateway from day one.

## 2. Backends

### 2a. Mac — primary local, inspectable (MLX)

- **Server:** `mlx_lm.server` (ships OpenAI-compatible `/v1/chat/completions`). Best Apple-Silicon
  perf + scriptable. Alts: **LM Studio** (GUI, also MLX) as a convenient fallback; **Ollama**
  (Metal/llama.cpp, simplest but a touch slower on these models). Pick: `mlx_lm.server` daily.
- **Models:** Qwen3-Coder-Next 30B MoE @ 4-bit (daily); Qwen3-Coder-32B dense @ 4-bit (inspection).
- **Memory:** raise the GPU wired limit so MLX can use more of the 48 GB —
  `sudo sysctl iogpu.wired_limit_mb=40960` (~40 GB). Budget ~32–36 GB for weights + KV.
- **Launch:**
  ```bash
  python -m mlx_lm.server --model <mlx-community/Qwen3-Coder-...-4bit> --host 0.0.0.0 --port 8080
  ```
- **Perf:** MoE (~3B active) snappy; dense 32B ~10–15 tok/s — fine for inspection sessions.

### 2b. ROG — small CUDA companion (vLLM)

- **Server:** vLLM, OpenAI-compatible on :8000. 8 GB VRAM → a **7–8B** coder.
- **Quantization:** the RTX 3070 Ti is **Ampere (sm_86) → no FP8.** Use **4-bit AWQ/GPTQ**, not fp8.
- **Launch:**
  ```bash
  vllm serve <Qwen3-Coder-7B-AWQ> --quantization awq \
    --max-model-len 8192 --gpu-memory-utilization 0.90 --port 8000
  ```
  KV cache is the squeeze at 8 GB → keep `max-model-len` modest. For a 14B, fall back to
  llama.cpp with CPU offload (slower).
- **Reach from the Mac cockpit:** SSH local-forward — `ssh -L 8001:localhost:8000 rog` — encrypted,
  no firewall fuss. Gateway then targets `http://localhost:8001`.
- **Role:** fast small-model iteration and CUDA-specific experiments (vLLM features, speculative
  decode). Not the 30B tier.

### 2c. Hosted — near-frontier open (provider → own-host)

- **Option A (stage 1):** a hosted inference **provider** serving the open model via
  OpenAI-compatible API — zero ops, just a key. Gets us running.
- **Option B (later):** our own **vLLM/SGLang** on a rented GPU box — needed for true white-box on
  the *big* model (logits/internals), at real GPU cost.
- **Trade-off:** white-box at *both* tiers is a stated goal, so the eventual answer is own-host for
  the big model. Pick: **provider now, own-host when big-model inspection is needed** — a gateway
  config swap.

## 3. Gateway — LiteLLM

One endpoint, friendly model names mapped to backends. OpenCode swaps models by changing a name.

```yaml
# config.yaml
model_list:
  - model_name: local-mac          # primary local
    litellm_params:
      model: openai/qwen3-coder-30b
      api_base: http://localhost:8080/v1
      api_key: "none"
  - model_name: local-rog          # CUDA companion (SSH-forwarded)
    litellm_params:
      model: openai/qwen3-coder-7b
      api_base: http://localhost:8001/v1
      api_key: "none"
  - model_name: hosted-big         # near-frontier open
    litellm_params:
      model: <provider>/<deepseek-v4-or-minimax-m3>
      api_key: os.environ/HOSTED_API_KEY

litellm_settings:
  callbacks: ["otel"]              # uniform tracing for every backend
  # success_callback: ["langfuse"] # alternative: direct Langfuse integration

# launch:  litellm --config config.yaml --port 4000
```

OpenCode then points at `http://localhost:4000/v1` and selects `local-mac` / `local-rog` /
`hosted-big`. Optional: routing + fallbacks (e.g., `hosted-big` falls back to `local-mac` offline).

## 4. Observability hook

- Instrument **at the gateway**: LiteLLM's OTEL callback (or its native Langfuse integration)
  exports every call — prompt, model, tokens, latency, cost — to **Langfuse**.
- **Langfuse:** self-host via Docker for stage 1 (data stays local; good for a research box);
  point `OTEL_EXPORTER_OTLP_ENDPOINT` at it.
- Later, when we instrument the *harness internals* (tool calls, agent steps), add **OpenLLMetry**
  spans alongside — same OTLP sink. This is the seam that makes the shop "studyable."

## 5. Minimal first stand-up (within this layer)

Don't build all three at once. Stage-1 minimum:

1. **Mac MLX** serving one model on :8080.
2. **LiteLLM** with just `local-mac`, on :4000.
3. **Langfuse** (Docker) receiving traces.
4. Point OpenCode at the gateway; run one task; confirm a trace appears.

Add `local-rog` and `hosted-big` as config entries once the loop works. (Maps to build slices 1–3.)

## 6. Verify at install

- Exact MLX quant build names (community repos shift).
- vLLM AWQ 7–8B model choice that fits 8 GB with usable context.
- Langfuse self-host vs cloud.
- Provider choice for the hosted tier (and when to cut over to own-host).
