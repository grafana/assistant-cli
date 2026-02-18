# Setup & Authentication

This guide walks you through setting up the CLI and authenticating with your Grafana instance.

## Prerequisites

- A Grafana Cloud instance with the Assistant plugin enabled
- One of the following:
  - A **Grafana user account** (for browser-based authentication)
  - A **service account token** with the **Editor** role (for headless/CI use)

## Quick Setup

The fastest way to get started is to configure an instance and authenticate with the browser-based flow:

```bash
# 1. Add your Grafana instance
grafana-assistant config set-instance mystack \
  --url https://mystack.grafana.net

# 2. Set it as the default
grafana-assistant config use-instance mystack

# 3. Authenticate via browser
grafana-assistant auth
```

The `auth` command opens your browser, where you log in to Grafana and authorize the CLI. Once complete, the CLI stores short-lived tokens that auto-refresh — no service account needed.

## Authentication Methods

The CLI supports two authentication methods. You can use either one, depending on your use case.

### Browser-Based Authentication (Recommended)

Browser-based auth uses a secure PKCE flow to authenticate you as your Grafana user. Tokens are short-lived and automatically refresh.

```bash
grafana-assistant auth
```

This grants the CLI access to chat, prompt, agents, and tunnel commands — all in one step.

To authenticate a specific instance (if not the current one):

```bash
grafana-assistant auth --instance prod
```

**When to use:** Interactive use on your workstation. This is the recommended method for day-to-day usage.

### Service Account Token

Service account tokens are long-lived tokens created in the Grafana UI. They're useful for headless environments, CI/CD, and automation.

```bash
# Store the token in your config
grafana-assistant config set-instance mystack \
  --url https://mystack.grafana.net \
  --token '${GRAFANA_SA_TOKEN}'

# Or pass it directly
grafana-assistant prompt "Show recent alerts" \
  --url https://mystack.grafana.net \
  --token "$GRAFANA_SA_TOKEN"
```

> The service account must have the **Editor** role. See the [Prerequisites](../README.md#prerequisites) section in the README for setup instructions.

**When to use:** CI/CD pipelines, scripts, automation, or environments where a browser isn't available.

## Configuring Instances

Instances are named Grafana connections stored in your config file. This works similarly to kubeconfig — you can configure multiple instances and switch between them.

### Add an Instance

```bash
# Minimal — just a URL (use browser auth later)
grafana-assistant config set-instance dev --url http://localhost:3000

# With a service account token
grafana-assistant config set-instance prod \
  --url https://mystack.grafana.net \
  --token '${GRAFANA_PROD_TOKEN}'
```

### Switch Instances

```bash
grafana-assistant config use-instance prod
```

### List and Inspect

```bash
# List all instances
grafana-assistant config list

# Show the current instance
grafana-assistant config current

# Show the config file path
grafana-assistant config path
```

### Delete an Instance

```bash
grafana-assistant config delete-instance dev
```

## Credential Resolution

When you run a command, the CLI resolves credentials in this order (highest priority first):

1. `--url` and `--token` flags
2. `GRAFANA_URL` and `GRAFANA_SA_TOKEN` environment variables
3. `--instance` flag (selects a named instance from config)
4. `current-instance` from the config file

This means you can always override the config with explicit flags or environment variables, which is useful for one-off commands against a different instance:

```bash
# Use production for this one command only
grafana-assistant prompt "Show recent alerts" --instance prod
```

## Local vs Home Config

The CLI supports two config file locations:

| Location | Path | Use case |
|----------|------|----------|
| **Home** | `~/.config/grafana-assistant/config.yaml` | Personal settings, tokens |
| **Local** | `./grafana-assistant.yaml` | Project-specific, shared with team |

The local config takes priority over the home config. Use the `--local` flag with any `config` subcommand to target the local file:

```bash
# Add an instance to the local config
grafana-assistant config set-instance dev \
  --url http://localhost:3000 \
  --local
```

A common pattern is to commit a local config with instance URLs (but no tokens) to your repository, and have each team member set their tokens via environment variables:

```yaml
# grafana-assistant.yaml (committed to repo)
current-instance: dev
instances:
  dev:
    url: https://dev.grafana.example.com
    token: ${GRAFANA_DEV_TOKEN}
```

## Environment Variables

| Variable | Description |
|----------|-------------|
| `GRAFANA_URL` | Grafana instance URL (overrides config) |
| `GRAFANA_SA_TOKEN` | Service account token (overrides config) |
| `GRAFANA_ASSISTANT_CONFIG` | Override config file path entirely |

## Verifying Your Setup

After configuring and authenticating, verify everything works:

```bash
# Check your config
grafana-assistant config current

# List available agents (confirms connectivity)
grafana-assistant agents

# Start a chat
grafana-assistant chat
```

## Next Steps

- [Chat & Prompt](CHAT.md) — interactive chat and scripting
- [Tunnel](TUNNEL.md) — give the assistant access to your local files and tools
- [Configuration Reference](CONFIGURATION.md) — full config file format and options
