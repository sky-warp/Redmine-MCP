# Redmine MCP

Connect an AI assistant to your Redmine projects through a local MCP server. You can ask the assistant to find issues, explain project activity, read wiki pages, or work with time entries and attachments. It can also make changes when your Redmine account has permission.

This repository is a small Docker Compose setup for [jztan/redmine-mcp-server v2.8.0](https://github.com/jztan/redmine-mcp-server/tree/v2.8.0). It runs the published image `ghcr.io/jztan/redmine-mcp-server:2.8.0`; the server itself is maintained upstream.

## Differences from upstream

The upstream repository contains the server's source code and full documentation. This repository focuses on running it locally, with three changes to the upstream Compose setup:

- Uses the published `2.8.0` image, while upstream Compose builds the image from source.
- Exposes port `8000` only on `127.0.0.1`, while upstream Compose publishes it on all host interfaces.
- Adds an `attachments/` bind mount alongside `logs/` and `data/`, keeping downloaded attachments across container recreation until they expire or are cleaned up.

The server code and Redmine tools come directly from the upstream image.

## How it works

```text
Your AI assistant → local MCP server → your Redmine instance
```

The setup uses one Redmine API key, so every request runs as that Redmine account. The local MCP endpoint has no separate client authentication in this mode. Keep the Compose port binding at `127.0.0.1:8000:8000` so it is reachable only from your computer. Your AI client receives the Redmine information returned by its tool calls.

## What you need

- Docker running on your computer, with Docker Compose available as `docker compose`.
- A Redmine instance reachable from the container.
- Redmine's REST API enabled under **Administration → Settings → API**.
- An API key from your Redmine **My account** page. Use an account with the permissions you want the assistant to have.
- An AI client that supports MCP over HTTP, also called Streamable HTTP.

You do not need to install Python or build the server locally.

## Start the server

Run these commands from this repository's directory.

1. Create your private configuration file:

   ```sh
   cp -n .env.docker.example .env.docker
   ```

   The `-n` option preserves an existing `.env.docker`. If you already have one, keep using it.

2. Open `.env.docker` in your editor and set:

   ```dotenv
   REDMINE_URL=https://redmine.example.com
   REDMINE_API_KEY=replace-with-your-api-key
   ```

   Replace both example values with your own settings. Keep this file private.

3. Check the configuration, download the pinned image, and start it:

   ```sh
   docker compose config --quiet
   docker compose pull
   docker compose up -d
   ```

   The quiet configuration check avoids printing the resolved environment, which contains your API key.

4. Check the server:

   ```sh
   docker compose ps
   curl -fsS http://127.0.0.1:8000/health
   ```

   For a working Redmine connection, the JSON response should contain `"status":"ok"` and `"checks":{"redmine":"ok"}`. Other fields may also be present.

   Version 2.8.0 returns HTTP 200 even when the JSON reports `"status":"degraded"`. Missing credentials can produce `"redmine":"unconfigured"`. A successful `curl` command or Docker's “healthy” status alone does not confirm that Redmine is connected.

## Connect your AI assistant

Add an MCP server in your client's settings using:

| Setting | Value |
| --- | --- |
| Name | `redmine` |
| Transport | HTTP / Streamable HTTP |
| Server URL | `http://127.0.0.1:8000/mcp` |

The exact settings format depends on the client. The server URL ends in `/mcp`; `/health` is for diagnostics. This local address works for clients running on the same computer. A service running in the cloud cannot reach your computer through this address.

Once connected, try a request such as:

- “List the Redmine projects I can access.”
- “Show my open issues.”
- “Summarize issue 123, including its recent notes.”

Reading is a useful first check. Creating issues, updating them, logging time, and other writes use the same account and its permissions. To block write operations, optionally add `REDMINE_MCP_READ_ONLY=true` to `.env.docker`, then recreate the container as described below. This setting is optional and is not enabled by the example configuration.

Available tools and optional Redmine plugin features are documented in the [upstream v2.8.0 tool reference](https://github.com/jztan/redmine-mcp-server/blob/v2.8.0/docs/tool-reference.md).

## Configuration

The example file includes these settings:

| Setting | Purpose |
| --- | --- |
| `REDMINE_AUTH_MODE=legacy` | Use one shared Redmine account through the configured API key. |
| `REDMINE_URL` | Your Redmine base URL, including any installation subpath. |
| `REDMINE_API_KEY` | The private API key used to access Redmine. |
| `SERVER_HOST=0.0.0.0` | Listen on the container's network interfaces so Docker can forward requests. The host port remains restricted to loopback by Compose. |
| `SERVER_PORT=8000` | Port inside the container. It must match the Compose mapping and health check. |
| `PUBLIC_HOST=localhost` / `PUBLIC_PORT=8000` | Address used in generated attachment download links. |
| `ATTACHMENTS_DIR=/app/attachments` | Store downloaded attachments in the mounted attachment directory. |
| `AUTO_CLEANUP_ENABLED=true` | Enable automatic cleanup of expired downloaded attachments. |
| `CLEANUP_INTERVAL_MINUTES=10` | Check for expired attachments every ten minutes. |
| `ATTACHMENT_EXPIRES_MINUTES=60` | Generated attachment download links expire after sixty minutes. |

For advanced authentication, custom certificates, or optional plugin settings, see the [upstream configuration documentation for v2.8.0](https://github.com/jztan/redmine-mcp-server/blob/v2.8.0/README.md#environment-variables-configuration).

## Folders

Stopping the container keeps `.env.docker` and the host directories mounted by Compose:

| Directory | Container location | Purpose |
| --- | --- | --- |
| `logs/` | `/app/logs` | Space for application log files. Standard server output is available through `docker compose logs`. |
| `data/` | `/app/data` | Persistent application data, when used by the server's configuration. |
| `attachments/` | `/app/attachments` | Downloaded attachments and their metadata. Expired downloads may be cleaned up automatically. |

These directories can contain private Redmine information. Keep them and your real environment files out of Git. Commit only sanitized configuration examples, and redact credentials and private information before sharing logs.

## Troubleshooting

| Problem | What to check |
| --- | --- |
| Docker commands fail | Start Docker and confirm `docker compose version` works. |
| `.env.docker` is missing | Copy the example file using the setup command above, then fill in your own values. |
| Health reports `degraded`, HTTP 401, or HTTP 403 | Check the Redmine URL, API key, REST API setting, and account permissions. Inspect the JSON response and server logs. |
| Redmine cannot be reached | The container needs network access to Redmine, including any required VPN. Inside a container, `localhost` points to that container. For Redmine running on your host, Docker Desktop provides `host.docker.internal`; Linux needs a reachable host address or an explicitly configured host gateway. |
| Port 8000 is already in use | Stop the conflicting service, or update the host port in Compose together with the client URL and `PUBLIC_PORT`. |
| A directory is not writable on Linux | The image runs as user/group `1000:1000`. Give that user appropriate access to the mounted directories; avoid making them writable by everyone. |
| Certificate verification fails | Configure the appropriate CA certificate using the upstream guide. A certificate path inside the container requires the certificate to be mounted there. |
| The AI client cannot connect | Confirm the container is running and the client uses HTTP with `http://127.0.0.1:8000/mcp`. Check the client's connection instructions. |

More help: [upstream troubleshooting for v2.8.0](https://github.com/jztan/redmine-mcp-server/blob/v2.8.0/docs/troubleshooting.md) and [Redmine REST API documentation](https://www.redmine.org/projects/redmine/wiki/Rest_api).
