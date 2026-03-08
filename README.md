# MCP Stack (WSL)

MCP-Server-Stack für Claude Code in WSL. Ersetzt Docker Desktop als MCP-Host.

## Architektur

```
Claude Code (WSL) → localhost:3000/sse → mcp-gateway (Docker)
                                              ├── github-official
                                              ├── n8n
                                              ├── obsidian
                                              ├── prometheus
                                              ├── ffmpeg
                                              ├── youtube_transcript
                                              └── docker-docs
```

## Voraussetzungen

- Docker Engine in WSL (systemd-managed)
- `~/.docker/mcp/` mit Konfigurationsdateien

## Konfiguration

### Secrets (.env)

Datei `.env` im Projektverzeichnis anlegen (nicht committen):

```
GITHUB_PERSONAL_ACCESS_TOKEN=ghp_...
OBSIDIAN_API_KEY=...
N8N_API_KEY=...
```

### Aktive Server (registry.yaml)

`~/.docker/mcp/registry.yaml` – Server hier eintragen/entfernen:

```yaml
registry:
  github-official:
    ref: ""
  n8n:
    ref: ""
  # weiterer Server:
  # context7:
  #   ref: ""
```

### Server-Konfiguration (config.yaml)

`~/.docker/mcp/config.yaml` – Konfigurationswerte für einzelne Server:

```yaml
n8n:
  api_url: http://<HOST_IP>:5678/
```

## Starten / Stoppen

```bash
# Manuell
docker compose up -d
docker compose down
docker logs mcp-gateway -f

# Via systemd (automatisch beim WSL-Start)
sudo systemctl start mcp-stack
sudo systemctl stop mcp-stack
sudo systemctl status mcp-stack
```

## Externen MCP-Server hinzufügen

### Option A: Server aus dem Docker MCP Katalog

Eintrag in `~/.docker/mcp/registry.yaml` hinzufügen:

```yaml
registry:
  context7:      # Name aus docker-mcp.yaml Katalog
    ref: ""
```

→ Container neu starten: `docker compose restart`

### Option B: Eigener MCP-Server (stdio-basiert)

Neuen Service in `docker-compose.yml` hinzufügen:

```yaml
services:
  mcp-gateway:
    # ... bestehende Konfiguration ...

  mcp-meinserver:
    image: supercorp/supergateway
    ports:
      - "3001:8000"
    command: >
      --stdio "npx -y @modelcontextprotocol/server-meinserver"
      --port 8000
      --ssePath /sse
      --messagePath /message
    restart: unless-stopped
```

Dann in `~/.claude.json` registrieren:

```json
{
  "mcpServers": {
    "MCP_DOCKER": { "type": "sse", "url": "http://localhost:3000/sse" },
    "mein-server": { "type": "sse", "url": "http://localhost:3001/sse" }
  }
}
```

### Option C: Remote MCP-Server (HTTP/SSE)

Direkt in `~/.claude.json`:

```json
{
  "mcpServers": {
    "MCP_DOCKER": { "type": "sse", "url": "http://localhost:3000/sse" },
    "remote-server": {
      "type": "sse",
      "url": "http://192.168.0.xxx:3001/sse",
      "headers": { "Authorization": "Bearer TOKEN" }
    }
  }
}
```

## Automatischer Start

Der systemd-Service `mcp-stack` startet automatisch nach dem WSL-Boot:

```
/etc/systemd/system/mcp-stack.service → docker compose up -d
```

## Migration zum Heimserver

Wenn der MCP-Stack auf den Heimserver (LXC 101) umzieht:

1. `mcp-stack/` Verzeichnis kopieren
2. `~/.docker/mcp/` kopieren
3. `.env` mit Secrets anlegen
4. `docker compose up -d`
5. In `~/.claude.json` URL auf `http://<heimserver-ip>:3000/sse` ändern
