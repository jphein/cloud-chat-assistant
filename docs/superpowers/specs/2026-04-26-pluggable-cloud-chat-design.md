# Pluggable cloud-chat — design

**Date:** 2026-04-26
**Status:** Draft, awaiting JP review
**Author:** Claude (with JP)

## 1. Goal

Turn `cloud-chat-assistant` from a single-file MCP server into a multi-language drop-in library, mirroring the integration style of `realm-sigil`, `clock.realm.watch`, `dark.realm.watch`, and `lexicon.realm.watch`. Other JP projects in Go, Python, and JavaScript should be able to add chat-API access in one import without each reinventing model maps, SigV4 signing, or SSE parsing.

## 2. Why now

A grep across `~/Projects/` finds at least **eight active consumers** of cloud chat APIs, each using a different abstraction:

| Project | Language | Today |
|---|---|---|
| gnome-speaks | Python | already imports `llm_stream` from this repo |
| realmwatch (chat_bridge, oracle_daemon) | Python | direct `openai.AsyncAzureOpenAI` + shared session DB |
| the-oracle, oracle-mcp | Python | hand-rolled Anthropic clients |
| memorypalace | Python | own urllib-based Ollama / OpenAI-compat / Anthropic abstraction |
| rlm | Python | three separate per-provider files (anthropic, openai, azure_openai) |
| oracle (Sanctum) | Go | hand-rolled `providers/{anthropic,openai,bedrock}.go` with own model map |
| bestiary | TypeScript (Bun) | `@aws-sdk/client-bedrock-runtime` direct |
| portfolio | Go | references model names |

Two model-alias maps already duplicate each other across languages: `llm_stream.MODEL_MAP` (Python) and `oracle/providers/models.go` (Go). Both contain entries like `claude-opus-4-6 -> us.anthropic.claude-opus-4-6-v1`. Every cross-language drift is a future bug.

## 3. Pattern reference

The realm.watch family converges on a clean shape:

- **One canonical data file**, language-agnostic. `realm-sigil` keeps it in `words/realms.json`.
- **Per-language thin runtimes**. Each language reads the canonical data and ships small idiomatic bindings.
- **A sync script** that regenerates language-specific stubs from canonical data.
- **A tiered integration story**. CSS-only / JS / custom-element for `dark`; ldflags / one-line handler / static-site builder for `sigil`.

`cloud-chat`'s twist: the canonical data layer holds *both* a model registry **and** a set of declarative provider recipes (URL templates, auth scheme tags, body shapes, response extraction paths). The per-language runtime becomes a small "render the recipe, run the auth scheme, parse the stream" engine — roughly 500 lines per language.

## 4. Non-goals (v0.x)

The following stay out of the library proper. They live in the existing `mcp_cloud_chat.py` MCP server (the canonical Python demo consumer) or in higher-level packages.

- **Sessions / persistent history** (SQLite-backed)
- **Multi-chat fan-out** (parallel calls across N models)
- **MCP wire protocol** (the existing server uses the lib; the lib doesn't know about MCP)
- **CLI-based model scanning** (`az`, `aws`, `gcloud`)
- **Tool / function calling**
- **Image input or output**
- **Embeddings**
- **Audio**

These can be layered on later as their own per-language packages once the core is stable.

## 5. Architecture

```
                                 +--------------------------+
                                 |   models/models.json     |  canonical
                                 |   providers/providers.json|  data
                                 +--------------------------+
                                            |
                  +-------------------------+--------------------------+
                  |                         |                          |
              python/cloud_chat/        go/cloud_chat/             js/cloud-chat/
                  |                         |                          |
                  v                         v                          v
              consumers:                consumers:                consumers:
              gnome-speaks              oracle                    bestiary
              realmwatch                portfolio                 artcardsv5
              the-oracle, etc.                                    (Vercel/Bun/Node)
                  |
                  v
              mcp_cloud_chat.py  (stays in repo as canonical Python demo consumer)
              + sessions.db, MCP wire protocol, multi_chat fan-out, CLI scan
```

Three languages, identical canonical inputs, identical observable behavior. Adding a fourth language (Rust, Bash) costs ~1 file: a recipe runtime.

## 6. Repo layout

The existing `cloud-chat-assistant/` repo absorbs the multi-language structure. The MCP server moves out of root into a subdirectory once the Python package is in place, but the path is preserved via a thin shim during migration.

```
cloud-chat-assistant/
  models/
    models.json              canonical model registry
  providers/
    providers.json           canonical provider recipes
  python/
    cloud_chat/
      __init__.py            re-exports stream_chat, astream_chat, MODELS, PROVIDERS
      registry.py            loads models.json + providers.json
      runtime.py             generic recipe interpreter
      auth.py                auth schemes: bearer, x-api-key, sigv4, oauth, query
      streams.py             SSE parser + Bedrock event-stream parser
    pyproject.toml           publishable to PyPI
    tests/
  go/
    cloud_chat/
      sigil.go               package entry
      registry.go            generated from canonical JSON via sync.sh
      runtime.go             recipe interpreter
      auth.go                auth schemes
      streams.go             stream parsers
    cloud_chat_test.go
    go.mod
  js/
    cloud-chat/
      index.js               package entry
      registry.js            generated from canonical JSON via sync.sh
      runtime.js
      auth.js
      streams.js
    package.json
    tests/
  sync.sh                    regenerates registry.{go,js} from canonical JSON
  mcp/                       (existing MCP server, refactored to import python/cloud_chat)
    mcp_cloud_chat.py
    sessions.py
    multi_chat.py
    scan.py
  docs/
    superpowers/specs/2026-04-26-pluggable-cloud-chat-design.md  (this file)
  README.md                  realm.watch-style multi-tier integration guide
```

`models.json` and `providers.json` are the only two files where adding a new model or provider lives. Everything else is regenerated.

## 7. Canonical schema: `models.json`

A single registry of every supported chat model, the providers that serve it, and capability flags. Example:

```json
{
  "schema_version": 1,
  "models": {
    "claude-sonnet-4.6": {
      "family": "anthropic",
      "providers": {
        "anthropic":    "claude-sonnet-4-6",
        "bedrock":      "us.anthropic.claude-sonnet-4-6",
        "digitalocean": "anthropic-claude-4.6-sonnet",
        "puter":        "claude-sonnet-4-6"
      },
      "capabilities": ["streaming", "tool-use", "vision"],
      "context_window": 200000
    },
    "gpt-5.3": {
      "family": "openai",
      "providers": {
        "openai":       "gpt-5.3-chat",
        "azure":        "gpt-5.3-chat",
        "azure_codex":  "gpt-5.3-chat",
        "digitalocean": "openai-gpt-5.3-codex",
        "puter":        "gpt-5.2-chat-latest"
      },
      "capabilities": ["streaming", "reasoning"]
    }
  }
}
```

Resolution rule: given canonical name + provider, return the provider-specific ID. If the model isn't listed, fall back to canonical name (matches today's `llm_stream.resolve_model` behavior at `llm_stream.py:95-103`).

## 8. Canonical schema: `providers.json` (recipes)

Each provider is described declaratively. The runtime renders the recipe, runs the named auth scheme, posts the body, and routes the response stream through the named parser.

```json
{
  "schema_version": 1,
  "providers": {
    "anthropic": {
      "url_template": "https://api.anthropic.com/v1/messages",
      "method": "POST",
      "auth": { "scheme": "x-api-key", "key_field": "api_key" },
      "headers": { "anthropic-version": "2023-06-01", "content-type": "application/json" },
      "body_shape": "anthropic_messages",
      "stream_format": "sse_anthropic"
    },
    "openai": {
      "url_template": "https://api.openai.com/v1/chat/completions",
      "method": "POST",
      "auth": { "scheme": "bearer", "key_field": "api_key" },
      "body_shape": "openai_chat",
      "stream_format": "sse_openai"
    },
    "azure": {
      "url_template": "{endpoint}/openai/deployments/{model}/chat/completions?api-version=2024-12-01-preview",
      "method": "POST",
      "auth": { "scheme": "api-key-header", "key_field": "api_key" },
      "body_shape": "openai_chat_no_model",
      "stream_format": "sse_openai"
    },
    "azure_serverless": {
      "url_template": "{endpoint}/models/chat/completions?api-version={api_version}",
      "method": "POST",
      "auth": { "scheme": "api-key-header", "key_field": "api_key" },
      "body_shape": "openai_chat",
      "stream_format": "sse_openai",
      "defaults": { "api_version": "2024-05-01-preview" },
      "overrides_by_model_prefix": [
        { "prefix": ["o1", "o4"], "set": { "api_version": "2024-12-01-preview" } }
      ]
    },
    "azure_codex": {
      "url_template": "{endpoint}/openai/v1/responses",
      "method": "POST",
      "auth": { "scheme": "api-key-header", "key_field": "api_key" },
      "body_shape": "openai_responses",
      "stream_format": "sse_openai_responses"
    },
    "bedrock": {
      "url_template": "https://bedrock-runtime.{region}.amazonaws.com/model/{model_url_encoded}/converse-stream",
      "method": "POST",
      "auth": { "scheme": "sigv4", "service": "bedrock", "region_field": "aws_region",
                "access_key_field": "aws_access_key", "secret_key_field": "aws_secret_key" },
      "body_shape": "bedrock_converse",
      "stream_format": "bedrock_event_stream"
    },
    "google": {
      "url_template": "https://generativelanguage.googleapis.com/v1beta/models/{model}:streamGenerateContent?alt=sse&key={api_key}",
      "method": "POST",
      "auth": { "scheme": "query-key", "key_field": "google_api_key" },
      "body_shape": "google_generate_content",
      "stream_format": "sse_google"
    },
    "digitalocean": {
      "url_template": "https://inference.do-ai.run/v1/chat/completions",
      "method": "POST",
      "auth": { "scheme": "bearer", "key_field": "do_api_key" },
      "body_shape": "openai_chat",
      "stream_format": "sse_openai",
      "body_overrides": { "max_completion_tokens_min": 256 }
    },
    "puter": {
      "url_template": "https://api.puter.com/puterai/openai/v1/chat/completions",
      "method": "POST",
      "auth": { "scheme": "bearer", "key_field": "puter_api_key" },
      "body_shape": "openai_chat",
      "stream_format": "sse_openai"
    },
    "ollama": {
      "url_template": "{ollama_endpoint}/v1/chat/completions",
      "method": "POST",
      "auth": { "scheme": "none" },
      "body_shape": "openai_chat",
      "stream_format": "sse_openai",
      "defaults": { "ollama_endpoint": "http://localhost:11434" }
    }
  }
}
```

Every field is data. Adding a new OpenAI-compatible provider (Groq, Together, Fireworks, OpenRouter, vLLM) is one JSON entry — zero code in any language.

### 8.1 Body-shape catalog

A small fixed set of named body shapes is implemented once per language. Each shape takes `(model, messages, system_prompt, max_tokens, temperature, extras)` and returns a JSON-serializable dict.

- `openai_chat` — standard `{ model, messages, max_tokens, stream }`
- `openai_chat_no_model` — Azure deployed: model lives in the URL path, not the body
- `openai_responses` — Azure Responses API: `{ model, input, reasoning, max_output_tokens }`
- `anthropic_messages` — `{ model, max_tokens, system, messages, stream }`
- `bedrock_converse` — `{ modelId, messages: [{role, content:[{text}]}], inferenceConfig, system:[{text}] }`
- `google_generate_content` — `{ contents: [{role:"user|model", parts:[{text}]}], system_instruction }`

Six shapes cover all current providers and at least the next decade of OpenAI clones.

### 8.2 Auth scheme catalog

- `none` — no auth (Ollama, local servers)
- `bearer` — `Authorization: Bearer <key>` (OpenAI, DigitalOcean, Puter)
- `x-api-key` — `x-api-key: <key>` (Anthropic)
- `api-key-header` — `api-key: <key>` (Azure)
- `query-key` — append `?key=<key>` to URL (Google)
- `sigv4` — AWS Signature V4 over the request (Bedrock)
- `oauth-bearer` — bearer with token from a token endpoint (future: Vertex AI native, not Google AI Studio)

### 8.3 Stream-format catalog

- `sse_openai` — `data: {choices:[{delta:{content}}]}` lines
- `sse_anthropic` — `data: {type:"content_block_delta", delta:{text}}`
- `sse_google` — `data: {candidates:[{content:{parts:[{text}]}}]}`
- `sse_openai_responses` — `data: {type:"response.output_text.delta", delta:"..."}`
- `bedrock_event_stream` — AWS binary frames (already implemented in `llm_stream.py:148-189`, port to Go and JS)

## 9. Per-language API surface

The lib presents the same logical operation in idiomatic shape per language. Each shape is the one consumers already use today.

### 9.1 Python (matches existing `llm_stream`)

```python
from cloud_chat import stream_chat, astream_chat, MODELS, PROVIDERS, resolve_model

# Sync generator (gnome-speaks shape):
for token in stream_chat("anthropic", "claude-sonnet-4.6", messages, system_prompt="...", config=cfg):
    print(token, end="")

# Async generator (mcp_cloud_chat / realmwatch shape):
async for token in astream_chat("bedrock", "claude-opus-4.6", messages, config=cfg, client=httpx_client):
    ...
```

`config` defaults to merged `~/.config/cloud-chat-assistant/config.json` + `~/.config/speech-to-cli/config.json` (today's behavior in `llm_stream.py:76-86`). Migration: today's `from llm_stream import …` becomes `from cloud_chat import …` — identical signatures.

### 9.2 Go (matches existing `oracle/providers/`)

```go
import cloudchat "github.com/jphein/cloud-chat-assistant/go/cloud_chat"

cfg := cloudchat.Config{
    APIKey: os.Getenv("ANTHROPIC_API_KEY"),
}
err := cloudchat.StreamChat(ctx, "anthropic", "claude-sonnet-4.6", messages, cfg, func(token string) {
    fmt.Print(token)
})
```

Callback-based, matching what `oracle/providers/anthropic.go:23` already exposes. Migration path for `oracle`: delete `providers/`, swap to `cloudchat.StreamChat(...)`. Drops ~600 lines of hand-rolled provider code.

### 9.3 JavaScript / TypeScript (modern async-iterable)

```js
import { streamChat } from "@jphein/cloud-chat";

for await (const token of streamChat({
    provider: "bedrock",
    model: "claude-opus-4.6",
    messages,
    config: { aws_access_key: "...", aws_secret_key: "...", aws_region: "us-west-2" }
})) {
    process.stdout.write(token);
}
```

Async iterable matches modern Node / Bun / browser idioms. Migration path for `bestiary`: drop `@aws-sdk/client-bedrock-runtime`, use `streamChat({ provider: "bedrock", ... })`.

## 10. Data flow (worked example)

A `stream_chat("bedrock", "claude-sonnet-4.6", msgs, cfg)` call:

1. **Resolve model.** `MODELS["claude-sonnet-4.6"].providers["bedrock"] -> "us.anthropic.claude-sonnet-4-6"`.
2. **Look up recipe.** `PROVIDERS["bedrock"]`.
3. **Render URL.** Substitute `{region}` from `cfg.aws_region`, `{model_url_encoded}` from the resolved ID. Result: `https://bedrock-runtime.us-east-1.amazonaws.com/model/us.anthropic.claude-sonnet-4-6/converse-stream`.
4. **Build body.** Apply `bedrock_converse` shape: convert messages, attach `inferenceConfig`, attach `system` from `system_prompt`.
5. **Sign request.** Auth scheme is `sigv4`; runtime calls `sigv4_sign(method, url, payload, access_key, secret_key, region, service)`.
6. **POST and stream.** Response goes through the `bedrock_event_stream` parser, which yields tokens.

Same flow for every provider — only the recipe lookups differ.

## 11. Migration phases

Six phases. Each is independently mergeable; the tree is functional after every phase.

### Phase 0 — Extract canonical data, no behavior change

- Move `MODEL_MAP` from `llm_stream.py:25-55` to `models/models.json`.
- `llm_stream.py` reads the JSON at import time. Output is identical.
- Add a small `tests/test_resolve_parity.py` that asserts old hard-coded map == JSON-loaded map for every entry (catches silent drift on future edits).

### Phase 1 — Lift recipes into `providers.json`

- Refactor `_build_request` (`llm_stream.py:249-308`) into a generic interpreter that consumes `providers/providers.json`.
- Each provider's if-branch becomes a JSON entry.
- Bedrock and Codex get their own entries (they already did, just inline).
- Behavior unchanged. The `mcp_cloud_chat.py:701-770` Codex routing collapses into a recipe lookup.
- Tests: golden-file fixtures of `(provider, model, messages, cfg) -> (url, headers, body)` for every provider, before and after.

### Phase 2 — Promote to a real Python package

- Create `python/cloud_chat/` with `pyproject.toml`. Move `llm_stream.py` content into `cloud_chat/runtime.py`, split out `auth.py`, `streams.py`, `registry.py`.
- Re-export at `cloud_chat/__init__.py`: `stream_chat`, `astream_chat`, `MODELS`, `PROVIDERS`, `resolve_model`, `LLMStreamError`.
- Leave a thin shim `llm_stream.py` at repo root that does `from cloud_chat import *` so `gnome-speaks` (which imports `from llm_stream import ...`) keeps working until it's migrated.
- Update `mcp_cloud_chat.py` import line.
- Move MCP server to `mcp/mcp_cloud_chat.py`. Update entrypoint in `setup.sh`.

### Phase 3 — Add Go package

- Write `go/cloud_chat/` with the same logical surface plus a callback-based `StreamChat`.
- `registry.go` is generated by `sync.sh` from `models/models.json` + `providers/providers.json` (so JSON edits flow to Go automatically).
- Port the SigV4 signer (`llm_stream.py:117-143`) and the Bedrock event-stream parser (`llm_stream.py:148-189`) — well-defined, easy to mirror.
- Port `oracle/providers/anthropic.go`-style SSE parsers from existing oracle code.
- Tests: same golden-file fixtures from Phase 1, run against Go runtime — identical (url, headers, body) output for every (provider, model) combination.
- First consumer migration: `oracle` swaps `providers/` for `cloud-chat-go` import. Verifies parity with a real consumer.

### Phase 4 — Add JS package

- Write `js/cloud-chat/` mirroring the Python and Go shapes. Async iterable API.
- Implement SigV4 in pure JS (no `@aws-sdk` dep — that's a 50KB+ tree). Reference: `llm_stream.py:117-143` is ~30 lines.
- Bedrock event-stream parser likewise.
- Publish to npm as `@jphein/cloud-chat`.
- First consumer migration: `bestiary` swaps `@aws-sdk/client-bedrock-runtime` for `@jphein/cloud-chat`. Drops a ~50KB dep.

### Phase 5 — `sync.sh`, CI, and README

- `sync.sh`: validates `models.json` and `providers.json` against JSON Schema, regenerates `go/cloud_chat/registry.go` and `js/cloud-chat/registry.js` from canonical data. Mirrors `realm-sigil/sync-words.sh:1`.
- CI: per-language test suites all consume the *same* fixture file (a list of `(provider, model, messages) -> expected_request_shape` tuples). Cross-language parity is enforced by a single source of truth.
- `README.md` rewritten in realm.watch tier style: tier 1 (Python), tier 2 (Go), tier 3 (JS). Each tier shows the smallest possible drop-in.

## 12. Testing strategy

The single most valuable test surface is **cross-language parity**, modeled on `realm-sigil`'s "same hash + realm produces the same name in Go, Python, JS." Concretely:

- **One fixture file** per (provider, model). Inputs: messages, system prompt, config. Expected outputs: the *unsigned* HTTP request shape — `(method, url, headers_minus_auth, body_json)`.
- **Three test runners**, one per language. Each loads the fixtures, calls the runtime in record-only mode (build the request but don't send), and asserts equality.
- **One signing-roundtrip test per language** for SigV4 — independent of providers; just verifies the AWS canonical signing algorithm.
- **Real-call integration tests** are gated behind real credentials and run on demand, not in CI. Per-provider, one model each.

Drift detection is automatic: if Python and Go produce different headers for the same model, CI breaks.

## 13. Versioning

The canonical JSON files have a `schema_version` field. The three language packages share a major version aligned with `schema_version`. Minor versions track bug fixes within a schema version.

- `cloud_chat==0.1.0` (Python), `v0.1.0` (Go module), `0.1.0` (npm) all consume `schema_version: 1`.
- Adding a new provider entry: minor bump, no schema bump.
- Renaming a recipe field: schema bump.
- The library exposes a `SCHEMA_VERSION` constant so consumers can assert what they're talking to.

## 14. Open questions

These are deliberately not closed in this spec; flag during implementation.

1. **Repo rename.** Should the repo move from `cloud-chat-assistant` to `cloud-chat`? Spec is layout-neutral. Suggest deferring until after Phase 2 lands and the MCP server is no longer "the project."
2. **Vertex AI native auth.** Today the Google entry uses Google AI Studio (API-key-in-query). Vertex AI proper uses `gcloud auth print-access-token` OAuth. Add as a separate `vertex` provider entry once an actual consumer asks for it.
3. **Per-package distribution names.** Python: `cloud-chat` on PyPI. Go: GitHub module path under `github.com/jphein/cloud-chat-assistant/go`. JS: `@jphein/cloud-chat` on npm. Open: should we mirror sigil and use `realm-cloud-chat` everywhere?
4. **Streaming-only?** Today `llm_stream` is streaming-only. Some consumers (memorypalace JSON-mode classification) are non-streaming. Add `chat()` (non-streaming) to the v0.x surface, or punt? Recommend: add it Phase 2, since the recipe runtime makes it ~10 lines per language.
5. **Sync DB / sessions.** `realmwatch/chat_bridge.py` shares `sessions.db` with `mcp_cloud_chat.py`. Out of scope for the lib, but if a `cloud-chat-sessions` sub-package gets factored out later, JP's projects can stop reaching into each other's config dirs.

## 15. Success criteria

When this is done:

- [ ] `oracle` (Go) deletes its `providers/` directory and gains zero functionality regression.
- [ ] `gnome-speaks` (Python) changes one import line.
- [ ] `bestiary` (TypeScript) drops `@aws-sdk/client-bedrock-runtime`.
- [ ] `realmwatch/chat_bridge.py` (Python) replaces direct `AsyncAzureOpenAI` use with `cloud_chat.astream_chat`.
- [ ] Adding a new model is one JSON line. Adding a new OpenAI-compatible provider is one JSON object.
- [ ] Three languages, one canonical source, parity verified by shared fixtures in CI.
- [ ] `mcp_cloud_chat.py` shrinks dramatically (the provider routing branches all collapse into recipe lookups).
