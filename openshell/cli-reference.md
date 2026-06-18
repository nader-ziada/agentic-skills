# OpenShell CLI Reference

Quick-reference for the `openshell` CLI. For workflow guidance, see [SKILL.md](SKILL.md).

> **Self-teaching**: If a command or flag is not listed here, use `openshell <command> --help` to discover it. The CLI help is authoritative and always up-to-date.

## Global Options

| Flag | Description |
|------|-------------|
| `-v`, `--verbose` | Increase verbosity (`-v` = info, `-vv` = debug, `-vvv` = trace) |
| `-g`, `--gateway <NAME>` | Gateway to operate on. Also settable via `OPENSHELL_GATEWAY` env var. |

## Environment Variables

| Variable | Description |
|----------|-------------|
| `OPENSHELL_GATEWAY` | Override active gateway name (same as `--gateway`) |
| `OPENSHELL_SANDBOX_POLICY` | Path to default sandbox policy YAML (fallback when `--policy` is not provided) |

---

## Command Tree

```
openshell
├── gateway
│   ├── add <endpoint> [--name NAME] [--local] [--remote USER@HOST]
│   ├── login [name]
│   ├── logout [name]
│   ├── remove [name]
│   ├── info [--name NAME]
│   ├── list
│   └── select [name]
├── status
├── inference
│   ├── set --provider NAME --model ID
│   ├── update [--provider NAME] [--model ID]
│   └── get
├── sandbox
│   ├── create [--name N] [--from SRC] [--provider P] [--policy PATH]
│   │          [--upload PATH[:DEST]] [--no-keep] [--forward PORT]
│   │          [--cpu QTY] [--memory QTY] [--tty|--no-tty]
│   │          [--auto-providers|--no-auto-providers] [-- CMD...]
│   ├── get <name> [--policy-only]
│   ├── list [--limit N] [--offset N] [--ids] [--names]
│   ├── delete <name>...
│   ├── connect <name>
│   ├── upload <name> <path> [dest]
│   ├── download <name> <path> [dest]
│   ├── ssh-config <name>
│   └── image push [opts]
├── forward
│   ├── start <port> <name> [-d]
│   ├── stop <port> <name>
│   └── list
├── logs <name> [-n N] [--tail] [--since DURATION]
│               [--source gateway|sandbox|all] [--level LEVEL]
├── policy
│   ├── set <name> --policy PATH [--wait] [--timeout SECS]
│   ├── get <name> [--rev VERSION] [--full]
│   ├── list <name> [--limit N]
│   └── update <name> [--add-endpoint SPEC] [--remove-endpoint SPEC]
│                      [--add-allow SPEC] [--add-deny SPEC]
│                      [--remove-rule NAME] [--binary PATH]
│                      [--rule-name NAME] [--dry-run]
│                      [--wait] [--timeout SECS]
├── provider
│   ├── create --name NAME --type TYPE [--from-existing]
│   │          [--credential KEY[=VALUE]] [--config KEY=VALUE]
│   ├── get <name>
│   ├── list [--limit N] [--offset N] [--names]
│   ├── update <name> --type TYPE [same flags as create]
│   ├── delete <name>...
│   ├── profile import --file PATH
│   └── refresh
│       ├── configure <name> --credential-key KEY --strategy STRAT
│       │             [--material KEY=VALUE] [--secret-material-key KEY]
│       └── status <name>
├── doctor check
├── term
├── completions <shell>
└── ssh-proxy [opts]
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Install (macOS) | `brew install openshell` |
| Install (Linux) | `curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh \| sh` |
| Register local gateway | `openshell gateway add https://localhost:17670 --local` |
| Check gateway health | `openshell status` |
| List/switch gateways | `openshell gateway select [name]` |
| Create sandbox (interactive) | `openshell sandbox create` |
| Create sandbox with agent | `openshell sandbox create -- claude` |
| Create with custom policy | `openshell sandbox create --policy ./p.yaml` |
| Create with provider | `openshell sandbox create --provider my-claude -- claude` |
| Create from Dockerfile | `openshell sandbox create --from ./Dockerfile` |
| Connect to sandbox | `openshell sandbox connect <name>` |
| VS Code SSH config | `openshell sandbox ssh-config <name> >> ~/.ssh/config` |
| Upload files | `openshell sandbox upload <name> <path> [dest]` |
| Download files | `openshell sandbox download <name> <path> [dest]` |
| Stream live logs | `openshell logs <name> --tail` |
| Logs filtered by source | `openshell logs <name> --tail --source sandbox --level warn` |
| Recent logs (5 min) | `openshell logs <name> --since 5m` |
| Pull current policy | `openshell policy get <name> --full > p.yaml` |
| Push updated policy | `openshell policy set <name> --policy p.yaml --wait` |
| Incremental endpoint add | `openshell policy update <name> --add-endpoint host:443:read-only:rest:enforce --binary /usr/bin/curl --wait` |
| Add L7 allow rule | `openshell policy update <name> --add-allow "host:443:POST:/path" --wait` |
| Remove endpoint | `openshell policy update <name> --remove-endpoint host:443 --wait` |
| Preview policy change | `openshell policy update <name> --add-endpoint ... --dry-run` |
| Policy revision history | `openshell policy list <name>` |
| Specific policy revision | `openshell policy get <name> --rev 3 --full` |
| Create provider (auto) | `openshell provider create --name N --type T --from-existing` |
| Create provider (manual) | `openshell provider create --name N --type T --credential KEY=VALUE` |
| Import provider profile | `openshell provider profile import --file path.yaml` |
| Configure token refresh | `openshell provider refresh configure <name> --credential-key KEY --strategy STRAT --material K=V` |
| Check refresh status | `openshell provider refresh status <name>` |
| List providers | `openshell provider list` |
| Delete sandbox | `openshell sandbox delete <name>` |
| Delete provider | `openshell provider delete <name>` |
| Forward port (background) | `openshell forward start <port> <name> -d` |
| List forwards | `openshell forward list` |
| Stop forward | `openshell forward stop <port> <name>` |
| Launch TUI | `openshell term` |
| Self-teach any command | `openshell <group> <cmd> --help` |

---

## Endpoint Spec Format

Used with `--add-endpoint` and `--remove-endpoint`:

```
host:port[:access[:protocol[:enforcement]]]
```

| Field | Values | Default |
|-------|--------|---------|
| `host` | hostname or `*-pattern` | required |
| `port` | integer | required |
| `access` | `read-only`, `read-write`, `full` | none (L4) |
| `protocol` | `rest`, `sql` | none (L4) |
| `enforcement` | `enforce`, `audit` | `enforce` |

## Allow/Deny Spec Format

Used with `--add-allow` and `--add-deny`:

```
host:port:METHOD:path_glob
```

Example: `api.github.com:443:POST:/repos/*/issues`

## Provider Types

`claude`, `opencode`, `codex`, `generic`, `nvidia`, `gitlab`, `github`, `outlook`
