# HostVault Security Model

> **Navigation**: [Overview](./README.md) | [Protocol Spec](./PROTOCOL.md) | Security Model | [CLI Spec](./CLI.md) | [macOS App Spec](./MACOS-APP.md) | [Test Cases](./TEST-CASES.md)

## Table of Contents

- [Threat Model](#threat-model)
- [Security Features](#security-features)
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
|--------------|------------|--------------|
| Unauthorized socket access | 0600 permissions + ownership check | See [Test Cases](./TEST-CASES.md#socket-security-tests) |
| Replay attack | UUID + timestamp validation | See [Test Cases](./TEST-CASES.md#protocol-security-tests) |
| Memory scraping | Secure buffers + mlock | Code review + memory analysis |
| Social engineering | Mandatory biometric auth | See [Test Cases](./TEST-CASES.md#authorization-tests) |
| Allowlist bypass | Strict server-side filtering | See [Test Cases](./TEST-CASES.md#allowlist-tests) |
| Brute force auth | 3-attempt limit + passcode fallback | See [Test Cases](./TEST-CASES.md#biometric-tests) |
| Log tampering | Append-only logs, purge logged | File permissions audit |

---

*See also: [Protocol Security](./PROTOCOL.md#socket-protocol-specification), [macOS App Security](./MACOS-APP.md#core-features), and detailed [Threat Model Test Cases](./TEST-CASES.md)*

---

*Document Version: 1.0*  
*Part of the [HostVault Product Requirements](./README.md)*
