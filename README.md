# grafana-assistant

> **Warning: Experimental - Private Preview**
>
> This package is currently in **private preview** and is considered **experimental**. APIs and features may change without notice. Use at your own risk and do not rely on it for production workloads.

CLI for interacting with Grafana Assistant via the A2A (Agent-to-Agent) API.

## Installation

### Download from GitHub Releases (Recommended)

Download the latest pre-built binary for your platform from [GitHub Releases](https://github.com/grafana/assistant-cli/releases/latest).

#### macOS / Linux

Using the [GitHub CLI](https://cli.github.com/):

```bash
# Download the latest release asset matching your platform
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m | sed -e 's/x86_64/amd64/' -e 's/aarch64/arm64/')

gh release download --repo grafana/assistant-cli --pattern "grafana-assistant_*_${OS}_${ARCH}.tar.gz" -D /tmp
tar -xzf /tmp/grafana-assistant_*_${OS}_${ARCH}.tar.gz -C /tmp
sudo mv /tmp/grafana-assistant /usr/local/bin/

# Verify
grafana-assistant --help
```

Or download the archive for your platform directly from the [latest release](https://github.com/grafana/assistant-cli/releases/latest) page, extract it, and move the `grafana-assistant` binary to a directory in your `PATH`.

#### Windows

Download the `.zip` archive for your platform from the [latest release](https://github.com/grafana/assistant-cli/releases/latest), extract it, and add the `grafana-assistant.exe` binary to your `PATH`.

### Install from GitHub Releases using `go`

```bash
go install github.com/grafana/assistant-cli/cmd/grafana-assistant@latest
```

## Quick Start

```bash
# Add your Grafana instance
grafana-assistant config set-instance mystack \
  --url https://mystack.grafana.net

# Set it as the default
grafana-assistant config use-instance mystack

# Authenticate via browser
grafana-assistant auth

# Start interactive chat
grafana-assistant chat
```

The `auth` command opens your browser for a one-time login. After that, the CLI manages short-lived tokens that auto-refresh — no service account needed.

> [!IMPORTANT]
> The **Assistant CLI User** role is required to obtain access tokens via `grafana-assistant auth`. This role is automatically granted to users with the **Editor** role or above. For custom roles, the `grafana-assistant-app.tokens:access` permission must be included.

For non-interactive environments (CI/CD, scripts), you can use a service account token instead:

```bash
grafana-assistant config set-instance mystack \
  --url https://mystack.grafana.net \
  --token '${GRAFANA_SA_TOKEN}'
```

Configuration is stored in `~/.config/grafana-assistant/config.yaml` by default, or `./grafana-assistant.yaml` for project-specific configs. Session state (last chat ID for `--continue`) is stored in `~/.config/grafana-assistant/state.yaml`.

For detailed setup instructions, see [docs/SETUP.md](docs/SETUP.md). For the full config file reference, see [docs/CONFIGURATION.md](docs/CONFIGURATION.md).

## Usage

> For detailed usage guides, see [Setup & Auth](docs/SETUP.md), [Chat & Prompt](docs/CHAT.md), and [Tunnel](docs/TUNNEL.md).

### Interactive Chat (Recommended)

Start a full-screen interactive chat session with real-time streaming responses and conversation continuation.

```bash
# Start interactive chat
grafana-assistant chat

# With a specific agent
grafana-assistant chat --agent investigation_agent

# Resume a previous conversation by context ID
grafana-assistant chat --context "previous-context-id"

# Continue the last chat session
grafana-assistant chat --continue

# Using a specific instance
grafana-assistant chat --instance prod
```

**Commands:**

- `/clear` or `/new` - Start a new conversation
- `/exit`, `/quit`, or `/q` - Exit
- `/help` - Show help

**Keyboard Shortcuts:**

- `Enter` - Send message
- `Ctrl+C` - Cancel current request or exit
- `Ctrl+D` - Exit

### Single-Shot Prompt (for Scripting)

Send a single message and receive the response. Useful for automation and scripting.

```bash
# Send a single prompt
grafana-assistant prompt "Analyze my dashboard"

# Using a specific instance from config
grafana-assistant prompt "Analyze my dashboard" --instance prod

# Using explicit URL and token
grafana-assistant prompt "Analyze my dashboard" \
  --url https://your-stack.grafana.net \
  --token $GRAFANA_SA_TOKEN

# Set a custom timeout (default: 300 seconds)
grafana-assistant prompt "Query error rates from Prometheus" --timeout 600

# Continue a conversation with context ID
grafana-assistant prompt "Now show me the last 24 hours" --context "previous-context-id"

# Output as JSON (for scripting)
grafana-assistant prompt "Analyze my dashboard" --json
```

## CLI Reference

### Chat Command (Interactive)

| Option           | Description                              | Default                 |
| ---------------- | ---------------------------------------- | ----------------------- |
| `--url, -u`      | Grafana instance URL                     | -                       |
| `--token, -t`    | Service account token                    | -                       |
| `--instance, -i` | Instance name from config                | -                       |
| `--context, -c`  | Context ID for conversation continuation | -                       |
| `--continue`     | Continue the previous chat session       | `false`                 |
| `--timeout`      | Timeout in seconds per request           | `300`                   |

### Prompt Command (Non-Interactive)

| Option           | Description                           | Default                 |
| ---------------- | ------------------------------------- | ----------------------- |
| `--url, -u`      | Grafana instance URL                  | -                       |
| `--token, -t`    | Service account token                 | -                       |
| `--instance, -i` | Instance name from config             | -                       |
| `--wait, -w`     | Wait for completion                   | `true`                  |
| `--timeout`      | Timeout in seconds                    | `300`                   |
| `--context, -c`  | Context ID for conversation threading | auto-generated          |
| `--json`         | Output as JSON                        | `false`                 |

### Auth Command

| Command                  | Description                                    |
| ------------------------ | ---------------------------------------------- |
| `auth`                   | Authenticate via browser (recommended)         |
| `auth --instance <name>` | Authenticate a specific instance               |

### Config Commands

| Command                            | Description                   |
| ---------------------------------- | ----------------------------- |
| `config set-instance <name>`       | Add or update an instance     |
| `config use-instance <name>`       | Set the current instance      |
| `config list`                      | List all configured instances |
| `config current`                   | Show the current instance     |
| `config delete-instance <name>`    | Remove an instance            |
| `config add-project <name> <path>` | Add a project directory       |
| `config list-projects`             | List all configured projects  |
| `config remove-project <name>`     | Remove a project              |

### Tunnel Command

Run a tunnel that allows Grafana Assistant to execute tools on your machine.

```bash
# Authenticate (includes tunnel access)
grafana-assistant auth

# Add a project for the assistant to access
grafana-assistant config add-project my-app ~/projects/my-app

# Start tunnel with filesystem access (default)
grafana-assistant tunnel connect

# Start tunnel with terminal access (use with caution)
grafana-assistant tunnel connect --terminal
```

The `auth` command includes the `tunnel:connect` scope, so no separate tunnel auth step is needed.

The tunnel provides:
- **Filesystem tool**: Read-only access to local files with project-based configuration
  - Configure named projects (e.g., `my-app` → `~/projects/my-app`)
  - Access files with project-relative paths (e.g., `src/main.go`)
  - Search code with `grep` and `find_path` actions
  - Add/remove projects dynamically mid-session
- **Terminal tool**: Execute shell commands (with configurable allow/deny lists)

For security, the tunnel:
- Blocks access to sensitive files (`.ssh`, `.env`, private keys, etc.) by default
- Blocks dangerous commands (`rm -rf /`, `mkfs`, fork bombs, etc.) by default
- Runs commands with a minimal environment (only `PATH`, `HOME`, `USER`, etc.)
- Requires explicit project configuration before accessing codebases

See [docs/TUNNEL.md](docs/TUNNEL.md) for the full tunnel usage guide and [docs/CONFIGURATION.md](docs/CONFIGURATION.md#tunnel-configuration) for tunnel configuration options.

## Environment Variables

| Variable                   | Description               |
| -------------------------- | ------------------------- |
| `GRAFANA_URL`              | Grafana instance URL      |
| `GRAFANA_SA_TOKEN`         | Service account token     |
| `GRAFANA_ASSISTANT_CONFIG` | Override config file path |

## JSON Output

When using `--json` with the `prompt` command, the output format is:

### Prompt Response

```json
{
  "taskId": "a2a-task-123",
  "contextId": "uuid-for-threading",
  "status": "completed",
  "response": "The agent's response text..."
}
```

Possible status values: `completed`, `failed`, `timeout`, `canceled`, `unknown`

## Tool Approval Flow

When the Grafana Assistant needs to execute certain tools that require user confirmation (such as running queries or modifying resources), the CLI will prompt you for approval.

### How It Works

1. During a chat session, when a tool requires approval, you'll see a prompt:

```
Approve execute_query - Execute a database query? [y]/[n]
```

2. Press `y` or `Y` to approve and allow the tool to execute
3. Press `n`, `N`, or `Esc` to deny and skip the tool execution

### Approval Keys

| Key | Action |
|-----|--------|
| `y` or `Y` | Approve the tool execution |
| `n` or `N` | Deny the tool execution |
| `Esc` | Deny the tool execution |

When you deny a tool, the assistant will be informed and will continue the conversation without executing that tool.

## A2A Protocol

This CLI uses the A2A (Agent-to-Agent) Protocol v0.3.0 for communication:

- **Streaming**: Real-time SSE streaming of agent responses
- **Context threading**: Continue conversations with context IDs
- **Standard protocol**: JSON-RPC 2.0 over HTTP with SSE

For more details, see the [A2A Protocol Specification](https://a2a-protocol.org/latest/specification/).

## Prerequisites

1. A **Grafana instance** with the Assistant plugin enabled
2. A **Grafana user account** for browser-based authentication (recommended)

For non-interactive use (CI/CD, automation), create a **service account** with the **Editor** role and generate a token instead.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for development setup, testing, and snapshot test instructions.

## License

This software is licensed under the [Grafana Enterprise Plugin License Agreement](LICENSE).
