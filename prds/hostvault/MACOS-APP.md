# HostVault macOS App Specification

> **Navigation**: [Overview](./README.md) | [Protocol Spec](./PROTOCOL.md) | [Security Model](./SECURITY.md) | [CLI Spec](./CLI.md) | macOS App Spec | [Test Cases](./TEST-CASES.md)

## Table of Contents

- [Core Features](#core-features)
- [UI/UX Design](#uiux-design)
- [Configuration](#configuration)
- [Audit Logging](#audit-logging)
- [Component Breakdown](#component-breakdown)
- [Allowlist Configuration Format](#allowlist-configuration-format)

## Core Features

### Secret Management (Allowlist Model)

The server operates on a strict **allowlist model** where ONLY explicitly configured environment variable names can be accessed. All other requests are automatically blocked.

**Configuration Panel Features**:
- **Define Available Env Vars**: Create and manage the list of allowed environment variable names that clients may request
- **Set Secret Values**: For each env var name, store its corresponding secret value in macOS Keychain
- **Add by Existing Env Var**: Auto-detect and add environment variable names currently set in the shell (e.g., scan `env` and pick keys to expose)
- **Allowlist Enforcement**: Server maintains a strict allowlist; any client request for env vars NOT in this list is automatically denied with `not_in_allowlist` reason
- **Edit Secret**: Modify secret values for existing allowed env var names
- **Delete Secret**: Remove env var names from allowlist and delete their values
- **View Allowlist**: Display all configured env var names (secret values masked)
- **Import/Export**: JSON format for backup/restore of allowlist and values

### Authorization Flow with Biometric Authentication

**Mandatory Biometric Authentication**:
All authorization decisions MUST be authenticated via Touch ID, Face ID, or device passcode. No "Approve" buttons without biometric verification.

**Authorization Flow**:
1. Client connects and sends request for env var names
2. Server validates that ALL requested names are in the allowlist (any non-allowlist request is immediately denied)
3. Server prompts for biometric authentication (Touch ID / Face ID / Passcode)
4. Upon successful biometric auth, authorization dialog is displayed showing request details including the command to be executed
5. User can then approve or deny the request
6. Denied biometric auth immediately denies the request

**Authorization Dialog Requirements** (shown only after successful biometric auth):
- Modal window showing:
  - 🔒 "Authenticated via Touch ID" badge
  - Client hostname, user, PID, and CWD
  - **Command to be executed**
  - List of requested env var names (from allowlist)
  - Warning if any non-allowlist secrets were requested (already filtered out)
  - "Approve" button (requires biometric re-auth on next request)
  - "Deny" button
  - 30-second timeout with countdown timer

### Health Check Endpoint

The server exposes a simple health check endpoint for client pre-flight checks:

| Property | Value |
|----------|-------|
| **Endpoint** | Same socket, message type `"health"` |
| **Response Time** | Must respond within 100ms |
| **Authentication** | None required |
| **Logging** | Audit log entry marked as healthcheck |
| **Rate Limiting** | 10 requests/second per connection |

**Purpose**: Allow CLI to quickly determine if server is available before attempting full auth flow. This prevents user confusion (waiting for Touch ID when server is down).

### Whitelist/Blacklist

- **Per-Client Whitelist**: Remember approved clients by hostname+user hash
- **Per-Secret Rules**: Restrict which clients can request specific secrets
- **Time-Based Rules**: Expire whitelist entries after configurable duration

## UI/UX Design

### Menu Bar App

- Lives in macOS menu bar (system tray)
- Dropdown menu with:
  - "Manage Secrets" → Opens management window
  - "View Logs" → Opens audit log viewer
  - "Settings" → Opens preferences
  - "Quit" → Exit application

### Access Log Management

The server GUI must provide comprehensive access log collection and management:

**Log Collection Requirements**:
- Collect all access attempts including successful and **failed attempts**
- Log entries must include: timestamp, client hostname, client user, PID, requested env vars, **command to be executed** (if any), decision (approved/denied), reason for denial, and biometric auth status
- Failed authentication attempts (biometric failure, timeout, allowlist violations) must be explicitly logged
- Logs stored locally in JSON Lines format with rotation support

**Log Purge Capabilities**:
- GUI must provide ability to purge logs by date range (e.g., "older than 30 days")
- Option to purge all logs with confirmation dialog
- Purge operation must be logged itself for audit trail completeness
- Export logs before purge (optional but recommended)

### Secret Configuration Panel (Allowlist Management)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ HostVault - Configure Env Vars                                        [+]   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─ Allowed Env Var Names ──────────────────────────────────────┐   │
│  │                                                              │   │
│  │ Env Var Name        │ Last Access │ Actions              │   │
│  ├──────────────────────────────────────────────────────────────┤   │
│  │ API_KEY_PROD        │ 2 hours ago │ [✏ Set] [🗑 Del]   │   │
│  │ DATABASE_URL        │ 1 day ago   │ [✏ Set] [🗑 Del]   │   │
│  │ GITHUB_TOKEN        │ Never       │ [✏ Set] [🗑 Del]    │   │
│  │ AWS_ACCESS_KEY      │ 3 days ago  │ [✏ Set] [🗑 Del]   │   │
│  │ STRIPE_SECRET_KEY   │ Never       │ [✏ Set] [🗑 Del]   │   │
│  │                                                                     │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Total: 5 env vars configured                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Authorization Dialog with Biometric Auth

**Step 1: Biometric Authentication Prompt**

```
┌────────────────────────────────────────────────────────────┐
│ 🔐 Authentication Required                                  │
├────────────────────────────────────────────────────────────┤
│                                                            │
│     ┌─────────────┐                                        │
│     │             │                                        │
│     │   🔐 Touch   │                                        │
│     │    ID       │                                        │
│     │             │                                        │
│     └─────────────┘                                        │
│                                                            │
│  A client is requesting access to environment variables.   │
│                                                            │
│  Please authenticate to view and authorize this request.     │
│                                                            │
│  [  Use Passcode Instead  ]                                │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**Step 2: Authorization Dialog (after biometric success)**

```
┌────────────────────────────────────────────────────────────┐
│ ✅ Authenticated via Touch ID    Secret Access    00:27    │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  A client is requesting access to the following env vars:  │
│                                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ✅ API_KEY_PROD      (allowed - in allowlist)       │   │
│  │ ✅ DATABASE_URL      (allowed - in allowlist)       │   │
│  │ ❌ UNAUTHORIZED_VAR  (BLOCKED - not in allowlist)   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                            │
│  Command to Execute:                                       │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ $ node server.js                                    │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                            │
│  Client Details:                                           │
│    Host: dev-server-01.company.local                       │
│    User: deploybot                                         │
│    PID:  48291                                             │
│    CWD:  /opt/app/deployment                               │
│                                                            │
│  [  ] Remember this client (still requires biometric)      │
│                                                            │
│      [  👍  Approve  ]  [  👎  Deny  ]                     │
│                                                            │
│  Note: Approval valid for this request only.             │
│        Biometric auth required for all secret requests.     │
└────────────────────────────────────────────────────────────┘
```

## Configuration

**Settings Window**:

| Setting | Default | Description |
|---------|---------|-------------|
| **Socket Path** | `~/Library/Application Support/HostVault/hostvault.sock` | Unix domain socket location |
| **Authorization Timeout** | 30 seconds | How long user has to approve/deny |
| **Auto-Start** | Off | Launch at login option |
| **Logging Level** | Info | Debug/Info/Warning/Error |
| **Whitelist Duration** | 24 hours | How long to remember approved clients |
| **Theme** | System | Light/Dark/System |

## Audit Logging

### Log Format (JSON Lines)

```json
{
  "timestamp": "2024-01-15T09:23:45.123Z",
  "event": "authorization_request",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "client": {
    "hostname": "dev-server-01",
    "user": "deploybot",
    "pid": 48291
  },
  "secrets_requested": ["API_KEY_PROD", "DATABASE_URL"],
  "command": "node server.js",
  "decision": "approved",
  "decision_by": "user",
  "duration_ms": 3200
}
```

**Note**: The `command` field contains the full command string that will be executed.

### Log Events

| Event | Description |
|-------|-------------|
| `server_start` / `server_stop` | Server lifecycle events |
| `client_connected` / `client_disconnected` | Connection events |
| `authorization_request` | Request received from client |
| `authorization_approved` / `authorization_denied` | User decision events |
| `secret_accessed` | Secret value retrieved from Keychain |
| `error` | Error conditions |
| `authentication_failed` | Biometric failure, timeout, or passcode failure |
| `allowlist_violation` | Attempt to access non-allowlisted env var |
| `log_purged` | When admin purges logs |

## Component Breakdown

### Server (macOS)

- **Platform**: macOS 10.15+ (Catalina and later) for Touch ID support
- **Language**: Swift with SwiftUI for GUI
- **Socket Path**: `~/Library/Application Support/HostVault/hostvault.sock` (configurable)
- **Allowlist Storage**: JSON file (`~/Library/Application Support/HostVault/`)
- **Secret Storage**: macOS Keychain for secret values (keyed by env var name)
- **Biometric Auth**: LocalAuthentication framework (Touch ID / Face ID / Passcode)
- **Health Endpoint**: Simple ping/pong endpoint for availability checks (no auth, <100ms response)
- **GUI**: Menu bar app with configuration panel and authorization dialogs

**Core Components**:

1. **Allowlist Manager**: Maintains list of permitted env var names; all requests filtered against this list
2. **Biometric Auth Engine**: Handles Touch ID / Face ID / Passcode prompts using LocalAuthentication
3. **Authorization Dialog**: GUI shown only after successful biometric authentication, displays command to be executed
4. **Socket Handler**: Manages Unix domain socket connections and JSON protocol
5. **Secret Store**: macOS Keychain wrapper for encrypted secret value storage
6. **Health Handler**: Fast endpoint that responds immediately without auth checks (used for client pre-flight)
7. **Log Manager**: Collects, stores, and manages audit logs with purge capabilities

## Allowlist Configuration Format

The server stores the allowlist configuration in `~/Library/Application Support/HostVault/allowlist.json`:

```json
{
  "version": "1.0",
  "updated_at": "2024-01-15T10:30:00Z",
  "env_vars": [
    {
      "name": "API_KEY_PROD",
      "created_at": "2024-01-10T08:00:00Z",
      "updated_at": "2024-01-15T09:00:00Z",
      "access_count": 42,
      "last_access": "2024-01-15T14:22:00Z"
    },
    {
      "name": "DATABASE_URL",
      "created_at": "2024-01-12T10:00:00Z",
      "updated_at": null,
      "access_count": 0,
      "last_access": null
    }
  ]
}
```

Secret **values** are stored separately in macOS Keychain, keyed by env var name.

---

*See also: [Protocol Specification](./PROTOCOL.md), [Security Model](./SECURITY.md), [CLI Usage](./CLI.md), and [Test Cases](./TEST-CASES.md)*

---

*Document Version: 1.0*  
*Part of the [HostVault Product Requirements](./README.md)*
