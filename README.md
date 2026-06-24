# hermes-thinking-proxy

Patch for [Hermes Agent](https://github.com/NousResearch/hermes-agent) that emits `reasoning_content` in SSE streaming responses, making thinking/chain-of-thought output visible to OpenAI-compatible clients like [Cline](https://cline.bot/), [NextChat](https://github.com/ChatGPTNextWeb/NextChat), and others.

## Problem

When using Hermes Agent's OpenAI-compatible API server (`/v1/chat/completions`) with clients like Cline, the model's thinking/reasoning output is not visible. This is because:

1. The API server does not wire `reasoning_callback` when creating the AIAgent
2. Reasoning content from the model is captured internally but never emitted in the SSE stream
3. The `_fire_stream_delta` path strips `<think>` blocks from text deltas
4. The SSE `chat.completion.chunk` deltas only include `content`, never `reasoning_content`

## Solution

A minimal 35-line patch to `gateway/platforms/api_server.py` that:

1. Wires a `reasoning_callback` in the streaming handler to accumulate reasoning text
2. Passes it through `_run_agent()` → `_create_agent()` → `AIAgent()`
3. Emits the accumulated reasoning as a `<thinking>` block in the SSE stream before the first content delta

## Installation

```bash
cd ~/.hermes/hermes-agent
curl -sL https://raw.githubusercontent.com/AndrasSama/hermes-thinking-proxy/main/hermes-agent-api-server-reasoning.patch | git apply
hermes gateway restart
```

## Update safety

Hermes Agent preserves local modifications across updates via `git stash`. With the default config (`updates.non_interactive_local_changes: stash`), running `hermes update` will stash, pull, and auto-restore your changes.

## How it works

The patch adds 4 small changes to `gateway/platforms/api_server.py`:

1. **`_create_agent()`**: Accepts and forwards `reasoning_callback` to `AIAgent()`
2. **Streaming handler**: Defines `_on_reasoning()` callback that accumulates reasoning text
3. **`_run_agent()`**: Accepts and forwards `reasoning_callback` through to `_create_agent()`
4. **`_write_sse_chat_completion()` → `_emit()`**: Emits `<thinking>` block before first content delta

## Compatibility

- **Tested with:** Hermes Agent v0.17.0
- **Models:** Any reasoning-capable model (e.g., `openrouter/owl-alpha`, `anthropic/claude-*`, `deepseek/*`)
- **Clients:** Cline (VS Code), NextChat, any OpenAI-compatible client that parses `<thinking>` tags

## License

MIT