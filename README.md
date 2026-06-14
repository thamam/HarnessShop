# HarnessShop

A workbench for running and studying a Claude-Code-class coding agent on open-weight models — with observability and inspection built in from the start.

**Status:** Stage 1 — coarse planning. No implementation yet.

## Documents

- [PRD.md](PRD.md) — product brief: vision, scope, locked decisions, critical ingredients, verified hardware.
- [serving-stack.md](serving-stack.md) — model-serving topology: LiteLLM gateway + MLX / vLLM / hosted backends, traced at the gateway.
- [build-slices.md](build-slices.md) — the 6-slice stage-1 build path.

## Stage-1 summary

- **Engine:** OpenCode, wrapped in a thin Tauri shell with a built-in terminal.
- **Models:** two open-weight tiers — local/inspectable (Qwen3-Coder-class on Mac/ROG) + hosted near-frontier (DeepSeek-V4 / MiniMax-M3 class).
- **Observability:** OpenTelemetry GenAI spans → Langfuse.

See [PRD.md](PRD.md) for detail.
