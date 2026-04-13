# HostVault CLI Specification

> **Navigation**: [Overview](./README.md) | [Protocol Spec](./PROTOCOL.md) | [Security Model](./SECURITY.md) | CLI Spec | [macOS App Spec](./MACOS-APP.md) | [Test Cases](./TEST-CASES.md)

## Table of Contents

- [Connection Flow](#connection-flow-health-check-first)
- [Execution Pattern](#execution-pattern-with---)
- [CLI Options and Arguments](#cli-options-and-arguments)
- [Socket Path Resolution](#socket-path-resolution)
- [Exit Codes](#exit-codes)
- [Execution Model & Argument Parsing](#execution-model--argument-parsing)
- [Configuration](#configuration)
- [Error Handling](#error-handling)
- [Shell Integration](#shell-integration)

## Connection Flow: Health Check First

Before any secret request, the CLI **MUST** perform a health check to verify server availability:

1. **Connect to socket** (`~/Library/Application Support/HostVault/hostvault.sock`)
2. **Send health check request** (type: "health") - see [Protocol Spec](./PROTOCOL.md#health-check-endpoint)
3. **Wait for health OK response** (timeout: 500ms)
   - **If healthy**: Continue with full secret request + auth flow
   - **If timeout/no response**: **SHORT-CIRCUIT** with exit code 1 (connection error)
   - **If error response**: Exit with appropriate error code

This prevents waiting for the full biometric authorization flow if the server is down or unresponsive.

## Execution Pattern (with `--`)

The CLI follows the `env` tool pattern. When `--` is provided, the CLI:

1. **Health check** (short-circuit if server unavailable)
2. Connects to the server and requests the specified env vars
3. Waits for biometric authorization (user sees command in dialog)
4. Sets the retrieved values as environment variables
5. Executes the command after `--` with those env vars available

### Examples

```bash
# Execute command with env vars set
hv API_KEY_PROD DATABASE_URL -- node server.js
hv AWS_ACCESS_KEY AWS_SECRET_KEY -- terraform apply
hv DATABASE_URL -- psql $DATABASE_URL

# Multiple flags/args after --
hv API_KEY -- curl -H "Authorization: Bearer $API_KEY" https://api.example.com/data

# Complex commands
hv STRIPE_KEY GITHUB_TOKEN -- bash -c 'echo "Stripe: $STRIPE_KEY" && git push'
```

## CLI Options and Arguments

### Usage

```
hv [OPTIONS] ENV1 [ENV2 ENV3 ...] -- COMMAND [ARGS...]
```

**Note**: The `--` separator is **required**. The CLI always executes a command with the requested secrets; there is no mode to view secrets directly.

### Options

| Option | Description | Default |
|--------|-------------|---------|
| `--socket <PATH>` | Path to the Unix domain socket file | `~/Library/Application Support/HostVault/hostvault.sock` |
| `--status` | Check server health and exit | - |
| `--version` | Show version information and exit | - |
| `-h, --help` | Show help message and exit | - |

### Standalone Operations

```bash
# Check server status (health check only, no secrets requested)
hv --status
# Output: Server is running (version 1.0.0)
# Or: Error: Server not responding (timeout 500ms)

# Show version
hv --version
```

### Socket Path Resolution

The socket path is resolved in the following priority order:

1. `--socket` CLI flag (highest priority)
2. `HV_SOCKET` environment variable
3. `socket_path` in config file (`~/.config/hv/config.yaml`)
4. Default: `~/Library/Application Support/HostVault/hostvault.sock`

### Examples

```bash
# Use custom socket path
hv --socket /var/run/hv.sock API_KEY_PROD -- node app.js

# Socket path via environment variable
export HV_SOCKET=/custom/path.sock
hv API_KEY_PROD DATABASE_URL

# Check server status with custom socket
hv --socket /tmp/custom.sock --status
```

## Exit Codes

| Code | Meaning | Description |
|------|---------|-------------|
| `0` | Success | Command executed successfully with requested secrets |
| `1` | Connection Error | Health check failed, cannot connect to server, or socket not found |
| `2` | Authorization Denied | User explicitly denied the request or biometric auth failed |
| `3` | Timeout | Authorization dialog timed out |
| `4` | Secret Not Found | One or more requested secrets don't exist in allowlist or have no value set |
| `5` | Invalid Request | Malformed request or invalid secret name format |
| `6` | Protocol Mismatch | Version mismatch between client and server |
| `7` | Permission Denied | Client lacks permission to access socket |
| `8` | Not In Allowlist | Requested env var(s) not in server's configured allowlist |
| `9` | Biometric Failed | Biometric authentication failed or was cancelled |
| `10` | Command Not Found | The command after `--` was not found or not executable |
| `11` | Command Failed | Command executed but returned non-zero exit code |

### Health Check Short-Circuit

The CLI **always** performs a health check first (max 500ms). If this fails:
- **Exit code 1** is returned immediately
- **No biometric auth** is attempted
- **No command** is executed (in execution mode)
- Clear error message indicates server is unavailable

### Exit Code Rules for `-- command`

- Codes `1-9`: Connection/auth/validation failures (command never runs)
- Code `10`: Command after `--` not found or not executable
- Code `11`: Command executed but returned non-zero
- Otherwise: The exit code of the executed command

## Execution Model & Argument Parsing

The CLI requires the `--` separator to distinguish between secret names and the command to execute.

### Argument Parsing Rules

```
hv [OPTIONS] ENV1 [ENV2 ENV3 ...] -- COMMAND [ARGS...]
                                                      ^^^^ everything after -- is the command
```

- All arguments before `--` are env var names to request from the server
- All arguments after `--` form the command to execute
- `--` itself is NOT included in the command
- The command is executed with the retrieved env vars added to its environment
- Original env vars are preserved; retrieved vars are added/override
- The full command string is sent to the server for display in the auth dialog
- **No inspection mode**: Secrets are never output directly; they are only passed to the specified command

### Examples

**Execution Mode**:
```bash
# Simple command
hv API_KEY -- node app.js
# Executes: API_KEY=<secret> node app.js

# Command with arguments
hv AWS_KEY AWS_SECRET -- aws s3 ls s3://bucket
# Executes: AWS_KEY=<k> AWS_SECRET=<s> aws s3 ls s3://bucket

# Multiple env vars, complex command
hv TOKEN API_URL -- curl -H "Authorization: Bearer $TOKEN" "$API_URL/data"
# Note: Shell expands $TOKEN and $API_URL from the retrieved values

# Script with args
hv DB_URL -- ./script.sh --production --verbose
# All args after -- passed to the script
```

## Configuration

**Config File**: `~/.config/hv/config.yaml`

```yaml
socket_path: ~/Library/Application Support/HostVault/hostvault.sock
timeout: 30
retry:
  count: 3
  delay: 1
```

## Error Handling

### Health Check Failures (short-circuit, no auth attempted)

```bash
# Server not running - health check timeout
$ hv API_KEY -- ./deploy.sh
Error: Server health check failed: Connection timeout (500ms)
         The HostVault is not responding.
         Is the HostVault running on the macOS host?
         Hint: Check the menu bar app is running and the socket exists.
(Exit code 1)

# Server running but slow/unresponsive
$ hv API_KEY -- ./deploy.sh
Error: Server health check failed: Response timeout after 500ms
         The HostVault is running but not responding to health checks.
         Try restarting the HostVault application.
(Exit code 1)

# Socket not found
$ hv API_KEY -- ./deploy.sh
Error: Server health check failed: Socket not found at ~/Library/Application Support/HostVault/hostvault.sock
         Is the HostVault running?
         Hint: Start the HostVault from the macOS menu bar.
(Exit code 1)
```

### Error Examples

```bash
# Successful execution
$ hv API_KEY -- ./my-script.sh
Running with API_KEY=***... (hidden)
Script output here
(Exit code: whatever my-script.sh returned)

# Command not found
$ hv API_KEY -- nonexistent-cmd
Error: Command not found: nonexistent-cmd
         Ensure the command exists and is executable.
(Exit code 10)

# Auth denied (command never runs) - health check passed but auth failed
$ hv API_KEY -- ./deploy.sh
Error: Authorization denied by user
         Biometric prompt failed or was cancelled.
         Command './deploy.sh' was NOT executed.
(Exit code 2)

# Allowlist violation (command never runs) - health check passed but env var not in list
$ hv UNAUTHORIZED_VAR -- ./run.sh
Error: Not in allowlist: UNAUTHORIZED_VAR
         This env var name is not in the server's configured allowlist.
         Command './run.sh' was NOT executed.
(Exit code 8)

# Server stopped between health check and full request (very rare)
$ hv API_KEY -- ./deploy.sh
Error: Server disconnected during request
         The HostVault stopped responding after health check.
         Command './deploy.sh' was NOT executed.
(Exit code 1)
```

## Shell Integration

```bash
# Use in subshell
(hv API_KEY -- ./script.sh)
# Env vars only available within the subshell

# Chain multiple secrets
hv SECRET1 -- hv SECRET2 -- ./app
# Each requires separate biometric approval (by design)

# Note: Secrets are never output directly. To use secrets in the current shell,
# you must execute a command that uses them. For example:
#   hv API_KEY -- bash -c 'echo "Using API_KEY: $API_KEY"'
# Or write a wrapper script that reads the secret from the environment.
```

---

*See also: [Protocol Messages](./PROTOCOL.md#message-types), [Security Model](./SECURITY.md), and [Test Cases](./TEST-CASES.md)*

---

*Document Version: 1.0*  
*Part of the [HostVault Product Requirements](./README.md)*
