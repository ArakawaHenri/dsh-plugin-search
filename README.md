# dsh-plugin-search

DeepSeek Harness search provider (`native-preferring`) that prefers the initiating model's native web search on Anthropic Messages (`web_search_20250305`) and OpenAI Responses (`web_search`), falling back to the standard search provider (`deepseek-official` by default) when native search is unavailable or fails.

## Behavior

| Scenario | Behavior |
|---|---|
| Supported route (`messages` / `responses`) and `nativeSearch` omitted or `true` | Dispatch native search with the initiating session's model |
| Explicit `nativeSearch: false` (route or model level) | Use `standardProvider` directly |
| Chat Completions, unmapped route, or missing session/model | Use `standardProvider` directly |
| Native HTTP, transport, timeout, credential, parse, or tool error | Fall back to `standardProvider` |
| Native response where the search tool never ran (prose-only answer, error envelope) | Fall back to `standardProvider` |
| Native search ran and legitimately returned zero sources | Return empty result without fallback |
| Caller cancellation (`AbortSignal`) | Propagate cancellation immediately without fallback |

Native search makes at most one request. A failure goes directly to standard search. For a gateway that requires `web_search_preview`, set that route's `toolType` explicitly.

## Installation

DSH `>=0.2.1-rc.2` is accepted, including later prereleases under DSH's version-range check. There is no upper version cap: later releases can load the plugin as long as the web provider, initiating-session, loader, and credential interfaces it uses remain compatible.

When installed through DSH's plugin manager, this plugin uses the host's `@deepseek-ai/schemastery`. It is declared as a peer dependency because DSH resolves bundled packages for externally linked plugins through `peerDependencies`; an ordinary dependency requires a separate local installation.

The Schemastery peer range is `^3.18.5`, allowing compatible 3.x updates. The development dependency and lockfile retain the existing test baseline.

For a local checkout, add `link:/absolute/path/to/dsh-plugin-search` through DSH's plugin manager so DSH registers it as a linked plugin. With DSH's `autoInstallPeers: false` setting, pnpm can report the host-provided peer as missing; DSH supplies it when loading the registered plugin.

For standalone development or tests, install the development dependencies from the plugin directory:

```sh
npm ci
```

## Configuration

Routes declared in `@deepseek-ai/dsh-llm-pi-ai` with `anthropic-messages` or `openai-responses` work out of the box with `config: {}`. To disable native search for a whole route or a specific model, or to register a custom route, edit the profile's plugin configuration and reload the plugin:

```yaml
- id: local-search-prefer-native
  name: 'dsh-plugin-search'
  config:
    routes:
      my-gateway:
        nativeSearch: false        # whole-route opt-out
        models:
          model-without-search:
            nativeSearch: false    # per-model opt-out
      custom-route:
        api: responses             # explicit protocol for a non-pi-ai route
        baseURL: https://gateway.example
        apiKeyEnv: GATEWAY_API_KEY
```

Explicit route settings override the corresponding discovered fields. Setting `api: messages` or `api: responses` on a route whose chat protocol is Chat Completions upgrades its *search* dispatch to that protocol — chat traffic is untouched.

### Options

| Field | Default | Description |
|---|---|---|
| `standardProvider` | `'deepseek-official'` | Provider used for non-native searches and native fallbacks |
| `autoResolve` | `true` | Inherit `baseURL`, `apiKeyEnv`, and `headers` from `llm-pi-ai` |
| `maxTokens` | `2048` | Max output tokens for the native search request |
| `maxUses` | `3` | Max `web_search` uses per Messages request |
| `timeoutMs` | `60000` | Request + body timeout in milliseconds |
| `anthropicVersion` | `'2023-06-01'` | `anthropic-version` header for Messages requests |
| `messagesToolType` | `'web_search_20250305'` | Server-tool type for Messages |
| `responsesToolType` | `'web_search'` | Server-tool type for Responses |
| `debug` | `false` | Log the route, model, standard provider, and selection reason; native failures include the error, stack, and nested causes |
| `routes` | `{}` | Per-route overrides (`api`, `nativeSearch`, `baseURL`, `path`, `apiKey`, `apiKeyEnv`, `requireApiKey`, `headers`, `maxTokens`, `maxUses`, `timeoutMs`, `toolType`, `toolChoice`, `anthropicVersion`, `models`) |

Set `debug: true` in the plugin configuration and reload the plugin to diagnose fallbacks. A native failure log includes its HTTP status and provider message, timeout, credential or parsing error, or underlying transport cause when available. The native failure is logged even if the configured standard provider is missing. Successful native searches and caller cancellations do not produce fallback logs.

## Development

```sh
npm ci
npm test
npm run check
```

Opt-in live smoke tests against a local gateway:

```sh
DSH_SEARCH_LIVE=1 DSH_SEARCH_LIVE_BASE=http://127.0.0.1:3425 node --test test/live.test.js
```
