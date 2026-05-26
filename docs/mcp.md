# Immich MCP Server

The `claw2immich` service exposes the Immich API as an MCP server, allowing AI agents (e.g. Alfred on OpenClaw) to interact with the photo library.

## Service

Defined in `docker-compose.yml` as `claw2immich` (`immich_mcp` container).

- Image: `ghcr.io/joeru/claw2immich:latest`
- Transport: `streamable-http`
- Endpoint: `http://192.168.1.120:8765/mcp`
- Permission profile: `read_write` (allows asset upload)

## Setup

### 1. Create an Immich API key

In the Immich UI: **Account Settings → API Keys → Create new key** (e.g. `alfred-mcp`).

### 2. Add the key to `.env`

```bash
echo 'IMMICH_MCP_API_KEY=<your-key>' >> /opt/cortex-immich/.env
```

### 3. Start the service

```bash
cd /opt/cortex-immich
docker compose up -d claw2immich
```

## Connect to OpenClaw

In `openclaw.json`, under `mcp.servers`:

```json
"immich": {
  "command": "curl",
  "args": [
    "-s", "-X", "POST",
    "http://192.168.1.120:8765/mcp",
    "-H", "Content-Type: application/json",
    "--data-binary", "@-"
  ]
}
```

## Volumes

| Host path | Container path | Purpose |
|---|---|---|
| `/mnt/synology/Photo/upload` | `/mnt/inbox` | Files Alfred can upload by path |

## Available tools (read_write profile)

- `immich_uploadAsset` — upload a file by path
- `immich_updateAsset`, `immich_deleteAssets` — edit/delete assets
- `immich_getAllAssets`, `immich_getAssetById`, `immich_searchAssets` — search
- `immich_createAlbum`, `immich_getAllAlbums`, `immich_addAssetsToAlbum` — albums
- `downloadAsset` — download an asset
