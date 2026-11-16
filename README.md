# Hermes Agent — Command Code Go bridge setup

Personal setup for [Hermes Agent](https://github.com/NousResearch/hermes-agent)
(v0.23.1) using a Command Code Go subscription **directly** — no 9router,
no proxy hops, no extra API keys. The CLI's existing login is reused.

## Architecture

```
Hermes Agent  →  local bridge (commandcode_bridge.py, :8788)  →  Command Code API
  (provider:        pure-stdlib Python, OpenAI-compatible)
   commandcode)
```

The bridge reads `~/.commandcode/auth.json` (created by `cmd login`),
so **no token is ever copied or stored**. Everything model-side —
chat, vision, compression — goes through Command Code.

## Files in this repo

| File | What it is |
|---|---|
| `commandcode_bridge.py` | The bridge (v0.2 + vision patch: image parts → wire image) |
| `models.json` | Model catalog (102 models, incl. `meta/muse-ember`, `deepseek/deepseek-v4.1-flash`) |
| `config.yaml` | Sanitized config — `commandcode`, `hehehehe`, `zenmux` providers (no secrets) |
| `commandcode-plugin/` | Hermes model-provider plugin that overrides the bundled stub |
| `env-keys.example` | Which `.env` keys are needed (values redacted) |
| `index.html` | The landing page (existing) |

## Install

1. **Install Hermes**: `curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash`
2. **Log in to Command Code**: `cmd login` (creates `~/.commandcode/auth.json`)
3. **Copy files**:
   ```bash
   cp commandcode_bridge.py models.json ~/.hermes/
   cp -r commandcode-plugin ~/.hermes/plugins/model-providers/commandcode
   ```
4. **Merge `config.yaml`** — keep your existing providers; set:
   ```yaml
   model:
     default: z-ai/glm-5.3-flash
     provider: commandcode
     base_url: http://127.0.0.1:8788/v1
     key_env: COMMANDCODE_API_KEY
   ```
   plus the `commandcode` and `hehehehe` provider blocks from this repo's `config.yaml`.
5. **`.env`**: sync `COMMANDCODE_API_KEY` (and `HERMES_CUSTOM_COMMAND_CODE_API_KEY`) to the `apiKey` in `~/.commandcode/auth.json`.
6. **Run the bridge** (launchd, auto-restart on :8788):
   ```bash
   launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.commandcode.hermes-bridge.plist
   ```
   Or simply `python3 ~/.hermes/commandcode_bridge.py --host 127.0.0.1 --port 8788`.
7. **Test**:
   ```bash
   hermes --provider commandcode -m meta/muse-ember -z "Reply with exactly: EMBER_OK"
   hermes --provider hehehehe -m deepseek/deepseek-v4.1-flash -z "Reply with exactly: V41_OK"
   ```

## Notes

- The picker may need a restart or a search for "ember"/"v4.1" to show new models.
- Vision + compression run on `z-ai/glm-5.3-flash` via the bridge.
- `zenmux` is kept as a free-tier extra provider; the only fallback configured is OpenRouter `deepseek/deepseek-v4-flash`.
- Upstream bridge source: `github.com/MitoroMisaka/hermes-commandcode-provider`

## Current state (2026-11-15)

- Hermes v0.23.1, config format v53
- Default: `commandcode / z-ai/glm-5.3-flash`
- Live catalog: 100 models from Command Code
- Verified: `EMBER_OK`, `HEHE_OK`
