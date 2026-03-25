# Supabase Self-Hosted MCP Setup Guide

This guide covers connecting AI coding tools (Cursor, Claude Code, Windsurf) to your self-hosted Supabase instance running on Coolify via the Model Context Protocol (MCP).

## Prerequisites

- Coolify instance with Supabase deployed via the official template
- SSH access to your Coolify server or Wireguard VPN configured
- An AI coding tool that supports MCP (Cursor, Claude Code, or Windsurf)

## Quick Start: SSH Tunnel Method

The fastest way to get started. Best for local development.

### 1. Create an SSH tunnel

```bash
ssh -L 8000:supabase-mcp:8000 user@your-coolify-server
```

This forwards local port 8000 to the MCP container on your server.

### 2. Configure your IDE

#### Cursor

Add to `.cursor/mcp.json` in your project root:

```json
{
  "mcpServers": {
    "supabase-self-hosted": {
      "url": "http://localhost:8000/mcp"
    }
  }
}
```

#### Claude Code

Run from your project directory:

```bash
claude mcp add supabase-self-hosted http://localhost:8000/mcp
```

#### Windsurf

Add to `.windsurf/mcp.json` in your project root:

```json
{
  "mcpServers": {
    "supabase-self-hosted": {
      "url": "http://localhost:8000/mcp"
    }
  }
}
```

### 3. Test the connection

Ask your AI tool to list tables or describe your database schema. If it works, you are connected.

## Production: Wireguard VPN Method

For persistent, secure access without managing SSH tunnels.

### 1. Set up Wireguard on Coolify

Deploy the Wireguard Easy template from Coolify's template library. Configure your client and connect.

### 2. Configure MCP domain and IP whitelist

When deploying or updating your Supabase instance, set these environment variables:

| Variable | Default | Description |
|---|---|---|
| `MCP_DOMAIN` | `mcp.${COOLIFY_DOMAIN}` | Custom subdomain for MCP access |
| `MCP_ALLOWED_IPS` | `10.0.0.0/24` | Comma-separated IPs allowed to connect |

For Wireguard, the default `10.0.0.0/24` range works out of the box. Adjust if your Wireguard network uses a different subnet.

### 3. Configure your IDE

Use the MCP domain instead of localhost:

#### Cursor

```json
{
  "mcpServers": {
    "supabase-self-hosted": {
      "url": "https://mcp.yourdomain.com/mcp"
    }
  }
}
```

#### Claude Code

```bash
claude mcp add supabase-self-hosted https://mcp.yourdomain.com/mcp
```

#### Windsurf

```json
{
  "mcpServers": {
    "supabase-self-hosted": {
      "url": "https://mcp.yourdomain.com/mcp"
    }
  }
}
```

## Multi-Instance Setup

Each Supabase instance gets a unique Traefik router name based on `COOLIFY_CONTAINER_NAME`, preventing collisions when running multiple Supabase projects on the same Coolify server.

### How it works

Each instance needs its own `MCP_DOMAIN`:

| Instance | MCP_DOMAIN |
|---|---|
| Project A | `mcp-projectA.yourdomain.com` |
| Project B | `mcp-projectB.yourdomain.com` |

Configure DNS records pointing each subdomain to your Coolify server. Traefik handles TLS automatically via Let's Encrypt.

### Per-project IDE config

Each project directory should have its own MCP config pointing to the correct domain:

```
project-a/
  .cursor/mcp.json  ->  https://mcp-projectA.yourdomain.com/mcp
project-b/
  .cursor/mcp.json  ->  https://mcp-projectB.yourdomain.com/mcp
```

## Troubleshooting

### Connection refused

- Verify the `supabase-mcp` container is running: `docker ps | grep supabase-mcp`
- Check container logs: `docker logs <supabase-mcp-container-id>`
- Ensure `supabase-db` is healthy (MCP depends on it)

### 403 Forbidden / IP not allowed

- Your IP is not in `MCP_ALLOWED_IPS`. Check your Wireguard client IP with `wg show`
- Update `MCP_ALLOWED_IPS` to include your client's VPN IP range
- For SSH tunnel method, this is not an issue since traffic appears from localhost

### Domain not resolving

- Verify DNS records for `MCP_DOMAIN` point to your Coolify server
- Check Traefik router exists: look for `supabase-mcp-*` in Traefik dashboard
- Ensure TLS certificate was issued (check Traefik logs)

### MCP tools return empty results

- Verify the `DATABASE_URL` is correct and the postgres user has read access
- Check `supabase-db` container health: the MCP server requires a healthy database connection

### Timeout or slow responses

- The MCP server queries your database directly. Complex schemas may take longer
- Check network latency between your IDE and the Coolify server
- For Wireguard, verify your VPN connection is stable
