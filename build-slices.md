# HarnessShop — Stage-1 Build Slices

**Status:** Coarse (Stage 1) · **Version:** v0.1 · **Last updated:** 2026-06-14 · Companion to [PRD.md](PRD.md), [serving-stack.md](serving-stack.md)

> Principle: **thin vertical slices.** Each slice is end-to-end and leaves something that runs,
> so we always have a working system. Risk-ordered — prove the engine and the decoupling seam
> before investing in local serving or the (only) UI build. No UI *design* here; the stage-1
> "UI" is a terminal in a native window.

---

## Slices at a glance

| # | Slice | Done when | Depends | Size |
|---|---|---|---|---|
| 0 | Repo + engine smoke test | OpenCode completes a trivial task from the CLI against *any* model | — | S |
| 1 | Gateway in the path | Same task runs *through* LiteLLM; model-swap by config name works | 0 | S |
| 2 | Local model served (Mac/MLX) | OpenCode completes a task **fully offline** on the Mac 30B via the gateway | 1 | M |
| 3 | First observability trace | A Langfuse trace shows prompt/model/tokens/latency per call | 1 | S–M |
| 4 | Native cockpit (UI + terminal) | A Tauri window with a built-in xterm terminal runs the engine | 1 | L |
| 5 | End-to-end operate (stage-1 done) | From the cockpit: run a real task on the local model, see its trace | 2,3,4 | S |

**Critical path:** 0 → 1 → 2 → 4 → 5, with 3 branching off 1 and folding into 5. Effort
concentrates in slice 4 (the only real build); everything else is install + glue + config.

## Detail

**Slice 0 — Repo skeleton + engine smoke test.** Lay out the repo (`/cockpit`, `/serving`,
`/docs`), install OpenCode, and drive a trivial coding task (create file → edit → run command)
headless from the terminal against any model — even a throwaway hosted key. *Proves the engine
works independent of our infra; de-risks the harness choice on day one.*

**Slice 1 — Gateway in the path.** Stand up LiteLLM with one backend (same model as slice 0);
repoint OpenCode at the gateway endpoint. *Proves the decoupling seam and model-swap-by-config —
the architectural backbone of the whole shop.*

**Slice 2 — Local model served (Mac/MLX).** Install MLX, serve via `mlx_lm.server`, register as
`local-mac` in LiteLLM. Validate the pipe with a small MLX model first, then swap to the 30B MoE
and tune the wired-memory limit. *Proves the local/inspectable tier end-to-end and offline runs.*

**Slice 3 — First observability trace.** Run Langfuse in Docker; enable LiteLLM's OTEL/Langfuse
callback; run a task. *Proves the observability seam — the property that makes this a "shop" and
not just an agent.* (Do after slice 2 so traces are on the real local target.)

**Slice 4 — Native cockpit shell.** Minimal Tauri app: one window embedding an **xterm.js terminal
over a PTY**, in which we launch and drive the OpenCode TUI. Smallest possible native cockpit — the
"chat/agent pane" is just the engine running in the built-in terminal. *Proves the literal stage-1
UI requirement: a clean native window with a built-in terminal.* Any custom chat/agent UI is
deferred to stage 2.

**Slice 5 — End-to-end operate.** Glue: from the Tauri cockpit, run a real coding task on the
local Mac model and confirm the trace lands in Langfuse — all local. *This is stage-1 "done."*

## Deferred (not blocking stage-1)

- `local-rog` (ROG/vLLM) and `hosted-big` as additional gateway entries — config-only, add anytime.
- Custom chat/agent UI pane (beyond the embedded terminal).
- Harness-internals instrumentation (OpenLLMetry spans on tool calls / agent steps).
- Docker sandboxing of the tool/exec boundary.
- Eval harness, multi-agent, model routing/fallbacks.

## Open / verify as we go

- Whether slice 4 embeds the OpenCode TUI cleanly in a PTY, or needs a thin adapter to the engine.
- Exact MLX model build + memory tuning (slice 2).
- Langfuse self-host vs cloud (slice 3).
