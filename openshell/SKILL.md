---
name: openshell
description: Manage the full OpenShell sandbox lifecycle — initial setup, gateway management, provider configuration, sandbox creation/connection/deletion, and policy iteration. Works across macOS/Linux, Podman/Docker, and any LLM provider (Vertex AI, Anthropic API, OpenAI, etc.). Trigger keywords - openshell, sandbox, sandbox create, sandbox connect, gateway, provider, policy, openshell setup, start sandbox, launch sandbox, create sandbox, connect sandbox, delete sandbox, sandbox logs, policy denied, token refresh, openshell troubleshoot.
---

# OpenShell Sandbox Manager

Manage the full OpenShell sandbox lifecycle: initial setup, gateway management, provider configuration, sandbox creation/connection/deletion, and policy iteration.

## Overview

OpenShell is NVIDIA's secure, private runtime for autonomous AI agents. It provides isolated sandboxes governed by declarative YAML policies that control filesystem access, network traffic, and process identity. This skill automates the entire lifecycle — from first install to daily sandbox management and policy refinement.

**Companion reference**: See [cli-reference.md](cli-reference.md) for the full command tree. The CLI also has built-in help at every level: `openshell <group> <cmd> --help`.

## Step 1: Detect Environment

Before doing anything, detect the current environment. Run these checks and store the results for use throughout the skill:

```bash
# Platform
OS=$(uname -s)  # Darwin or Linux

# Container runtime (prefer podman, fall back to docker)
if command -v podman &>/dev/null; then
    RUNTIME=podman
elif command -v docker &>/dev/null; then
    RUNTIME=docker
else
    RUNTIME=none
fi

# OpenShell installed?
if command -v openshell &>/dev/null; then
    OPENSHELL_VERSION=$(openshell --version 2>/dev/null || echo "unknown")
    OPENSHELL_INSTALLED=true
else
    OPENSHELL_INSTALLED=false
fi

# Gateway reachable?
if [ "$OPENSHELL_INSTALLED" = true ]; then
    if openshell status &>/dev/null; then
        GATEWAY_UP=true
    else
        GATEWAY_UP=false
    fi
else
    GATEWAY_UP=false
fi
```

Report the detected environment to the user in a short summary, then route to the appropriate workflow.

## Step 2: Route to Workflow

Use the detected environment and user intent to pick the right workflow:

| User intent | Gateway reachable? | Route to |
|---|---|---|
| "set up openshell" / first time | Any | **Initial Setup** |
| "create sandbox" / "launch sandbox" | No | **Initial Setup** first, then **Sandbox Lifecycle** |
| "create sandbox" / "launch sandbox" | Yes | **Sandbox Lifecycle** |
| "connect to sandbox" | Yes | **Sandbox Lifecycle** (connect) |
| "start gateway" / "gateway not working" | Any | **Gateway Management** |
| "add provider" / "refresh token" | Yes | **Provider Management** |
| "create policy" / "policy denied" / "update policy" | Yes | **Policy Management** |
| Ambiguous | Any | Ask the user what they want to do |

If the gateway is down and the user wants to create a sandbox, run gateway startup first before proceeding.

---

## Workflow 1: Initial Setup

Run this when OpenShell isn't installed or no gateway exists.

### 1.1 Install OpenShell

Skip if `openshell --version` succeeds.

**macOS:**
```bash
brew install openshell
```

**Linux:**
```bash
curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
```

Confirm with the user before running the install command.

### 1.2 Ensure Container Runtime

Skip if `podman` or `docker` is available.

If neither is found, tell the user to install one and stop. Podman is recommended.

**macOS with Podman** — ensure the machine is running:
```bash
if ! podman machine inspect &>/dev/null; then
    podman machine init
fi
if ! podman machine list --format '{{.Running}}' | grep -q true; then
    podman machine start
fi
```

### 1.3 Generate TLS Certificates

Determine paths based on platform and runtime:

```bash
CONFIGDIR="$HOME/.config/openshell"
STATEDIR="$HOME/.local/state/openshell"
mkdir -p "$STATEDIR"
```

Determine the container socket path:

| Platform + Runtime | Socket path |
|---|---|
| macOS + Podman | `$(podman machine inspect --format '{{.ConnectionInfo.PodmanSocket.Path}}')` |
| Linux + Podman | `$XDG_RUNTIME_DIR/podman/podman.sock` |
| Linux + Docker | `/var/run/docker.sock` |
| macOS + Docker | `/var/run/docker.sock` |

Generate certificates:
```bash
$RUNTIME run --rm \
    -e HOME=$HOME \
    -v $CONFIGDIR:$CONFIGDIR \
    -v $STATEDIR:$STATEDIR \
    ghcr.io/nvidia/openshell/gateway:latest \
    generate-certs --output-dir $STATEDIR/tls \
    --server-san host.openshell.internal
```

Confirm with the user before running.

### 1.4 Start the Gateway

Build the gateway run command based on detected platform and runtime:

**Base command (all platforms):**
```bash
$RUNTIME run -d --name openshell-gateway --restart always \
    -p 0.0.0.0:17670:17670 \
    -v $CONFIGDIR:$CONFIGDIR \
    -v $STATEDIR:$STATEDIR \
    -e HOME=$HOME \
    -e OPENSHELL_DRIVERS=$RUNTIME \
    ghcr.io/nvidia/openshell/gateway:latest \
    --bind-address 0.0.0.0 --port 17670
```

**Platform-specific additions:**

| Platform + Runtime | Additional flags |
|---|---|
| macOS + Podman | `-v $(podman machine inspect --format '{{.ConnectionInfo.PodmanSocket.Path}}'):/var/run/podman.sock -e OPENSHELL_PODMAN_SOCKET=/var/run/podman.sock` |
| Linux + Podman | `-v $XDG_RUNTIME_DIR/podman/podman.sock:/var/run/podman.sock -e OPENSHELL_PODMAN_SOCKET=/var/run/podman.sock --security-opt label=type:container_runtime_t` |
| Linux + Docker | `-v /var/run/docker.sock:/var/run/docker.sock -e OPENSHELL_DOCKER_SOCKET=/var/run/docker.sock` |
| macOS + Docker | `-v /var/run/docker.sock:/var/run/docker.sock -e OPENSHELL_DOCKER_SOCKET=/var/run/docker.sock` |

Confirm with the user before running.

### 1.5 Register the Gateway

```bash
openshell gateway add https://localhost:17670 --local
```

Verify:
```bash
openshell status
```

If the gateway isn't responding, wait up to 20 seconds with retries:
```bash
for i in $(seq 1 10); do
    openshell status &>/dev/null && break
    sleep 2
done
```

### 1.6 Prompt for Provider Setup

Ask: "Gateway is up. Do you want to set up an LLM provider now?"

If yes, route to **Provider Management**.

---

## Workflow 2: Gateway Management

For starting, stopping, troubleshooting, and switching gateways.

### Start a stopped gateway

```bash
$RUNTIME start openshell-gateway
```

Verify with `openshell status`.

### Restart the gateway

```bash
$RUNTIME restart openshell-gateway
```

### View gateway logs

```bash
$RUNTIME logs openshell-gateway --tail 50
```

### Check gateway health

```bash
openshell status
```

### Switch between gateways

```bash
openshell gateway select          # List all, show active
openshell gateway select <name>   # Switch active
```

### Remove a gateway registration

```bash
openshell gateway remove <name>
```

### Common troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `openshell status` fails with connection refused | Gateway container not running | `$RUNTIME start openshell-gateway` |
| Gateway starts then exits immediately | Socket path wrong or runtime not running | Check `$RUNTIME logs openshell-gateway`, verify socket exists |
| TLS handshake error | Cert mismatch or expired certs | Regenerate certs (Step 1.3), restart gateway |
| macOS: gateway can't reach Podman socket | Podman machine not running | `podman machine start` |

---

## Workflow 3: Sandbox Lifecycle

### Create a sandbox

**Minimal (interactive shell):**
```bash
openshell sandbox create
```

**With a specific agent:**
```bash
openshell sandbox create -- claude
openshell sandbox create -- codex
```

**With options:**
```bash
openshell sandbox create \
    --name my-sandbox \
    --provider <provider-name> \
    --policy ./my-policy.yaml \
    --upload <local-path>:<sandbox-path> \
    -- <command>
```

Key flags:
- `--provider <NAME>` — attach a provider (repeatable for multiple providers)
- `--policy <PATH>` — custom policy YAML
- `--upload <PATH>[:<DEST>]` — upload local files (default dest: `/sandbox`)
- `--no-keep` — delete sandbox after initial command exits
- `--forward <PORT>` — forward a local port to the sandbox
- `--name <NAME>` — name the sandbox (auto-generated if omitted)
- `--from <SOURCE>` — use a Dockerfile, image reference, or community sandbox name
- `--cpu <QTY>` — CPU limit (e.g., `2`)
- `--memory <QTY>` — memory limit (e.g., `4Gi`)

### Provider-specific sandbox creation

When the user wants to run Claude Code via Vertex AI, the sandbox needs specific env vars. Build the create command with the appropriate uploads and inline env setup:

```bash
openshell sandbox create \
    --provider <vertex-provider-name> \
    --upload "$HOME/.config/gcloud/application_default_credentials.json:/sandbox/.config/gcloud/application_default_credentials.json" \
    -- bash -c '
        export CLAUDE_CODE_USE_VERTEX=1
        export ANTHROPIC_VERTEX_PROJECT_ID=<project-id>
        export CLOUD_ML_REGION=<region>
        export VERTEX_LOCATION=<region>
        export GOOGLE_APPLICATION_CREDENTIALS=/sandbox/.config/gcloud/application_default_credentials.json
        claude
    '
```

Ask the user for `ANTHROPIC_VERTEX_PROJECT_ID` and `VERTEX_LOCATION` if not already known. Check for the ADC file at `$HOME/.config/gcloud/application_default_credentials.json` before proceeding.

For direct Anthropic API:
```bash
openshell sandbox create \
    --provider <anthropic-provider-name> \
    -- claude
```

For OpenAI-based tools:
```bash
openshell sandbox create \
    --provider <openai-provider-name> \
    -- <agent-command>
```

### Connect to a running sandbox

```bash
openshell sandbox connect <name>
```

For VS Code Remote-SSH:
```bash
openshell sandbox ssh-config <name> >> ~/.ssh/config
```

### List sandboxes

```bash
openshell sandbox list
```

### View sandbox details

```bash
openshell sandbox get <name>
```

### Upload/download files

```bash
openshell sandbox upload <name> <local-path> [sandbox-dest]
openshell sandbox download <name> <sandbox-path> [local-dest]
```

### View sandbox logs

```bash
openshell logs <name>                              # Recent logs
openshell logs <name> --tail                       # Stream live
openshell logs <name> --tail --source sandbox      # Sandbox-only
openshell logs <name> --since 5m --level warn      # Last 5 min, warn+
```

### Delete sandboxes

```bash
openshell sandbox delete <name>
openshell sandbox delete <name1> <name2>   # Multiple at once
```

### Port forwarding

```bash
openshell forward start <port> <name> -d   # Background
openshell forward list                      # List active
openshell forward stop <port> <name>        # Stop
```

### BYOC (Bring Your Own Container)

```bash
openshell sandbox create --from ./Dockerfile --name my-app
openshell sandbox create --from myregistry.com/img:tag --name my-app
openshell sandbox create --from ./Dockerfile --forward 8080 -- ./start-server.sh
```

---

## Workflow 4: Provider Management

Providers supply credentials (API keys, tokens) to sandboxes.

### Supported provider types

`claude`, `opencode`, `codex`, `generic`, `nvidia`, `gitlab`, `github`, `outlook`

### Create a provider from local credentials

```bash
openshell provider create --name my-claude --type claude --from-existing
openshell provider create --name my-github --type github --from-existing
```

The `--from-existing` flag discovers credentials from local state (e.g., `gh auth` tokens, Claude config).

### Create with explicit credentials

```bash
openshell provider create --name my-api --type generic \
    --credential API_KEY=sk-abc123

# Read from env var (bare KEY without =VALUE):
openshell provider create --name my-api --type generic --credential API_KEY
```

### Vertex AI provider with token refresh

This is a multi-step process for Claude Code on Vertex AI:

**Step 1: Create or import a custom provider profile (one-time)**

The built-in `google-vertex-ai` profile doesn't include `oauth2.googleapis.com` in its endpoint list, which blocks ADC token refresh. A custom profile is needed.

Ask the user to save a profile YAML or create one for them. The profile must include endpoints for:
- `*-aiplatform.googleapis.com:443` (Vertex AI inference)
- `oauth2.googleapis.com:443` (token refresh)
- `statsig.anthropic.com:443` (Claude telemetry)
- `sentry.io:443` (error reporting)

Import it:
```bash
openshell provider profile import --file <profile-path>
```

**Step 2: Get a fresh access token**

```bash
ACCESS_TOKEN=$(gcloud auth application-default print-access-token)
```

If `gcloud` isn't authenticated:
```bash
gcloud auth application-default login
```

**Step 3: Create the provider**

```bash
openshell provider create \
    --name claude-vertex \
    --type claude-code-vertex \
    --credential "GOOGLE_VERTEX_AI_TOKEN=$ACCESS_TOKEN"
```

**Step 4: Configure automatic token refresh**

Extract ADC material and configure refresh:
```bash
ADC=$HOME/.config/gcloud/application_default_credentials.json
CLIENT_ID=$(python3 -c "import json; print(json.load(open('$ADC'))['client_id'])")
CLIENT_SECRET=$(python3 -c "import json; print(json.load(open('$ADC'))['client_secret'])")
REFRESH_TOKEN=$(python3 -c "import json; print(json.load(open('$ADC'))['refresh_token'])")

openshell provider refresh configure claude-vertex \
    --credential-key GOOGLE_VERTEX_AI_TOKEN \
    --strategy oauth2-refresh-token \
    --material "client_id=$CLIENT_ID" \
    --material "client_secret=$CLIENT_SECRET" \
    --material "refresh_token=$REFRESH_TOKEN" \
    --secret-material-key client_secret \
    --secret-material-key refresh_token
```

Verify:
```bash
openshell provider refresh status claude-vertex
```

### List, inspect, update, delete providers

```bash
openshell provider list
openshell provider get <name>
openshell provider update <name> --type <type> --from-existing
openshell provider delete <name>
```

### Refresh an existing provider's token

For Vertex AI providers that need a manual token refresh:
```bash
ACCESS_TOKEN=$(gcloud auth application-default print-access-token)
openshell provider update <name> --credential "GOOGLE_VERTEX_AI_TOKEN=$ACCESS_TOKEN"
```

---

## Workflow 5: Policy Management

Policies control filesystem access, network traffic, and process identity in sandboxes.

### Key concept

Policies have **static fields** (immutable after sandbox creation: `filesystem_policy`, `landlock`, `process`) and one **dynamic field** (`network_policies`). Only `network_policies` can be updated on a running sandbox without recreating it.

### Create a new policy file

Generate a policy YAML with sensible defaults:

```yaml
version: 1

filesystem_policy:
  include_workdir: true
  read_only:
    - /usr
    - /lib
    - /proc
    - /dev/urandom
    - /app
    - /etc
    - /var/log
  read_write:
    - /sandbox
    - /tmp
    - /dev/null

landlock:
  compatibility: best_effort

process:
  run_as_user: sandbox
  run_as_group: sandbox

network_policies: {}
```

Starting with `network_policies: {}` gives zero network access — the most secure starting point. Add endpoints incrementally.

### Create a sandbox with a policy

```bash
openshell sandbox create --name my-sandbox --policy ./my-policy.yaml -- <command>
```

### The policy iteration loop

This is the core workflow for refining a sandbox policy based on observed behavior:

```
1. Create sandbox with initial policy
2. Monitor logs for denied actions
3. Pull current policy
4. Modify policy (add allowed endpoints)
5. Push updated policy
6. Verify reload succeeded
7. Repeat from step 2
```

**Monitor logs for denials:**
```bash
openshell logs <name> --tail --source sandbox
```

Look for log lines with `action: deny` — these show:
- Destination host and port (what was blocked)
- Binary path (which process attempted the connection)
- Deny reason (why it was blocked)

**Pull current policy:**
```bash
openshell policy get <name> --full > current-policy.yaml
```

**Push updated policy:**
```bash
openshell policy set <name> --policy current-policy.yaml --wait
```

Exit codes with `--wait`: 0 = loaded, 1 = failed, 124 = timeout.

**Verify:**
```bash
openshell policy list <name>
```

Check that the latest revision shows status `loaded`.

### Incremental policy updates (no file editing needed)

For simple additions, use `policy update` instead of the full get/edit/set cycle:

**Add a network endpoint:**
```bash
openshell policy update <name> \
    --add-endpoint "api.github.com:443:read-only:rest:enforce" \
    --binary /usr/bin/curl \
    --wait
```

Endpoint spec format: `host:port[:access[:protocol[:enforcement]]]`
- `access`: `read-only`, `read-write`, `full`
- `protocol`: `rest`, `sql`
- `enforcement`: `enforce`, `audit`

**Add L7 allow rules to an existing endpoint:**
```bash
openshell policy update <name> \
    --add-allow "api.github.com:443:POST:/repos/*/issues" \
    --wait
```

**Add deny rules:**
```bash
openshell policy update <name> \
    --add-deny "api.github.com:443:DELETE:/repos/*" \
    --wait
```

**Remove an endpoint:**
```bash
openshell policy update <name> \
    --remove-endpoint "api.github.com:443" \
    --wait
```

**Remove a named rule:**
```bash
openshell policy update <name> \
    --remove-rule <rule-name> \
    --wait
```

**Preview changes without applying:**
```bash
openshell policy update <name> \
    --add-endpoint "api.github.com:443:read-only:rest:enforce" \
    --dry-run
```

### View policy revision history

```bash
openshell policy list <name>
openshell policy get <name> --rev 3 --full    # Specific revision
```

### Common policy patterns

**Read-only API access (preset):**
```yaml
network_policies:
  github_api:
    name: github_api
    endpoints:
      - host: api.github.com
        port: 443
        protocol: rest
        tls: terminate
        enforcement: enforce
        access: read-only
    binaries:
      - { path: /usr/bin/curl }
```

**Fine-grained L7 rules:**
```yaml
network_policies:
  github_api:
    name: github_api
    endpoints:
      - host: api.github.com
        port: 443
        protocol: rest
        tls: terminate
        enforcement: enforce
        rules:
          - allow:
              method: GET
              path: "/repos/**"
          - allow:
              method: POST
              path: "/repos/*/issues"
    binaries:
      - { path: /usr/bin/curl }
      - { path: /usr/bin/gh }
```

**L4 passthrough (no HTTP inspection):**
```yaml
network_policies:
  my_api:
    name: my_api
    endpoints:
      - { host: api.example.com, port: 443 }
    binaries:
      - { path: /usr/bin/curl }
```

**Locked-down (zero network, add incrementally):**
```yaml
network_policies: {}
```

### Policy rules

- `protocol: rest` on port 443 requires `tls: terminate` (proxy can't inspect encrypted traffic without it)
- `rules` and `access` are mutually exclusive on the same endpoint — use one or the other
- `*` in path globs does not cross `/` boundaries; use `**` for recursive matching
- Deny rules take precedence over allow rules

---

## Troubleshooting

### Common issues and fixes

| Symptom | Likely cause | Diagnosis | Fix |
|---|---|---|---|
| `policy_denied` on token refresh | Proxy blocking `oauth2.googleapis.com` | `openshell logs <name> --tail` shows denied host | Use a provider profile that includes `oauth2.googleapis.com` in endpoints |
| `Not logged in` inside sandbox | Missing env vars for Vertex AI | Check if `CLAUDE_CODE_USE_VERTEX=1` is set | Set env vars inside sandbox or in the create command |
| `Could not load the default credentials` | ADC file not uploaded or wrong path | Check if file exists at expected path in sandbox | Use `--upload` flag with correct paths |
| Gateway not running after reboot | Container stopped | `$RUNTIME ps -a \| grep openshell-gateway` | `$RUNTIME start openshell-gateway` |
| Sandbox create hangs | Gateway can't reach container runtime socket | `$RUNTIME logs openshell-gateway` | Verify socket path, restart gateway |
| `connection refused` on `openshell status` | Gateway not registered or not running | `openshell gateway select` to check registration | Re-register or start gateway |
| Port forward not working | Sandbox not running or port not exposed | `openshell sandbox get <name>` | Verify sandbox is running, use `--forward` on create |

### Diagnostic commands

```bash
openshell status                                    # Gateway health
openshell sandbox get <name>                        # Sandbox details + active policy
$RUNTIME logs openshell-gateway --tail 50           # Gateway container logs
openshell logs <name> --tail --source sandbox       # Sandbox logs (live)
openshell logs <name> --since 5m --level warn       # Recent warnings
openshell policy list <name>                        # Policy revision history
openshell provider refresh status <provider-name>   # Token refresh status
```

---

## Self-Teaching

The CLI has comprehensive built-in help. When you encounter a command or option not covered in this skill:

```bash
openshell --help                    # Top-level commands
openshell <group> --help            # Subcommands in a group
openshell <group> <cmd> --help      # Flags for a specific command
```

The CLI help is authoritative. If it contradicts this skill, follow the CLI.
