---
name: chatgpt-oauth-bridge
description: Add ChatGPT OAuth / OpenAI Codex OAuth support to an existing project, especially Python or CLI apps that do not natively support ChatGPT OAuth login. Use when a project already supports OpenAI-style model selection but needs: (1) browser-based ChatGPT OAuth login, (2) local credential storage and refresh, (3) dynamic discovery of available openai-codex models, (4) a bridge from Python or another runtime into the ChatGPT Codex backend, or (5) UI integration for model/reasoning selection. Useful for adapting agent frameworks, CLIs, LangChain apps, or other codebases to reuse ChatGPT OAuth without depending on OpenClaw at runtime.
---

Build the integration in this order.

## 1. Add a local OAuth helper

Create a small helper that can:
- start browser-based ChatGPT OAuth login
- save credentials locally
- refresh/reuse them later
- run a minimal probe request

Prefer a tiny Node helper when reusing `@mariozechner/pi-ai` or an equivalent Codex/ChatGPT OAuth implementation.

Store credentials in a project-local file such as:
- `.tradingagents/chatgpt-oauth.json`
- or another hidden project-local path

Do not assume OpenAI API keys and do not fake an `OPENAI_API_KEY` when the target runtime is actually ChatGPT OAuth.

## 2. Separate login/probe from runtime inference

Treat these as different layers:
- **login/probe layer**: obtains and validates OAuth credentials
- **runtime layer**: uses those credentials to actually run model requests during the app's real workflow

A common failure mode is getting OAuth login working but still sending real model traffic through standard OpenAI API-key codepaths.

## 3. Use a real runtime bridge for inference

If the host app is Python or another runtime that cannot directly reuse the OAuth client library, add a bridge.

Preferred pattern:
- host app → local bridge → ChatGPT Codex runtime

For the TradingAgents-style integration, this means:
- Python client object for the app/framework
- Node bridge script that reads OAuth credentials
- bridge calls the ChatGPT Codex runtime (`openai-codex-responses` path via `pi-ai`)
- bridge returns a normalized payload back to Python

Do not route ChatGPT OAuth inference through the normal OpenAI `/v1/responses` API unless you have verified the token scopes actually permit that.

## 4. Support tool calls explicitly

If the target app uses tools/functions:
- convert host-side tool definitions to OpenAI-style tool schemas
- forward them through the bridge
- normalize returned tool calls back into the host framework's expected message format

For LangChain-like apps, return an `AIMessage` with:
- `content`
- `tool_calls`
- response metadata

## 5. Handle all input shapes

The runtime client must correctly accept:
- plain strings
- prompt values / prompt templates
- single messages
- message lists
- assistant messages with tool calls
- tool result messages

Do not iterate blindly over `input`; otherwise strings get split into one-character messages and the backend will reject the request.

## 6. Integrate into the CLI/UI flow

When adding a provider like `ChatGPT OAuth`:
- add it as a first-class provider option
- let users log in or refresh credentials in-app
- provide a probe action
- provide a back action to return to provider selection
- after successful login/probe, continue naturally into model selection

Use the app's normal selector UI where possible (for example arrow-key selectors instead of manual number entry) once terminal redraw issues are under control.

## 7. Discover models dynamically from the ChatGPT Codex backend

Do not hardcode only two or three models.

Preferred pattern:
- read OAuth token from the local auth file
- query the ChatGPT Codex models endpoint directly
- cache the result locally with a TTL
- show only the newest/highest-value subset if the full list is noisy

Important lesson: the returned model set can depend on the `client_version` parameter. If the list looks stale, probe newer `client_version` values and cache the one that returns the modern set.

## 8. Make the model menu human-friendly

Format the model picker so it is easy to scan:
- normalize model name casing (`GPT-5.4`, `GPT-5.3 Codex`, etc.)
- align model names, descriptions, and context length columns
- pin recommended models to the top
- mark them clearly with a short ASCII-safe tag such as `* recommended`
- set sensible defaults (for example Quick = mini, Deep = flagship)

## 9. Add reasoning-effort selection

After model selection, expose reasoning effort as its own selector.

Use the same interaction style as the model picker.

Typical options:
- Medium (Default)
- High (More thorough)
- Low (Faster)

Pass that value through the bridge into the actual runtime request.

## 10. Fix Windows terminal and encoding issues early

Expect Windows-specific problems around:
- terminal redraw after subprocess-based OAuth flows
- Unicode encoding when writing logs/reports
- subprocess lookup for CLI executables (`openclaw` vs `openclaw.cmd`)

Mitigations:
- use UTF-8 explicitly for file writes
- prefer ASCII-safe UI labels if the console is not guaranteed UTF-8
- if using subprocess helpers, detect success from output/auth-file side effects instead of trusting fragile interactive terminal state alone

## 11. Test incrementally

Validate in this order:
1. login helper launches browser and stores credentials
2. probe request succeeds
3. model discovery returns the expected modern set
4. runtime bridge returns plain text successfully
5. runtime bridge returns tool calls successfully
6. app/framework runs a real end-to-end task
7. saving logs/reports works under Windows

## References

Read `references/tradingagents-notes.md` when adapting this pattern to another Python CLI or LangChain-style project.
