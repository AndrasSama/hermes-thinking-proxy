# hermes-thinking-proxy

Patch for [Hermes Agent](https://github.com/NousResearch/hermes-agent) that emits `reasoning_content` in SSE streaming responses, making thinking/chain-of-thought output visible to OpenAI-compatible clients like [Cline](https://cline.bot/), [NextChat](https://github.com/ChatGPTNextWeb/NextChat), and others.

## Problem

When using Hermes Agent's OpenAI-compatible API server (`/v1/chat/completions`) with clients like Cline, the model's thinking/reasoning output is not visible. This is because:

1. The API server does not wire `reasoning_callback` when creating the AIAgent
2. Reasoning content from the model is captured internally but never emitted in the SSE stream
3. The `_fire_stream_delta` path strips `<think>` blocks from text deltas
4. The SSE `chat.completion.chunk` deltas only include `content`, never `reasoning_content`

## Solution

A minimal 26-line patch to `gateway/platforms/api_server.py` that:

1. Wires a `reasoning_callback` in the streaming handler to accumulate reasoning text
2. Passes it through `_run_agent()` → `_create_agent()` → `AIAgent()`
3. Injects the accumulated `reasoning_content` into every `chat.completion.chunk` SSE delta

## Installation

```bash
cd ~/.hermes/hermes-agent
curl -sL https://raw.githubusercontent.com/AndrasSama/hermes-thinking-proxy/main/hermes-agent-api-server-reasoning.patch | git apply
hermes gateway restart
```

## Update safety

Hermes Agent preserves local modifications across updates via `git stash`. With the default config (`updates.non_interactive_local_changes: stash`), running `hermes update` will stash, pull, and auto-restore your changes.

## License

MIT
