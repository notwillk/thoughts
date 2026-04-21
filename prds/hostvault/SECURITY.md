# HostVault Security Model

> **Navigation**: [Overview](./README.md) | [Protocol Spec](./PROTOCOL.md) | Security Model | [CLI Spec](./CLI.md) | [macOS App Spec](./MACOS-APP.md) | [Test Cases](./TEST-CASES.md)

## Table of Contents

- [Threat Model](#threat-model)
- [Security Features](#security-features)
- [Container Socket Security (Hardened)](#container-socket-security-hardened)
- [Privacy](#privacy)
- [Biometric Authentication Requirements](#biometric-authentication-requirements)
- [Security Mitigations Summary](#security-mitigations-summary)

## Threat Model

| Threat | Mitigation |
|--------|------------|
| Unauthorized socket access | Socket file permissions (0600), socket ownership verification on every connection |
| Eavesdropping on socket | Unix domain socket (local only, no network exposure) |
| Replay attacks | Request ID uniqueness check, timestamp validation (±5 minutes) |
| Secret interception in memory | Client clears memory after use, server uses secure buffers (memset_s on free) |
| Malicious client spoofing | Client PID/hostname verification, user confirmation + biometric auth required |
| Secret storage compromise | macOS Keychain encryption, no plaintext storage |
| Unknown/unauthorized env var enumeration | Strict allowlist - only configured names accessible |
| Unauthorized approval by attacker | Touch ID / Face ID / Passcode mandatory for ALL approvals |
| Social engineering attacks | Biometric requirement prevents "just click approve" attacks |
| Command injection via logged command | Command string displayed for user review, not executed by server |
| Log tampering | Logs stored with append-only permissions where possible, export before purge |

## Security Features

### 1. Socket Security

- Socket created with permissions 0600 (owner read/write only)
- Socket ownership verified on every connection
- Socket directory must be owned by user
- Default path: `~/Library/Application Support/HostVault/hostvault.sock`

### 2. Request Validation

- Request ID UUID v4 validation
- Timestamp within ±5 minutes of server time (prevent replay attacks)
- Secret name validation (alphanumeric + underscore only)
- Maximum 10 secrets per request (prevent DoS)

### 3. Secret Storage

- All secrets stored in macOS Keychain
- Server memory uses secure buffers (memset_s on free)
- No swap/pagefile for secret data (mlock where possible)
- Secret values never logged or displayed in UI

### 4. Authorization Enforcement

- **Biometric Authentication Required**: Touch ID, Face ID, or device passcode mandatory for ALL approvals
- **No exceptions**: Even whitelisted clients require biometric auth (whitelist only skips the dialog, not the biometric check)
- Server uses LocalAuthentication framework for biometric prompts
- Biometric auth has 3-attempt limit before falling back to passcode
- Failed biometric auth immediately denies request (no dialog shown)

### 5. Allowlist Enforcement

- Server maintains explicit allowlist of permitted env var names
- Client can ONLY request env vars explicitly added to the allowlist
- Request for any non-allowlist env var is automatically denied
- Server pre-filters requests: non-allowlist names are logged and rejected before authorization stage
- Empty allowlist = all requests blocked (fail-safe default)

### 6. Health Endpoint Security

- Health check endpoint requires **no authentication** (by design)
- Health check returns only: status, version, capabilities (no secrets, no user data)
- Health check response time < 100ms (fast, minimal resource usage)
- Health check does **not** log to audit log (reduces noise)
- Health check is **rate-limited**: max 10 req/sec from same connection (prevent DoS)

### 7. Command Display Security

- The command to be executed is included in the request for display purposes only
- Server NEVER executes the command - it only provides the secrets
- Command is displayed in the authorization dialog for user review
- Command is logged for audit purposes
- This prevents "hidden" command execution where user doesn't know what will run

## Container Socket Security (Hardened)

When running in containerized environments, HostVault implements a **hardened 3-tier architecture** with strict socket boundaries to prevent rogue process interference.

### External Interface (Only Entry Point)

**Socket Configuration**:
- **Type**: Unix domain socket
- **Path**: Configurable (default: `/run/hostvault/external.sock`)
- **Directory Permissions**: `0700` (owner rwx only)
- **Socket Permissions**: `0600` (owner read/write only)
- **Ownership**: Bootstrap proxy exclusively

**Behavior**:
- Only one process ever binds and accepts this socket
- That process is the bootstrap proxy
- No other container process can bind to or accept connections on this socket

### Bootstrap Proxy (Trusted Gatekeeper)

**Lifecycle**:
- Starts immediately at container launch (PID 1 or early init)
- First (and only) process allowed to interact with external socket
- Establishes connection to upstream host server
- Transitions into proxy mode after successful upstream connection

**Responsibilities**:
1. Bind external socket with exclusive ownership
2. Connect/accept on external socket (host→container traffic only)
3. Perform initial handshake with upstream server
4. Create internal IPC plane (socketpair)
5. Forward client requests with original context preserved

### Internal IPC Plane (Trusted Boundary)

After bootstrap completes, all internal container communication uses FD-based channels:

**Technology**: `socketpair(AF_UNIX, SOCK_STREAM)`

**Properties**:
- No filesystem socket path exposed
- Not discoverable via filesystem enumeration
- Not raceable via path lookup
- Only reachable via explicit FD handoff from proxy

**Alternative (if socketpair unavailable)**:
- Internal Unix socket in private directory: `/run/hostvault/internal/`
- Directory permissions: `0700`
- Only proxy knows the path

### Container Clients (Untrusted Except Proxy)

**Behavior**:
- Do NOT access external socket directly
- Communicate only with proxy via internal FD-based channel
- Have no ability to create new trusted connections to external socket

### Connection Flow

**Startup Phase (Critical Window)**:
1. Container starts
2. Bootstrap proxy starts immediately
3. Proxy binds external socket (exclusive ownership)
4. Proxy becomes only accepted peer on external socket

**Post-Bootstrap Steady State**:
```
external socket (host→container)
    ↓
bootstrap proxy (only accepted connection owner)
    ↓
internal socketpair / FD handoff
    ↓
client processes (local IPC only)
```

### Socket Configuration Summary

| Socket | Path | Permissions | Owner | Role |
|--------|------|-------------|-------|------|
| External | `/run/hostvault/external.sock` | 0700/0600 | Bootstrap proxy only | Ingress boundary only |
| Internal | FD-based (socketpair) | N/A (kernel-owned) | Proxy ↔ Clients | Trusted internal IPC |

### Security Properties Achieved

**Prevented Threats**:

| Threat | Mitigation |
|--------|------------|
| Inter-process MITM on external socket | Proxy exclusive ownership after bootstrap |
| Rogue process connecting later and hijacking traffic | External socket not accessible to non-proxy processes |
| Socket file replacement attacks | Permissions 0700/0600 + exclusive bind timing |
| Internal interception | FD-only channels, no filesystem presence |
| Path-based race conditions | socketpair has no filesystem path |

**Acknowledged Limitations**:
- Startup race window exists (external socket takeover before proxy starts)
- Same-privilege processes could interfere during bootstrap if they execute early enough
- No protection against fully compromised container runtime

### Core Invariant

> After bootstrap completes, all trusted communication flows only through kernel-owned, non-path-based file descriptor channels controlled exclusively by the proxy process.

This invariant ensures that even if a rogue process gains execution within the container after startup, it cannot intercept, impersonate, or hijack HostVault communication channels.

## Privacy

- Client hostname/user info only sent to server for authorization
- No telemetry or analytics
- No network communication (offline-only operation)
- Audit logs stored locally only
- Secret values never leave the macOS Keychain except when explicitly authorized

## Biometric Authentication Requirements

### Minimum Requirements

- macOS 10.15+ (Catalina) for Touch ID support
- LocalAuthentication.framework
- Device must have biometric hardware OR passcode enabled

### Biometric Policy

- `biometryCurrentSet` - Touch ID must be enrolled
- `userPresence` - Device passcode enabled (fallback)
- No biometric authentication sharing between users
- Authentication context invalidated after 5 minutes or on dialog close

### Fallback Order

1. Touch ID (if available and enrolled)
2. Face ID (if available on Mac with Face ID)
3. Device Passcode (always available as fallback)

## Security Mitigations Summary

| Attack Vector | Mitigation | Verification |
|--------------|------------|------------|
| Unauthorized socket access | 0600 permissions + ownership check | See [Test Cases](./TEST-CASES.md#socket-security-tests) |
| Replay attack | UUID + timestamp validation | See [Test Cases](./TEST-CASES.md#protocol-security-tests) |
| Memory scraping | Secure buffers + mlock | Code review + memory analysis |
| Social engineering | Mandatory biometric auth | See [Test Cases](./TEST-CASES.md#authorization-tests) |
| Allowlist bypass | Strict server-side filtering | See [Test Cases](./TEST-CASES.md#allowlist-tests) |
| Brute force auth | 3-attempt limit + passcode fallback | See [Test Cases](./TEST-CASES.md#biometric-tests) |
| Log tampering | Append-only logs, purge logged | File permissions audit |
| Container rogue process MITM | Bootstrap proxy + exclusive socket ownership | See [Test Cases](./TEST-CASES.md#bootstrap-proxy-security-tests) |
| Container socket replacement | 0700/0600 permissions + early bind | See [Test Cases](./TEST-CASES.md#bootstrap-proxy-security-tests) |
| Container internal interception | FD-only IPC (socketpair, no filesystem) | See [Test Cases](./TEST-CASES.md#bootstrap-proxy-security-tests) |

---

*See also: [Protocol Security](./PROTOCOL.md#socket-protocol-specification), [macOS App Security](./MACOS-APP.md#core-features), and detailed [Threat Model Test Cases](./TEST-CASES.md)*

---

*Document Version: 1.0*  
*Part of the [HostVault Product Requirements](./README.md)*
