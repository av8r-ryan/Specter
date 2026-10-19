# Specter - Development Guide

## Project Overview

Specter is a real-time SIEM security dashboard that connects to Wazuh and Suricata, displays live alerts, and includes an AI chat panel (Claude) that can autonomously search historical alerts.

## Tech Stack

- **Runtime**: Bun API/package manager; Node.js 22+ runs Vite (22.22.2 tested)
- **Language**: TypeScript
- **Frontend**: Vue 3 + Vite + Tailwind CSS (CarbeneAI dark theme)
- **Backend**: Bun HTTP + WebSocket server
- **AI**: Claude CLI over SSH (default), Ollama, or Anthropic API (with Wazuh search tools)

## Project Structure

```
Specter/
├── apps/
│   ├── server/          # Bun HTTP + WebSocket server
│   │   └── src/
│   │       ├── index.ts              # Main server (HTTP + WebSocket)
│   │       ├── types.ts              # Shared TypeScript interfaces
│   │       ├── alert-ingest.ts       # Wazuh poller + JSONL watcher
│   │       ├── pai-client.ts         # Claude AI chat with tool use
│   │       ├── triage-provider.ts    # Provider selection and CLI/Ollama triage
│   │       ├── alert-history.ts      # Validated remote history lookup
│   │       └── suricata-suppression.ts  # SSH-based rule suppression
│   └── client/          # Vue 3 Vite app
│       └── src/
│           ├── App.vue              # Main layout, split panels
│           ├── types.ts             # Client-side types
│           ├── components/          # Vue components
│           └── composables/         # Vue composables
├── .env.example         # Environment variable template
├── manage.sh            # Start/stop script
├── start-specter.sh     # Foreground launcher for systemd
├── specter.service      # System systemd unit template
└── deploy/specter.service # User systemd unit template
```

## Development

```bash
# Install dependencies
(cd apps/server && bun install)
(cd apps/client && bun install)

# Start both (requires .env configured)
./manage.sh start

# Foreground service launcher (loads the root .env)
./start-specter.sh
```

Both launch commands stay in the foreground; run one at a time. For individual
`bun run dev` sessions, export the root `.env` in each terminal first: Bun is
started inside an app directory. For systemd installation and service lifecycle
commands, follow [docs/deployment.md](docs/deployment.md).

All four browser callers use `apps/client/src/lib/endpoints.ts`: `/api` and current-origin
`/stream` by default. API and Vite dev/preview share exact Host/Origin HTTP admission
and bind to `127.0.0.1`. API `/stream` upgrades use the same guard after the
proxy preserves Host and Origin. Vite HMR has its own Host/token checks and is
disabled in the network fixtures. Read [docs/network-boundary.md](docs/network-boundary.md)
before changing binds, browser origins, proxy routes, or public `VITE_*` overrides.
Do not treat the boundary or reported identity headers as authentication.

## Environment Variables

Copy `.env.example` to `.env` and configure. Required:
- `WAZUH_DASHBOARD_URL` - Your Wazuh server URL
- `WAZUH_DASHBOARD_PASSWORD` - Your Wazuh admin password

AI configuration is optional for alert ingestion and deterministic scoring.
`TRIAGE_PROVIDER` selects the Cloud toggle's backend:

- `claude` (default): set `CLAUDE_CLI_SSH_HOST` to a host with an authenticated
  Claude CLI. CLI/SSH failure falls back to the configured Ollama service.
- `ollama`: set `OLLAMA_URL` and `OLLAMA_MODEL` for your local service.
- `anthropic`: configure `ANTHROPIC_API_KEY` (or `WAZUH_PAI_API_KEY`) for the paid
  Messages API. An Anthropic key is not required by the other providers.

The Local toggle explicitly selects Ollama. Optional historical correlation uses
`QUERY_INDEXER_PATH` and `QUERY_INDEXER_ENV`; both must be absolute paths on the
SSH host. Tilde-prefixed and relative paths are rejected.

## Key Architecture Notes

- Server polls Wazuh Indexer every 30s via Dashboard API proxy (`/api/console/proxy`)
- WebSocket streams alerts to all connected clients in real-time
- Provider routing lives in `triage-provider.ts`; Anthropic tool use lives in
  `pai-client.ts`. CLI triage receives server-fetched history when configured;
  Ollama receives the supplied alert context without that lookup.
- Alert suppression requires SSH access to Suricata/Wazuh hosts
- Wazuh polling and direct search verify HTTPS by default; configure private trust
  with `WAZUH_CA_FILE`, never a process-wide TLS bypass. See `docs/wazuh-tls.md`.

## Validation

```bash
python3 tests/test_launcher.py LauncherTests
bash -n start-specter.sh
(cd apps/server && bun test && bun run typecheck)
(cd apps/client && bun test && bun run build)
```

On Linux, also run `python3 tests/test_launcher.py ServiceTemplateTests` with
`systemd-analyze` installed. Launcher tests use stub processes in temporary
checkouts. Server integration tests use dummy Wazuh configuration and stub
external HTTP calls. Do not use live SIEM, SSH, or model endpoints for these checks.

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | /health | Health check |
| GET | /alerts/recent | Get recent alerts |
| GET | /alerts/stats | Alert counts by severity |
| POST | /alerts/ingest | Ingest alerts via HTTP (n8n webhook) |
| POST | /chat | Send message to AI analyst |
| GET | /chat/prompts | Get quick prompt templates |
| POST | /alerts/search | Search Wazuh Indexer |
| POST | /alerts/suppress | Suppress a rule |
| GET | /alerts/suppressed | List suppressed rules |
| WS | /stream | WebSocket alert stream |

## Alert identity

All ingestion sources call `addAlerts()` to reserve process-local IDs before
asynchronous scoring. Never derive IDs from the bounded alert store length.
See [alert identity and retention](docs/alert-identities.md) for tests and restart limits.

## Severity Levels (Wazuh)

| Level | Severity |
|-------|----------|
| 12+ | Critical |
| 7-11 | High |
| 3-6 | Medium |
| 0-2 | Low |

## Untrusted Markdown output

Keep assistant HTML insertion behind `apps/client/src/lib/markdown.ts`.
The helper must sanitize after Markdown parsing; do not return raw parsed HTML
or post-process it into additional HTML. Preserve the strict tag, attribute,
and HTTP(S)-only link policy. Model-provided images must not fetch resources.

From `apps/client`, run `bun install --frozen-lockfile`,
`bunx playwright install chromium`, `bun --no-env-file run test:markdown`, and `bun run build`.
The browser test CLI uses Node.js 22 alongside Bun 1.3.14. Tests mount the real
component with synthetic data on an isolated loopback fixture and block requests
after loading. Do not use live services or credentials to run these checks.
Run `bun --no-env-file test` separately for client logic. The client `bunfig.toml`
keeps this runner within `src`; browser specs belong to Playwright.
See `docs/markdown-safety.md` for policy, maintenance, and validation limits.
