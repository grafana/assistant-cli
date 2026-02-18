# Running the Tunnel with Docker

Running the Grafana Assistant Tunnel as a Docker container is the simplest way to get a persistent, daemon-like tunnel without platform-specific service configuration.

## Benefits

- **No installation required** - Just `docker run`, no system service setup
- **Platform independent** - Works anywhere Docker runs (macOS, Linux, Windows, cloud VMs)
- **Sandboxing included** - Container isolation provides security by default
- **Pre-installed tools** - Comes with useful CLI tools (git, kubectl, helm, jq, etc.)
- **Easy updates** - Pull the new image and restart

## Prerequisites

1. **Docker** installed and running
2. **Authenticate** - this creates tokens in `~/.config/grafana-assistant/` that will be mounted into the container:
   ```bash
   docker run --rm -it \
     -p 54321:54321 \
     -v ~/.config/grafana-assistant:/home/tunnel/.config/grafana-assistant \
     --entrypoint grafana-assistant \
     grafana/assistant-cli:latest \
     auth --bind 0.0.0.0
   ```
   This opens a browser for authentication. The credentials (which include tunnel access) are saved to your host's config directory.

## Quick Start

```bash
# Basic usage - filesystem tool only
docker run -d \
  --name grafana-assistant-cli \
  --restart unless-stopped \
  -v ~/.config/grafana-assistant:/home/tunnel/.config/grafana-assistant:ro \
  -v ~/projects/my-app:/projects/my-app:ro \
  grafana/assistant-cli:latest
```

## Docker Compose (Recommended)

For easier management, use Docker Compose. Copy the example from the repository:

```bash
curl -o docker-compose.yaml https://raw.githubusercontent.com/grafana/assistant-cli/main/docker-compose.yaml
```

Edit the file to add your project paths, then:

```bash
# Start the tunnel
docker compose up -d

# View logs
docker compose logs -f

# Stop the tunnel
docker compose down

# Update to latest version
docker compose pull && docker compose up -d
```

## Configuration

### Mounting Projects

Projects must be explicitly mounted into the container. The assistant sees them by their mount name:

```bash
-v ~/projects/my-app:/projects/my-app:ro
-v ~/work/backend:/projects/backend:ro
```

With this configuration, the assistant can access files like `my-app/src/main.go` or `backend/cmd/server.go`.

> **Note:** Use `:ro` (read-only) mounts for safety unless you need the assistant to modify files.

### Enabling the Terminal Tool

By default, only the filesystem tool is enabled. To enable the terminal tool:

```bash
docker run -d \
  --name grafana-assistant-cli \
  -v ~/.config/grafana-assistant:/home/tunnel/.config/grafana-assistant:ro \
  -v ~/projects/my-app:/projects/my-app:ro \
  grafana/assistant-cli:latest \
  --filesystem --terminal
```

> **Caution:** The terminal tool allows the assistant to execute shell commands inside the container. Only enable it if needed.

### Git and SSH Access

If the assistant needs to run git commands (with terminal tool enabled):

```bash
-v ~/.gitconfig:/home/tunnel/.gitconfig:ro
-v ~/.ssh:/home/tunnel/.ssh:ro
```

### Kubernetes Access

For kubectl commands:

```bash
-v ~/.kube/config:/home/tunnel/.kube/config:ro
-e KUBECONFIG=/home/tunnel/.kube/config
```

### Environment Variables

Pass through environment variables for use by the terminal tool:

```bash
-e GITHUB_TOKEN
-e GITLAB_TOKEN
-e AWS_PROFILE
```

> **Note:** By default, the terminal tool only passes through `PATH`, `HOME`, `USER`, `SHELL`, `TERM`, `LANG`, and `LC_ALL` to executed commands. To pass additional environment variables like `GITHUB_TOKEN`, add them to your config file's `passthrough_env` list:
>
> ```yaml
> tunnel:
>   tools:
>     terminal:
>       passthrough_env:
>         - GITHUB_TOKEN
>         - GITLAB_TOKEN
>         - AWS_PROFILE
> ```

### Using a Specific Instance

If you have multiple Grafana instances configured:

```bash
grafana/assistant-cli:latest --filesystem --instance prod
```

### Resource Limits

Limit container resources:

```bash
docker run -d \
  --memory=512m \
  --cpus=0.5 \
  ...
```

## Pre-installed Tools

The container includes these tools for use with the terminal tool:

| Category | Tools |
|----------|-------|
| Version Control | git, git-lfs, openssh-client |
| Build | make, build-essential (gcc, g++, etc.) |
| Data Processing | jq, yq |
| Text Processing | grep, sed, gawk |
| File Utilities | findutils, tree, file |
| Kubernetes | kubectl, helm |
| Network | curl, wget, dnsutils, netcat |
| Shell | bash |
| Process | procps (ps, top, etc.) |

## Extending the Image

The image is based on `debian:bookworm-slim`, making it easy to extend with additional tools using `apt`.

### Adding Custom Tools

Create your own Dockerfile:

```dockerfile
FROM grafana/assistant-cli:latest

# Switch to root to install packages
USER root

# Install additional tools
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    nodejs \
    npm \
    postgresql-client \
    redis-tools \
    && rm -rf /var/lib/apt/lists/*

# Install pip packages
RUN pip3 install --break-system-packages awscli boto3

# Switch back to tunnel user
USER tunnel
```

Build and use your custom image:

```bash
docker build -t my-assistant-cli .
docker run -d \
  --name grafana-assistant-cli \
  -v ~/.config/grafana-assistant:/home/tunnel/.config/grafana-assistant:ro \
  -v ~/projects:/projects:ro \
  my-assistant-cli
```

### Common Extensions

**Build Tools (gcc, g++, etc.):**
```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends build-essential \
    && rm -rf /var/lib/apt/lists/*
```

**AWS CLI:**
```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends python3-pip \
    && pip3 install --break-system-packages awscli \
    && rm -rf /var/lib/apt/lists/*
```

**Node.js:**
```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends nodejs npm \
    && rm -rf /var/lib/apt/lists/*
```

**Terraform:**
```dockerfile
RUN curl -sSL https://releases.hashicorp.com/terraform/1.7.0/terraform_1.7.0_linux_amd64.zip -o /tmp/terraform.zip \
    && unzip /tmp/terraform.zip -d /usr/local/bin \
    && rm /tmp/terraform.zip
```

**Database Clients:**
```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    postgresql-client \
    default-mysql-client \
    redis-tools \
    && rm -rf /var/lib/apt/lists/*
```

## Managing the Container

```bash
# View logs
docker logs -f grafana-assistant-cli

# Check status
docker ps -f name=grafana-assistant-cli

# Stop the tunnel
docker stop grafana-assistant-cli

# Start again
docker start grafana-assistant-cli

# Remove completely
docker rm -f grafana-assistant-cli
```

## Troubleshooting

### "No instances are authenticated"

You need to authenticate first:

```bash
docker run --rm -it \
  -p 54321:54321 \
  -v ~/.config/grafana-assistant:/home/tunnel/.config/grafana-assistant \
  --entrypoint grafana-assistant \
  grafana/assistant-cli:latest \
  auth --bind 0.0.0.0
```

Then ensure the config directory is mounted correctly:

```bash
-v ~/.config/grafana-assistant:/home/tunnel/.config/grafana-assistant:ro
```

### Permission Denied on Mounted Volumes

The container runs as user `tunnel` (UID 1000). Ensure mounted files are readable:

```bash
# Check permissions
ls -la ~/.config/grafana-assistant/

# Fix if needed (make readable by others)
chmod 644 ~/.config/grafana-assistant/config.yaml
```

### Container Keeps Restarting

Check the logs for errors:

```bash
docker logs grafana-assistant-cli
```

Common issues:
- Config file not found or malformed
- Authentication token expired (re-run `grafana-assistant auth`)
- Network connectivity to Grafana

### Testing Connectivity

Run interactively to debug:

```bash
docker run -it --rm \
  -v ~/.config/grafana-assistant:/home/tunnel/.config/grafana-assistant:ro \
  grafana/assistant-cli:latest \
  --filesystem --verbose
```

## CI/CD Usage

The tunnel can run in CI/CD environments to give Grafana Assistant access to your codebase during builds.

### GitHub Actions

```yaml
services:
  tunnel:
    image: grafana/assistant-cli:latest
    volumes:
      - ${{ github.workspace }}:/projects/repo:ro
    env:
      GRAFANA_TUNNEL_TOKEN: ${{ secrets.GRAFANA_TUNNEL_TOKEN }}
```

### GitLab CI

```yaml
services:
  - name: grafana/assistant-cli:latest
    alias: tunnel
    variables:
      GRAFANA_TUNNEL_TOKEN: $GRAFANA_TUNNEL_TOKEN
```

> **Note:** CI/CD usage requires a tunnel token that can be passed via environment variable rather than config file. This feature may require additional setup.

## Security Considerations

1. **Mount config read-only** - The config contains authentication tokens
2. **Mount projects read-only** - Unless modification is explicitly needed
3. **Be cautious with terminal tool** - It can execute arbitrary commands
4. **Don't mount sensitive directories** - Avoid `~/.ssh` unless necessary
5. **Use resource limits** - Prevent runaway resource usage
