# Chat & Prompt

The CLI provides two ways to talk to Grafana Assistant: **interactive chat** for conversational use, and **prompt** for single-shot messages and scripting.

## Interactive Chat

The `chat` command opens a full-screen terminal UI with real-time streaming responses, markdown rendering, and conversation continuation.

```bash
grafana-assistant chat
```

### Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `Enter` | Send message |
| `Ctrl+C` | Cancel current request, or exit if idle |
| `Ctrl+D` | Exit |

### Chat Commands

Type these in the input area:

| Command | Action |
|---------|--------|
| `/clear` or `/new` | Start a new conversation |
| `/exit`, `/quit`, `/q` | Exit the chat |
| `/help` | Show help |

### Choosing an Agent

Agents are specialized assistants for different tasks. By default, the CLI uses the `grafana_assistant_cli` agent, but you can select a different one:

```bash
# Use a specific agent
grafana-assistant chat --agent investigation_agent

# List available agents first
grafana-assistant agents
```

### Resuming a Conversation

Every chat session has a context ID. You can resume a previous session in two ways:

```bash
# Continue the last session
grafana-assistant chat --continue

# Resume a specific session by context ID
grafana-assistant chat --context "abc123-def456"
```

### Tool Approval

When the assistant needs to execute a tool (like running a query), you'll be prompted to approve it:

```
Approve execute_query - Execute a database query? [y]/[n]
```

Press `y` to approve or `n` / `Esc` to deny. The assistant will continue the conversation either way.

### Options

| Flag | Description | Default |
|------|-------------|---------|
| `--url, -u` | Grafana instance URL | from config |
| `--token, -t` | Service account token | from config |
| `--instance, -i` | Instance name from config | current instance |
| `--agent, -a` | Agent ID | `grafana_assistant_cli` |
| `--context, -c` | Context ID to resume | new session |
| `--continue` | Continue the last session | `false` |
| `--timeout` | Timeout per request (seconds) | `300` |

## Single-Shot Prompt

The `prompt` command sends a single message and prints the response. It's designed for scripting, automation, and quick one-off questions.

```bash
grafana-assistant prompt "What alerts fired in the last hour?"
```

### Conversation Threading

Like chat, prompts support context IDs for multi-turn conversations:

```bash
# First message — the CLI prints the context ID
grafana-assistant prompt "Show me the error rate for the payments service"

# Follow up using the context ID from the previous response
grafana-assistant prompt "Now show me the last 24 hours" \
  --context "abc123-def456"
```

### JSON Output

Use `--json` for machine-readable output, useful for integrating with other tools:

```bash
grafana-assistant prompt "Analyze my dashboard" --json
```

Output format:

```json
{
  "taskId": "a2a-task-123",
  "contextId": "uuid-for-threading",
  "agentId": "grafana_assistant_cli",
  "status": "completed",
  "response": "The agent's response text..."
}
```

Possible `status` values: `completed`, `failed`, `timeout`, `canceled`, `unknown`.

On failure, an `error` field is included:

```json
{
  "status": "failed",
  "error": "Error message describing what went wrong"
}
```

### Options

| Flag | Description | Default |
|------|-------------|---------|
| `--url, -u` | Grafana instance URL | from config |
| `--token, -t` | Service account token | from config |
| `--instance, -i` | Instance name from config | current instance |
| `--agent, -a` | Agent ID | `grafana_assistant_cli` |
| `--wait, -w` | Wait for completion | `true` |
| `--timeout` | Timeout in seconds | `300` |
| `--context, -c` | Context ID for threading | auto-generated |
| `--json` | Output as JSON | `false` |

## Listing Agents

The `agents` command shows which agents are available on your Grafana instance:

```bash
grafana-assistant agents
```

Example output:

```
Available Agents:
-----------------

  Grafana Assistant (Web)
    ID:          grafana_assistant_web
    Name:        Grafana Assistant (Web)
    Description: General-purpose assistant for Grafana web interface
    Skills:      5
```

Use `--json` for machine-readable output:

```bash
grafana-assistant agents --json
```

## Examples

### Quick Question

```bash
grafana-assistant prompt "What's the current state of my Kubernetes cluster?"
```

### Scripting with JSON

```bash
# Get alert summary as JSON, extract the response with jq
response=$(grafana-assistant prompt "Summarize active alerts" --json | jq -r '.response')
echo "$response"
```

### Using in a Shell Script

```bash
#!/bin/bash
# Check for critical alerts and notify
result=$(grafana-assistant prompt "Are there any critical alerts firing right now?" --json)
status=$(echo "$result" | jq -r '.status')

if [ "$status" = "completed" ]; then
  response=$(echo "$result" | jq -r '.response')
  echo "Assistant says: $response"
else
  echo "Failed to get response: $(echo "$result" | jq -r '.error')"
  exit 1
fi
```

### One-Off Against a Different Instance

```bash
# Query staging without changing your default
grafana-assistant prompt "Show error rates" --instance staging
```

## Next Steps

- [Tunnel](TUNNEL.md) — give the assistant access to your local files and tools
- [Setup & Authentication](SETUP.md) — configure instances and authentication
- [Configuration Reference](CONFIGURATION.md) — full config file format and options
