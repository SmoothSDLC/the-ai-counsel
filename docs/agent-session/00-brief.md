# Agent-session provider — owner brief (authoritative)

Owner: SmoothSDLC. Branch: `feat/agent-session-provider`. Upstream: jacob-bd/the-ai-counsel @ 614dfb9.
This file is the record of what is being built and why. Design, plan, delta and verification documents in this
directory must trace to it. Nothing is coded until those four documents exist and the owner has answered the
decisions they raise.

## Claims
- The AI Counsel stays the caller. It asks models questions and waits for answers. That loop is not changed.
- A live Claude Code or Codex CLI session can be one of those "models".
- A session cannot be woken from outside. It must be sitting in a wait call when a question arrives; that wait
  returns the instant a question is posted.
- Nothing from the SmoothSDLC/agent-chatroom codebase is reused. Lessons only.

## Features
- New provider type **agent session**. It appears in the model picker like any other model.
- One slot per session. The AI Counsel's own MCP server gains tools to list slots, take one by id, wait for a
  question, answer it, and heartbeat. Streamable HTTP on the AI Counsel's existing address; the address a session
  connects to never changes.
- A slot is *occupied* while its session has a recent heartbeat or is waiting. A question for an empty slot fails
  immediately. A slot id is returned on take so the session can retake it later.
- The preflight "Reply with OK." is sent to the session for real; no auto-reply.
- The session receives only the new question. The full exchange per slot is logged and readable by slot id.
- Slow answers are passed through as they arrive; the AI Counsel waits for the whole answer, so its per-model
  timeout must be long enough (local model at ~10 tokens/s must not be cut off).
- No password, no encryption on this path. LAN, plain HTTP.
- A skill for the session: take a slot, wait, answer, repeat, keep the heartbeat going. No hooks required.

## Objectives
- Use the owner's Claude, GitHub Copilot and Codex **subscriptions** together in one council with no API keys. The
  AI Counsel already signs in with ChatGPT and Copilot subscriptions; sessions bring Claude.
- Let a **local model** sit in the same council (the AI Counsel already supports Ollama and LM Studio).
- **Outcome to code**: a council verdict can be sent to a chosen session as its next task so the decision lands in
  the repository.

## Process
- Pass 1 design document → pass 2 implementation plan → pass 3 deltas against upstream `main` → pass 4
  verification of all three. Each pass is a separate agent; the coordinating session keeps context. Owner
  decisions are collected in each document's last section and answered before code.
