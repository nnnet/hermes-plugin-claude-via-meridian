# claude-via-meridian

Hermes model-provider plugin that speaks the **Anthropic Messages**
protocol against the **host Meridian** proxy at `127.0.0.1:3456`.

## What it does

Subclasses `AnthropicProfile` (from the gateway image) and forces
`base_url` to the host Meridian endpoint so requests go:

```
Hermes gateway (in container) ──/v1/messages──> 127.0.0.1:3456 (host Meridian)
                                                      │
                                                      └──> Claude API (via OAuth subscription)
```

Wire protocol: byte-identical to native `anthropic` — `x-api-key`
header, `anthropic-version`, native `/v1/messages` shape. Just pointed
at Meridian instead of `api.anthropic.com`.

## Why this exists alongside `claude-agent-sdk`

- **`claude-agent-sdk`** plugin carries CLI-bridge semantics — the
  runtime recognises the name and wraps the request in
  agent-CLI transport conventions (e.g. `base_url` rewriting to
  `claude-agent-sdk://`).
- **`claude-via-meridian`** is a plain HTTP-to-Meridian path with no
  special-casing in `runtime_provider` — useful when the caller wants
  the standard Anthropic SDK to hit Meridian directly without
  agent-side rewriting.

Without this plugin, `provider: claude-via-meridian` in config used to
resolve to a generic-custom path in `runtime_provider` that fell back to
`chat_completions` (OpenAI shape) and hit `/chat/completions` on
Meridian, returning 404 because Meridian only speaks Anthropic.

## Configuration

`plugin.yaml` is bundled-defaults. Usage:

```yaml
# ~/.hermes/config.yaml
providers:
  some-anthropic-model:
    provider: claude-via-meridian
    model: claude-sonnet-4-6
    api_key: not-needed  # Meridian handles auth via host subscription
```

## Mounting

External plugin: pulled by
[`nnnet/AiManager:infra/hermes/scripts/sync-external-plugins.sh`](https://github.com/nnnet/AiManager/blob/prod/infra/hermes/scripts/sync-external-plugins.sh)
into `sources/hermes-external-plugins/claude-via-meridian/`, then
bind-mounted into the container at
`/opt/hermes/plugins/model-providers/claude-via-meridian/`.

Note: `127.0.0.1` inside the container is mapped to the host's loopback
via `host.docker.internal` (compose config). Host Meridian must be
running (`scripts/meridian.sh start`) for this provider to work.

## Related

- `nnnet/hermes-agent` — gateway fork; this plugin overlays the
  baked-in `model-providers/` dir via bind-mount, no image rebuild
  required for edits.
- `infra/hermes/scripts/meridian.sh` (in AiManager) — host-side Meridian
  lifecycle.

## License

MIT — see LICENSE.
