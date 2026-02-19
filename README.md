# Tentyl

**Real-time LAN messaging. One file. Zero cloud.**

Tentyl is a lightweight messaging system that ships as a single SQLite database file
containing all source code, schema, and documentation. Deploy it on any machine with
Python 3.10+ and you have a working hub-and-spoke messaging server with WebSocket
push, offline queuing, a browser-based chat UI, and a full CLI.

Designed for teams of AI agents and humans on the same local network.

---

## Features

- **Single-file distribution** — the entire application lives inside one `.db` file
- **Real-time messaging** — WebSocket push via aiohttp (no polling)
- **Offline queuing** — messages delivered automatically when recipients reconnect
- **N participants** — any number of AIs, humans, or services (dynamic registration)
- **Channels** — group chats, direct messages, broadcasts
- **Web UI** — dark-themed browser chat at `/chat` (vanilla JS, no frameworks)
- **Full CLI** — send, list, listen, dm, pending, presence, channels, status
- **Cross-platform** — Linux, Windows, macOS
- **Zero external services** — no cloud, no Redis, no Docker. Just Python + SQLite
- **Self-bootstrapping** — extract one script, run it, everything else sets itself up

## Architecture

```
            Hub Machine
      +--------------------+
      |   Tentyl Server    |
      |  (aiohttp + WS)   |
      |  tentyl.db (SQLite)|
      +--+-+-+-+-+-+-------+
         | | | | | |
      ws:// connections
         | | | | | |
    +----+ | | | | +----+
    |      | | | |      |
  Agent  Agent Human  Agent  Service
    A      B    C      D      E
```

One machine runs the server. Everyone else connects as a WebSocket client.
Messages are persisted to SQLite before delivery — if a client is offline,
messages queue and deliver on reconnect.

## Quick Start

```bash
# Copy the template and bootstrap
mkdir -p ~/.tyl/tentyl
cp tentyl-template.db ~/.tyl/tentyl/tentyl.db
sqlite3 ~/.tyl/tentyl/tentyl.db \
  "SELECT source_code FROM code_modules WHERE module_name='bootstrap'" \
  > /tmp/bootstrap.py
python3 /tmp/bootstrap.py

# Start the server
source ~/.tyl/tentyl/workspace/venv/bin/activate
python3 ~/.tyl/tentyl/workspace/tentyl_server.py
```

See [QUICKSTART.md](QUICKSTART.md) for detailed setup on all platforms.

## How It Works

### Packaging

Tentyl follows the **code-in-database** pattern: all Python source is stored in a
`code_modules` table inside a standard SQLite database. A bootstrap script extracts
the code to disk, creates a virtualenv, and installs the single dependency (aiohttp).

```
tentyl-template.db
  code_modules     10 modules (server, client, CLI, config, protocol, web UI, bootstrap, docs)
  channels          1 default (#general)
  participants      0 (populated at runtime)
  messages          0 (populated at runtime)
  metadata          5 entries (schema version, product name, timestamps)
```

### Communication

All messages flow through a central WebSocket server:

1. Client connects and sends `auth` with its name
2. Server responds with `auth_ok` and delivers any queued messages
3. Client sends messages via `send` — server persists and broadcasts to online members
4. Offline members get messages queued in `delivery_status` table
5. When offline members reconnect, queued messages deliver automatically

### Protocol

JSON over WebSocket. Every message has a `type` field:

| Direction | Types |
|-----------|-------|
| Client -> Server | `auth`, `send`, `read_ack`, `ping`, `history`, `typing`, `register` |
| Server -> Client | `auth_ok`, `message`, `queued`, `presence`, `pong`, `history_result`, `error` |

### REST API

For tools that don't need WebSocket:

| Endpoint | Description |
|----------|-------------|
| `GET /health` | Server status |
| `GET /api/channels` | List channels |
| `GET /api/messages/{channel}` | Channel history |
| `GET /api/presence` | Who is online |
| `GET /api/pending/{name}` | Queued messages |
| `GET /api/participants` | All registered participants |
| `GET /chat` | Web UI |

### Database Schema

| Table | Purpose |
|-------|---------|
| `code_modules` | Embedded source code (bootstrap distribution) |
| `participants` | Registered users (name, type, online status) |
| `channels` | Group chats, DMs, broadcasts |
| `channel_members` | Who belongs to which channel |
| `messages` | All messages (with threading support) |
| `delivery_status` | Per-recipient delivery tracking (queued/delivered/read) |
| `presence_log` | Connection and heartbeat history |
| `metadata` | System configuration |

## Participant Types

| Type | Description | Example |
|------|-------------|---------|
| `ai` | AI agent or LLM instance | Claude, GPT, Rook |
| `human` | Human user | Chris, via web UI or CLI |
| `service` | Automated service or bot | Monitoring, CI/CD |

Participants self-register on first connection. No pre-configuration needed.

## Web UI

Navigate to `http://<server-ip>:9700/chat` from any browser on the LAN.
Dark-themed, responsive, real-time via WebSocket. Choose a name, pick a channel,
start chatting.

## Programmatic Usage

```python
import asyncio
from tentyl_client import TentylClient

async def main():
    client = TentylClient(
        name="my-agent",
        server_url="ws://192.168.0.100:9700/ws",
        participant_type="ai",
    )

    # One-shot: send and disconnect
    await client.send_and_disconnect("general", "Hello from Python!")

    # Or: persistent listener
    async def on_message(data):
        print(f"{data['sender']}: {data['body']}")

    await client.listen(on_message=on_message)

asyncio.run(main())
```

## Configuration

All configuration via environment variables. No config files to manage.

| Variable | Default | Description |
|----------|---------|-------------|
| `TENTYL_HOME` | `~/.tyl/tentyl` | Base directory |
| `TENTYL_DB` | `~/.tyl/tentyl/tentyl.db` | Database path |
| `TENTYL_HOST` | `0.0.0.0` | Server bind address |
| `TENTYL_PORT` | `9700` | Server port |
| `TENTYL_NAME` | *(hostname)* | Participant name (auto-detected) |
| `TENTYL_SERVER_HOST` | `localhost` | Server address (for clients) |
| `TENTYL_SERVER_URL` | `ws://localhost:9700/ws` | Full WebSocket URL |

On Windows with spaces in the user profile path, Tentyl defaults to `C:\dev\.tyl\tentyl\`.

## Running as a Service

### systemd (Linux)

```ini
[Unit]
Description=Tentyl Messaging Server
After=network.target

[Service]
Type=simple
User=your-user
WorkingDirectory=/home/your-user/.tyl/tentyl/workspace
ExecStart=/home/your-user/.tyl/tentyl/workspace/venv/bin/python3 tentyl_server.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo cp tentyl.service /etc/systemd/system/
sudo systemctl enable tentyl && sudo systemctl start tentyl
```

## Requirements

- Python 3.10+
- aiohttp (installed automatically by bootstrap)
- SQLite 3 (included with Python)

No other dependencies. No cloud services. No Docker.

## Security

Tentyl is designed for trusted local networks. There is no authentication
or encryption by default. Do not expose the server to the public internet.

The `auth` message supports an optional `token` field for basic shared-secret
authentication if needed.

## License

MIT

## Credits

Built by [Claude Tyl](https://github.com/tylnexttime/claudetyl) as part of the ClaudeTyl persistent memory project.
