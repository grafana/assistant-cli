# Configuration

grafana-assistant supports a configuration file for managing multiple Grafana instances, similar to kubeconfig.

## Config File Locations

Configuration files are searched in the following order (first found wins):

1. **Environment override**: `GRAFANA_ASSISTANT_CONFIG` (if set)
2. **Local config**: `./grafana-assistant.yaml` (current directory)
3. **Home config**: `~/.config/grafana-assistant/config.yaml` (XDG conventions)

The local config is useful for project-specific or repository-specific configurations, similar to `.nvmrc` or `.python-version`.

## Config File Format

```yaml
# Currently selected instance
current-instance: prod

# Instance definitions
instances:
  localhost:
    url: http://localhost:3000
    token: glsa_abcd1234
  prod:
    url: https://mystack.grafana.net
    # Use environment variable for token (recommended for security)
    token: ${GRAFANA_PROD_TOKEN}
```

### Instance Fields

Each instance can have two types of credentials:

| Field | Set by | Description |
|-------|--------|-------------|
| `url` | `config set-instance` | Grafana instance URL |
| `token` | `config set-instance` | Service account token (supports `${ENV_VAR}` syntax) |
| `cli_token` | `auth` | Short-lived access token (`gat_...`) from browser login |
| `cli_token_expires_at` | `auth` | Expiration time of the CLI token (RFC 3339) |
| `cli_refresh_token` | `auth` | Refresh token (`gar_...`) for obtaining new access tokens |
| `cli_refresh_expires_at` | `auth` | Expiration time of the refresh token (RFC 3339) |
| `api_endpoint` | `auth` | Direct API endpoint URL (bypasses Grafana plugin proxy) |

You don't need to set all of these. A typical instance needs either:
- **Browser auth** (recommended): just `url`, then run `grafana-assistant auth` to populate the `cli_*` and `api_endpoint` fields automatically.
- **Service account**: `url` and `token` — no `auth` step needed.

After running `grafana-assistant auth`, your config will look like this:

```yaml
instances:
  mystack:
    url: https://mystack.grafana.net
    # These fields are managed automatically — don't edit them by hand
    cli_token: gat_xxxxxxxxxxxx
    cli_token_expires_at: "2025-03-17T11:30:00Z"
    cli_refresh_token: gar_xxxxxxxxxxxx
    cli_refresh_expires_at: "2025-03-24T11:30:00Z"
    api_endpoint: https://assistant-api.grafana.net
```

CLI tokens auto-refresh in the background. If the refresh token itself expires, re-run `grafana-assistant auth`.

## Managing Instances

### Add an Instance and Authenticate

The recommended flow is to add an instance with just a URL, then authenticate via the browser:

```bash
# Add the instance
grafana-assistant config set-instance mystack \
  --url https://mystack.grafana.net

# Authenticate (opens browser)
grafana-assistant auth
```

### Add an Instance with a Service Account Token

For headless environments or CI/CD, use a service account token instead:

```bash
# Add with literal token
grafana-assistant config set-instance prod \
  --url https://mystack.grafana.net \
  --token glsa_abcd1234

# Add with token from environment variable (recommended)
grafana-assistant config set-instance prod \
  --url https://mystack.grafana.net \
  --token '${GRAFANA_PROD_TOKEN}'
```

> Grafana service account token must have the **Editor** role. See the [Prerequisites](../README.md#prerequisites) section in the README for setup instructions.

### Set the Current Instance

```bash
grafana-assistant config use-instance prod
```

### List All Instances

```bash
grafana-assistant config list
```

Example output:
```
CURRENT  NAME       URL                          AUTH                API ENDPOINT
         localhost  http://localhost:3000         Not authenticated   -
*        prod       https://mystack.grafana.net   Authenticated       https://assistant-api.grafana.net
```

### Show Current Instance

```bash
grafana-assistant config current
```

### Delete an Instance

```bash
grafana-assistant config delete-instance dev
```

## Credential Resolution Priority

Credentials are resolved in the following order (highest priority first):

1. `--url` and `--token` flags (explicit)
2. `GRAFANA_URL` and `GRAFANA_SA_TOKEN` environment variables
3. `--instance <name>` flag (select from config)
4. `current-instance` from config file

When resolving from config (steps 3 and 4), the CLI prefers browser-auth credentials over service account tokens:

- If the instance has `cli_token` + `api_endpoint` (set by `grafana-assistant auth`), those are used for direct API access with automatic token refresh.
- Otherwise, the `token` field (service account token) is used via the Grafana plugin proxy.

This means you can always override the config file by passing explicit flags or setting environment variables. Explicit `--token` or `GRAFANA_SA_TOKEN` always takes precedence over CLI auth tokens.

## Environment Variables

| Variable | Description |
|----------|-------------|
| `GRAFANA_URL` | Grafana instance URL (overrides config) |
| `GRAFANA_SA_TOKEN` | Service account token (overrides config) |
| `GRAFANA_ASSISTANT_CONFIG` | Override config file path |

## Security

### Token Storage

**Browser auth** (`grafana-assistant auth`) stores short-lived tokens directly in the config file. These tokens auto-refresh and expire quickly, limiting exposure if the file is compromised.

**Service account tokens** are long-lived. For these, we recommend storing them as environment variable references rather than literal values:

```yaml
instances:
  prod:
    url: https://mystack.grafana.net
    token: ${GRAFANA_PROD_TOKEN}
```

Then set the environment variable in your shell profile or environment:

```bash
export GRAFANA_PROD_TOKEN="glsa_your_actual_token"
```

This way, long-lived tokens are never written to disk in plain text.

### File Permissions

The config file is automatically created with `0600` permissions (owner read/write only). If permissions are too permissive, you'll see a warning:

```
⚠️  config file /home/user/.config/grafana-assistant/config.yaml has insecure permissions 644; consider running 'chmod 600 /home/user/.config/grafana-assistant/config.yaml'
```

To fix this:

```bash
chmod 600 ~/.config/grafana-assistant/config.yaml
```

## Examples

### Project-Specific Config

Create a `grafana-assistant.yaml` in your project root to share instance configuration with your team:

```yaml
# grafana-assistant.yaml (commit this to your repo)
current-instance: dev

instances:
  dev:
    url: https://dev.grafana.example.com
    token: ${GRAFANA_DEV_TOKEN}
  staging:
    url: https://staging.grafana.example.com
    token: ${GRAFANA_STAGING_TOKEN}
```

Team members set the tokens in their environment:

```bash
export GRAFANA_DEV_TOKEN="glsa_your_dev_token"
export GRAFANA_STAGING_TOKEN="glsa_your_staging_token"
```

### Multi-Environment Setup

A typical home config with local development and production instances:

```yaml
current-instance: localhost

instances:
  localhost:
    url: http://localhost:3000
    token: ${GRAFANA_LOCAL_TOKEN}
  staging:
    url: https://staging.grafana.example.com
    token: ${GRAFANA_STAGING_TOKEN}
  prod:
    url: https://grafana.example.com
    token: ${GRAFANA_PROD_TOKEN}
```

Then in your `.bashrc` or `.zshrc`:

```bash
export GRAFANA_LOCAL_TOKEN="glsa_local_dev_token"
export GRAFANA_STAGING_TOKEN="glsa_staging_token"
export GRAFANA_PROD_TOKEN="glsa_prod_token"
```

### Quick Instance Switching

```bash
# Work with localhost
grafana-assistant config use-instance localhost
grafana-assistant prompt "Show me recent alerts"

# Switch to production
grafana-assistant config use-instance prod
grafana-assistant prompt "Show me recent alerts"

# One-off command against a different instance
grafana-assistant prompt "Show me recent alerts" --instance staging
```

## Projects Configuration

Projects are named directories that the Grafana Assistant can access via the tunnel. Configure them at the top level of your config file.

### Adding Projects via CLI

The easiest way to manage projects is via the CLI:

```bash
# Add a project
grafana-assistant config add-project my-app ~/projects/my-app

# List projects
grafana-assistant config list-projects

# Remove a project
grafana-assistant config remove-project my-app
```

### Adding Projects via Grafana Assistant

If you have the Tunnel connected to the Grafana Assistant, you can just ask the Assistant to add a project with a message like:

> Add the 'my-app' project to your config. It's at ~/projects/my-app

And the Assistant will take care of it for you.

### Projects Config Structure

```yaml
projects:
  - name: my-app
    path: ~/projects/my-app
  - name: infrastructure
    path: ~/work/infra
```

The `name` identifies the project when using filesystem tools.

## Tunnel Configuration

The tunnel allows Grafana Assistant to execute tools on your machine. Authentication is handled by `grafana-assistant auth` (which includes the `tunnel:connect` scope). The settings below control which tools the tunnel exposes and how they behave.

### Tunnel Tools Config Structure

```yaml
tunnel:
  tools:
    filesystem:
      # Optional: additional paths outside projects (e.g., logs)
      allowed_paths:
        - /var/log/myapp
      deny_paths:
        - "**/*.key"
        - "**/secrets.yaml"
    terminal:
      allowed_commands:
        - git
        - kubectl
        - docker
      deny_commands:
        - rm -rf
      passthrough_env:
        - AWS_PROFILE
        - KUBECONFIG
        - DOCKER_HOST
```

### Filesystem Tool

The filesystem tool provides read-only access to local files and directories. It uses a **project-based** model where you configure named projects that the assistant can access.

#### Projects

Projects are configured at the top level of the config file (see [Projects Configuration](#projects-configuration) above).

The assistant accesses files using project-relative paths like `my-app/src/main.go` instead of absolute paths. This makes it easier for the assistant to understand your codebase structure.

**Dynamic project management**: Projects can be added or removed mid-session using the `add_project` and `remove_project` tools, or via the CLI with `grafana-assistant config add-project`. Changes are persisted to your config file and take effect immediately.

#### Available Actions

| Action | Description |
|--------|-------------|
| `list_projects` | List all configured projects |
| `add_project` | Add a new project (persists to config) |
| `remove_project` | Remove a project from config |
| `read_file` | Read file contents with optional line range |
| `list_directory` | List directory contents |
| `grep` | Search file contents with regex (paginated) |
| `find_path` | Find files by glob pattern (paginated) |

#### Configuration Options

| Option | Description | Default |
|--------|-------------|---------|
| `projects` | Named project directories | None |
| `allowed_paths` | Additional paths outside projects | None |
| `deny_paths` | Paths that are never accessible | See below |

**Default denied paths** (always enforced):
- `~/.ssh` - SSH keys
- `~/.gnupg` - GPG keys
- `~/.aws/credentials` - AWS credentials
- `**/.env` - Environment files
- `**/secrets.yaml` - Secret files
- `**/*.pem`, `**/*.key` - Private keys

**File size limit**: 1MB (prevents accidentally reading huge files)

### Terminal Tool

The terminal tool executes shell commands on your local machine.

| Option | Description | Default |
|--------|-------------|---------|
| `allowed_commands` | Commands that are permitted | All (if empty) |
| `deny_commands` | Commands that are blocked | See below |
| `passthrough_env` | Extra env vars to pass to commands | None |

**Default denied commands** (always enforced):
- `rm -rf /`, `rm -rf /*` - Destructive deletion
- `mkfs` - Filesystem formatting
- `dd if=/dev/zero`, `dd if=/dev/random` - Disk overwrites
- `:(){:|:&};:` - Fork bomb
- `chmod -R 777 /`, `chown -R` - Mass permission changes

**Default passthrough environment variables**:
- `PATH`, `HOME`, `USER` - Basic shell operation
- `SHELL`, `TERM` - Terminal configuration
- `LANG`, `LC_ALL` - Locale settings

Commands run with a minimal environment by default. Only the variables listed above (plus any you add to `passthrough_env`) are passed to executed commands. This prevents accidental leakage of sensitive environment variables like API tokens.

### Example: Kubernetes Development Setup

```yaml
projects:
  - name: my-app
    path: ~/projects/my-app

tunnel:
  tools:
    filesystem:
      allowed_paths:
        - ~/.kube
    terminal:
      allowed_commands:
        - kubectl
        - helm
        - git
      passthrough_env:
        - KUBECONFIG
        - HELM_HOME
```

### Example: Restrictive Production Setup

```yaml
# No projects configured - only allow reading logs

tunnel:
  tools:
    filesystem:
      allowed_paths:
        - /var/log/myapp
      deny_paths:
        - "**/*.log.gz"  # Don't read archived logs
    terminal:
      allowed_commands:
        - tail
        - grep
        - cat
      # No extra env vars - use minimal defaults only
```

### Example: Multi-Project Development

```yaml
projects:
  - name: frontend
    path: ~/projects/web-app
  - name: backend
    path: ~/projects/api-server
  - name: shared
    path: ~/projects/shared-libs

tunnel:
  tools:
    filesystem:
      deny_paths:
        - "**/node_modules/**"  # Skip dependency folders
        - "**/vendor/**"
```

With this configuration, the assistant can access files using:
- `read_file` with `project: "frontend"` and `path: "src/App.tsx"`
- `read_file` with `project: "backend"` and `path: "cmd/server/main.go"`
- `grep` with `project: "shared"` to search across the shared libraries
