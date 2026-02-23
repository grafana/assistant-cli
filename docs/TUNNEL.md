# Tunnel

The Assistant Tunnel allows Grafana Assistant to execute tools on your machine — reading your project files, searching code, and optionally running shell commands. This gives the assistant direct context about your codebase and infrastructure.

## How It Works

When you start the tunnel, your machine connects to Grafana and listens for tool requests. When the assistant needs to read a file or run a command, the request is routed through the tunnel to your machine, executed locally, and the result is sent back.

The tunnel only responds to requests from the assistant — it does not expose any network ports or services on your machine.

## Quick Start

```bash
# 1. Authenticate (opens browser)
grafana-assistant auth

# 2. Add a project for the assistant to access
grafana-assistant config add-project my-app ~/projects/my-app

# 3. Start the tunnel
grafana-assistant tunnel connect
```

That's it. The assistant can now read files in your `my-app` project during chat sessions.

## Authentication

The tunnel uses the same credentials as the rest of the CLI. Run `grafana-assistant auth` to authenticate -- the token includes the `tunnel:connect` scope, so no separate tunnel auth step is needed.

### Checking Auth Status

```bash
grafana-assistant config list
```

This shows all configured instances and their authentication status:

```
CURRENT  NAME     URL                           AUTH               API ENDPOINT
*        mystack  https://mystack.grafana.net   Authenticated      https://mystack.grafana.net/api/cli/v1
         prod     https://prod.grafana.net      Not authenticated  -
```

## Connecting

### Basic Connection

```bash
# Connect using the current instance
grafana-assistant tunnel connect

# Connect to a specific instance
grafana-assistant tunnel connect --instance prod

# Connect all authenticated instances at once
grafana-assistant tunnel connect --all
```

The tunnel runs in the foreground until you press `Ctrl+C`.

### Tools

The tunnel provides two tools that the assistant can use:

| Tool           | Default  | Description                                                      |
| -------------- | -------- | ---------------------------------------------------------------- |
| **Filesystem** | Enabled  | Read-only access to project files, search, and directory listing |
| **Terminal**   | Disabled | Execute shell commands on your machine                           |

Control which tools are active:

```bash
# Filesystem only (default)
grafana-assistant tunnel connect

# Filesystem and terminal
grafana-assistant tunnel connect --terminal

# Disable filesystem (terminal only)
grafana-assistant tunnel connect --filesystem=false --terminal
```

> **Caution:** The terminal tool allows the assistant to execute shell commands on your machine. Only enable it if you need it, and review the [security configuration](CONFIGURATION.md#terminal-tool) to restrict what commands are allowed.

### Verbose Output

Use `--verbose` for detailed logging, helpful for troubleshooting:

```bash
grafana-assistant tunnel connect --verbose
```

## Managing Projects

Projects are named directories that the assistant can access through the filesystem tool. The assistant references files using project-relative paths like `my-app/src/main.go`.

### Adding Projects

```bash
# Via the CLI
grafana-assistant config add-project my-app ~/projects/my-app
grafana-assistant config add-project infra ~/work/infrastructure

# Or ask the assistant to do it during a chat session:
# "Add the project 'my-app' at ~/projects/my-app"
```

### Listing and Removing

```bash
grafana-assistant config list-projects
grafana-assistant config remove-project my-app
```

Projects can also be added and removed dynamically during an active tunnel session — changes take effect immediately without restarting the tunnel.

## Running as a Daemon

For always-on tunnel access, install the tunnel as a background service. The daemon automatically starts on login and restarts if it crashes.

### Installing

```bash
grafana-assistant tunnel daemon install
```

This creates a platform-specific service:

- **macOS**: launchd user agent (`~/Library/LaunchAgents/`)
- **Linux**: systemd user service

By default, the daemon:

- Connects to the current instance
- Enables tools based on your config file
- Starts automatically on login
- Restarts automatically on crash

#### Install Options

```bash
# Connect a specific instance
grafana-assistant tunnel daemon install --instance prod

# Connect all authenticated instances
grafana-assistant tunnel daemon install --all

# Don't start on login
grafana-assistant tunnel daemon install --start-on-login=false

# Verbose logging
grafana-assistant tunnel daemon install --verbose
```

> Tool enablement (filesystem/terminal) for the daemon is controlled by the config file, not install flags. See [Tunnel Configuration](CONFIGURATION.md#tunnel-configuration) for details.

### Managing the Daemon

```bash
# Start the service
grafana-assistant tunnel daemon start

# Stop the service
grafana-assistant tunnel daemon stop

# Restart the service
grafana-assistant tunnel daemon restart

# Check status
grafana-assistant tunnel daemon status

# View logs
grafana-assistant tunnel daemon logs

# Stream logs in real time
grafana-assistant tunnel daemon logs --follow

# Show more history
grafana-assistant tunnel daemon logs --lines 200
```

### Uninstalling

```bash
grafana-assistant tunnel daemon uninstall
```

This stops the service if running and removes the service configuration files.

### Example: Daemon Setup

A typical setup for always-on tunnel access:

```bash
# 1. Authenticate
grafana-assistant auth

# 2. Add your projects
grafana-assistant config add-project webapp ~/projects/webapp
grafana-assistant config add-project api ~/projects/api-server

# 3. Install and start the daemon
grafana-assistant tunnel daemon install
grafana-assistant tunnel daemon start

# 4. Verify it's running
grafana-assistant tunnel daemon status
```

Now the assistant can access your project files whenever you're chatting, even from the Grafana web UI.

## Docker

The tunnel can also run as a Docker container, which provides sandboxing and a consistent set of pre-installed tools. See [DOCKER.md](DOCKER.md) for the full guide.

Quick example:

```bash
# Authenticate (one-time)
docker run --rm -it \
  -p 54321:54321 \
  -v ~/.config/grafana-assistant:/home/tunnel/.config/grafana-assistant \
  grafana/assistant-cli:latest \
  auth --bind 0.0.0.0

# Run the tunnel
docker run -d \
  --name grafana-assistant-cli \
  --restart unless-stopped \
  -v ~/.config/grafana-assistant:/home/tunnel/.config/grafana-assistant:ro \
  -v ~/projects/my-app:/projects/my-app:ro \
  grafana/assistant-cli:latest \
  tunnel connect --filesystem
```

## Security

The tunnel is designed with security in mind:

- **Read-only filesystem** — the filesystem tool cannot modify files
- **Project-scoped access** — only configured projects and paths are accessible
- **Sensitive files blocked** — `.ssh`, `.env`, private keys, and other sensitive files are denied by default
- **Dangerous commands blocked** — destructive commands like `rm -rf /` and `mkfs` are always denied
- **Minimal environment** — shell commands run with only basic environment variables (`PATH`, `HOME`, etc.)
- **No inbound connections** — the tunnel connects outward to Grafana; nothing is exposed on your machine

For fine-grained control over allowed paths, denied paths, allowed commands, and environment variables, see [Tunnel Configuration](CONFIGURATION.md#tunnel-configuration).

## Troubleshooting

### "Not authenticated for tunnel access"

Run `grafana-assistant auth`, then try connecting again.

### Tunnel connects but assistant can't see files

Make sure you've added at least one project:

```bash
grafana-assistant config add-project my-app ~/projects/my-app
```

The assistant can only access files in configured projects and explicitly allowed paths.

### Token expired

Tokens auto-refresh in the background. If refresh fails (e.g., the refresh token itself expired), re-authenticate:

```bash
grafana-assistant auth
```

### Daemon won't start

Check the logs for errors:

```bash
grafana-assistant tunnel daemon logs
```

Common issues:

- Config file not found — ensure `~/.config/grafana-assistant/config.yaml` exists
- Auth tokens expired — re-run `grafana-assistant auth`
- Another instance already running — check with `grafana-assistant tunnel daemon status`

## Next Steps

- [Docker Deployment](DOCKER.md) — run the tunnel in a container
- [Configuration Reference](CONFIGURATION.md#tunnel-configuration) — fine-tune filesystem and terminal tool settings
- [Setup & Authentication](SETUP.md) — configure instances and authentication
- [Chat & Prompt](CHAT.md) — talk to the assistant
