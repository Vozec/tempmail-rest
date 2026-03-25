# Tempmail-MCP

<p align="center">
  <img src=".github/image.png" alt="tempmail UI" width="900">
</p>

A self-hosted temporary email REST API with a built-in web UI and optional MCP server.
Multiple disposable-inbox providers are aggregated behind a single API; broken providers are automatically detected and disabled at startup.

---

## Features

- **Multi-provider** — mail.tm, tempmail.io, Gmail (IMAP), mailticking, tempmailo, tempail
- **Auto-fallback** — picks the first healthy provider in priority order
- **Circuit breaker** — providers that fail 3× in a row are auto-disabled; re-enable via API
- **Startup health-check** — broken providers are disabled before the first request
- **Web UI** — tab-based email client, multi-account, localStorage persistence, auto-refresh, browser notifications
- **Shared/pinned emails** — pin mailboxes server-side so every client can use them
- **Swagger docs** — `/docs`
- **MCP server** — optional, usable standalone by Claude / any MCP client
- **Cloudflare bypass** — via [FlareSolverr](https://github.com/FlareSolverr/FlareSolverr) for providers that require it

---

## Quick start (Docker Compose)

```bash
cp .env.example .env
# edit .env if needed (Gmail credentials, ports, …)
docker compose up -d
```

- Web UI → http://localhost:8000
- Swagger → http://localhost:8000/docs

---

## Configuration (`.env`)

| Variable | Default | Description |
|---|---|---|
| `HOST` | `0.0.0.0` | Bind address |
| `PORT` | `8000` | HTTP port |
| `RELOAD` | `false` | Uvicorn hot-reload (dev only) |
| `ENABLE_FRONTEND` | `true` | Serve the web UI |
| `FLARESOLVERR_URL` | `http://localhost:8191` | FlareSolverr endpoint |
| `HEALTH_CHECK_ON_STARTUP` | `true` | Probe all providers at startup and disable failures |
| `SHARED_EMAILS_PATH` | `shared_emails.json` | Path to the server-side pinned-emails store |
| `GMAIL_EMAIL` | — | Gmail address (for Gmail IMAP provider) |
| `GMAIL_APP_PASSWORD` | — | Gmail app password |

---

## REST API

### System

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/health` | Health status of all providers (200 if all ok, 207 if degraded) |

**Response `GET /api/health`**
```json
{
  "healthy": true,
  "providers": {
    "mail.tm":    { "status": "ok" },
    "mailticking": { "status": "disabled", "failures": 3 }
  }
}
```

---

### Providers

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/providers` | List all providers with enabled/failure state |
| `POST` | `/api/providers/{name}/disable` | Manually disable a provider |
| `POST` | `/api/providers/{name}/enable` | Re-enable a provider and reset failure counter |
| `GET` | `/api/domains?name=<provider>` | List available domains for a provider |

**Response `GET /api/providers`**
```json
[
  { "name": "gmail",      "disabled": false, "failures": 0 },
  { "name": "mail.tm",    "disabled": false, "failures": 0 },
  { "name": "mailticking","disabled": true,  "failures": 3 }
]
```

---

### Email

`name` is optional on every endpoint — omitted means "auto" (first healthy provider in priority order).

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/email?name=<provider>` | Create a new temporary mailbox |
| `GET` | `/api/email/{email}/messages?token=X&name=Y` | List messages for a mailbox |
| `GET` | `/api/email/{email}/message/{id}?token=X&name=Y` | Get a full message (with body) |
| `DELETE` | `/api/email/{email}?token=X&name=Y` | Delete the mailbox |

**Request `POST /api/email`**
```json
{
  "min_name_length": 10,
  "max_name_length": 10,
  "domain": null
}
```

**Response `POST /api/email`**
```json
{
  "email": "randomname@mail.tm",
  "token": "eyJ...",
  "provider": "mail.tm"
}
```

**Response `GET /api/email/{email}/messages`**
```json
[
  {
    "id": "abc123",
    "from_addr": "\"Alice\" <alice@example.com>",
    "to_addr": "randomname@mail.tm",
    "subject": "Your verification code",
    "body_text": null,
    "body_html": null,
    "created_at": "1742900000",
    "attachments": []
  }
]
```

**Response `GET /api/email/{email}/message/{id}`**
```json
{
  "id": "abc123",
  "from_addr": "\"Alice\" <alice@example.com>",
  "to_addr": "randomname@mail.tm",
  "subject": "Your verification code",
  "body_text": "Your code is 123456",
  "body_html": "<html>...</html>",
  "created_at": "1742900000",
  "attachments": [
    { "filename": "invoice.pdf", "content_type": "application/pdf", "size": 42300, "url": null }
  ]
}
```

---

### Shared / Pinned emails

Pin a mailbox server-side so every client can see and reuse it.

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/shared` | List all pinned email accounts |
| `POST` | `/api/shared` | Pin an email (visible to all clients) |
| `DELETE` | `/api/shared/{email}` | Unpin an email |

**Request `POST /api/shared`**
```json
{
  "email": "randomname@mail.tm",
  "token": "eyJ...",
  "provider": "mail.tm",
  "label": "My shared inbox"
}
```

**Response `GET /api/shared`**
```json
[
  {
    "email": "randomname@mail.tm",
    "token": "eyJ...",
    "provider": "mail.tm",
    "label": "My shared inbox",
    "pinned_at": 1742900000
  }
]
```

---

## Providers

Priority order (first healthy provider wins):

| Priority | Name | Type | Requires FlareSolverr | Notes |
|---|---|---|---|---|
| 1 | `gmail` | IMAP | No | Needs `GMAIL_EMAIL` + `GMAIL_APP_PASSWORD` |
| 2 | `mail.tm` | REST API | No | Most reliable fallback |
| 3 | `tempmail.io` | REST API | No | Reliable |
| 4 | `mailticking` | Scraping | Yes | CF-protected |
| 5 | `tempmailo` | Scraping | Yes | CF-protected |
| 6 | `tempail` | Scraping | Yes | May be blocked by reCAPTCHA |

At startup, each provider is probed with a real `create_email` call. Any provider that fails is auto-disabled before serving requests.

---

## MCP server (optional)

```bash
python -m src.mcp_server
```

Tools exposed: `list_providers`, `get_domains`, `create_email`, `get_messages`, `read_message`, `delete_email`.

Claude Desktop config (`~/.config/claude/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "tempmail": {
      "command": "python",
      "args": ["-m", "src.mcp_server"],
      "cwd": "/path/to/tempmail-api"
    }
  }
}
```

---

## Development

```bash
pip install -r src/requirements.txt
RELOAD=true python -m src.main
```

FlareSolverr (required for CF-protected providers):

```bash
docker run -p 8191:8191 ghcr.io/flaresolverr/flaresolverr:latest
```
