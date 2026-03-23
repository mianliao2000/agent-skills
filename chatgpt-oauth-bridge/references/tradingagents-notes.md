# TradingAgents Notes

Use this reference when applying the same pattern to a Python CLI or LangChain-style app.

## Architecture that worked

### Login / probe layer
- `scripts/chatgpt_oauth_helper.mjs`
- `scripts/chatgpt_oauth_helper.py`

Responsibilities:
- browser-based ChatGPT OAuth login
- local credential storage
- credential probe
- terminal-friendly wrapper behavior

### Runtime inference layer
- `scripts/chatgpt_oauth_bridge.mjs`
- `tradingagents/llm_clients/codex_oauth_client.py`

Responsibilities:
- read OAuth credentials
- call ChatGPT Codex runtime through `pi-ai`
- normalize plain text and tool calls back into Python/LangChain

### Framework integration layer
- custom provider branch in the client factory
- custom CLI provider option
- dynamic model selection
- reasoning effort selection

## Important implementation lessons

### 1. OAuth success is not enough
A working login page does not mean runtime inference is wired correctly. The runtime must stop using normal OpenAI API-key assumptions.

### 2. Dynamic model discovery matters
Static model lists get stale quickly. Query the remote model source and cache it.

### 3. `client_version` changes the returned model set
Older values can return stale model lists (for example only up to 5.2). Newer values can expose newer models (for example 5.4 / 5.4-mini).

### 4. Strings vs message lists
If the runtime client blindly iterates over `input`, long prompt strings become thousands of one-character messages. Normalize input carefully.

### 5. LangChain needs a Runnable
If the host app uses LangChain composition like `prompt | llm.bind_tools(...)`, the custom chat model should implement `Runnable` semantics.

### 6. Tool calls must round-trip
If the app uses tools/functions, support:
- assistant tool calls going out
- tool result messages coming back
- normalized `tool_calls` in the returned AI message

### 7. Windows encoding issues are real
Use UTF-8 explicitly for:
- logs
- reports
- saved markdown files

### 8. Terminal UX can break after subprocess OAuth
If a full-screen prompt library misbehaves after a helper subprocess returns, verify whether the real issue is:
- helper process not exiting
- terminal redraw state not recovering
- parent process still waiting

## Suggested reusable file set for future projects

- `scripts/chatgpt_oauth_helper.mjs`
- `scripts/chatgpt_oauth_helper.py`
- `scripts/chatgpt_oauth_bridge.mjs`
- a runtime client in the host language
- a local auth file path
- a local model-cache file path

## Good defaults

### Recommended models
- Quick-thinking default: latest mini model
- Deep-thinking default: latest flagship model

### Model menu
- show only newest ~10 models
- pin recommended models to top
- align columns for readability

### Cache
- local JSON cache
- TTL around 30 minutes is a good starting point
