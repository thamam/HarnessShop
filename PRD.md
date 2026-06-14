# HarnessShop — Product Brief / PRD

**Status:** Coarse (Stage 1) · **Version:** v0.3 · **Last updated:** 2026-06-14

> A "harness shop": a cockpit to run a Claude-Code-class coding agent on swappable
> open models, built so we can study and instrument it from many angles — tracing,
> evaluation, and white-box inspection. Built gradually. This document is coarse by
> intent: it fixes the critical ingredients and key decisions. No design yet.

---

## 1. Vision & purpose

We want a central harness that behaves like Claude Code / Codex, but with:
- a clean native interface we control,
- easy seams for observability and inspection tooling as we build it,
- an **open** model inside, so we can inspect the model as deeply as the harness.

The end state is a workbench where harnesses, models, and tools are swappable, and any
configuration can be observed and measured. Stage 1 is the smallest version of that
which actually runs.

## 2. Goals & non-goals (Stage 1)

**In scope**
- Native cockpit UI with a built-in real terminal (PTY).
- OpenCode wired as the agent engine.
- One model backend connected and driving the agent.
- Agent runs a coding task end-to-end from the cockpit.
- Basic trace/log capture in place (the observability seam exists, even if thin).

**Out of scope (deferred)**
- Multi-agent / parallel runners.
- Sandbox hardening (Docker isolation comes later).
- Evaluation harness (SWE-bench-style task runner).
- Model training or fine-tuning.
- Polished/elaborate UI.
- The second model tier — wire one tier now, stub the other.

## 3. Locked decisions

| Area | Decision | Why |
|---|---|---|
| **Engine** | **OpenCode**, wrapped in a thin **Tauri** shell + **xterm.js** terminal | Closest to Claude Code in behavior; model-agnostic; large community; hackable enough to instrument. Tauri/xterm keeps Stage 1 small while giving a native cockpit. |
| **Models** | **Two open-weight tiers, no closed/Anthropic models** | White-box inspection is a core project goal, so both tiers must be open. |
| → Local / inspectable | **Primary: Mac (M4 Pro, 48 GB).** Qwen3-Coder-Next 30B MoE @ 4-bit (daily) / Qwen3-Coder-32B dense @ 4-bit (inspection). ROG (8 GB) runs a 7–8B coder only. | Full white-box access (logits/activations/attention). Dense preferred for inspection — no expert routing to disentangle. See Hardware below. |
| → Hosted / near-frontier | DeepSeek-V4 / MiniMax-M3 class on our own cloud endpoint | Drives the harness at ~Claude-Code strength. ("Claude service" = a cloud service, not Anthropic.) |
| **Observability** | OTel GenAI semantic-convention spans via OpenLLMetry → Langfuse | Standard, near-free if instrumented at the gateway; decouples capture from the harness. |

**Alternatives considered (engine):** Pi (sub-1k-token loop, maximally hackable — revisit if
deep loop-inspection later outweighs Claude-Code-likeness); Goose Desktop (ships a native UI,
fastest zero-build start, but heavier internals to modify).

**Note on model versions:** exact SOTA open-weight versions shift monthly; lock the specific
build with a fresh benchmark check at decision time rather than hard-coding a version here.

### Hardware (verified 2026-06-14)

| Machine | Spec | Local model |
|---|---|---|
| **Mac — primary local box** | MacBook Pro · Apple M4 Pro · 48 GB unified · 20-core GPU · Metal 4 | Qwen3-Coder-Next 30B MoE @ 4-bit (daily, fast — ~3B active); Qwen3-Coder-32B dense @ 4-bit (inspection). ~32–36 GB usable for weights + KV. |
| **ROG — constrained companion** | RTX 3070 Ti Laptop · **8 GB VRAM** · Linux (driver 595.71) | 7–8B coder @ 4-bit on-GPU, or 14B @ 4-bit with CPU offload (slow). For small-model CUDA/vLLM experiments, not the 30B tier. |

The Mac is the real local-inference machine. The ROG's 8 GB makes it a small-model/secondary box — useful for CUDA-specific serving (vLLM) and fast iteration, not for the 30B-class local tier.

## 4. Critical ingredients

1. **Harness / engine** — OpenCode: the agent loop, tools, and context management.
2. **Model gateway** — LiteLLM, one OpenAI-compatible endpoint; the harness swaps backends by config.
3. **Inference serving** — local: MLX (Mac) / llama.cpp / vLLM (ROG); hosted: vLLM / SGLang on a cloud box.
4. **Native cockpit** — Tauri shell, chat pane + xterm.js terminal over a PTY.
5. **Observability seam** — OTel → Langfuse; the thing that turns "an agent" into "a studyable agent."
6. **Tool / exec boundary** — file + shell tools behind a logged interface (later: Docker sandbox).
7. **Config / adapter layer** — declarative `config = harness × model × tools` for reproducible experiments.

## 5. Architecture seams (coarse — not design)

Two instrumentation points carry most of the research value:

- **harness ↔ model:** OTel GenAI spans capturing the prompt, response, and token stream.
  Routing all model traffic through the gateway makes this near-free.
- **harness ↔ tools:** terminal, filesystem, and exec events emitted from the cockpit.

Everything downstream (tracing UI, eval, replay, inspection) hangs off these two seams.

## 6. Open inputs & next decisions

- **Hardware** — *resolved (§3):* Mac M4 Pro 48 GB (primary) + ROG RTX 3070 Ti 8 GB (companion). Exact quant/serving config still to validate empirically.
- **Stage-1 build slices** — *outlined:* see [build-slices.md](build-slices.md). 6 risk-ordered slices; critical path 0→1→2→4→5.
- **Serving topology** — *specced:* see [serving-stack.md](serving-stack.md). LiteLLM gateway + MLX/vLLM/hosted backends, traced at the gateway.
- **Engine revisit** — only if deep harness-loop inspection later outweighs Claude-Code-likeness.

## 7. Changelog

- **v0.3 (2026-06-14)** — Added companions: [serving-stack.md](serving-stack.md) (gateway + per-machine serving) and [build-slices.md](build-slices.md) (6-slice stage-1 build path).
- **v0.2 (2026-06-14)** — Pinned hardware (verified by direct check + SSH): Mac M4 Pro 48 GB unified
  → 30B MoE / 32B dense local picks; ROG RTX 3070 Ti 8 GB → 7–8B companion only. Mac set as primary local box.
- **v0.1 (2026-06-14)** — Initial coarse brief. Locked engine (OpenCode + Tauri/xterm) and
  two-tier open-weight model strategy; identified critical ingredients and observability seams.
