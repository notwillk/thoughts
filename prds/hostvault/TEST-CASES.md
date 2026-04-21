# HostVault Threat Model Test Cases

> **Navigation**: [Overview](./README.md) | [Protocol Spec](./PROTOCOL.md) | [Security Model](./SECURITY.md) | [CLI Spec](./CLI.md) | [macOS App Spec](./MACOS-APP.md) | Test Cases

## Table of Contents

- [Socket Security Tests](#socket-security-tests)
- [Protocol Security Tests](#protocol-security-tests)
- [Authorization Tests](#authorization-tests)
- [Biometric Tests](#biometric-tests)
- [Allowlist Tests](#allowlist-tests)
- [Health Endpoint Tests](#health-endpoint-tests)
- [Command Display Tests](#command-display-tests)
- [Log Security Tests](#log-security-tests)
- [Bootstrap Proxy Security Tests](#bootstrap-proxy-security-tests)
- [Integration Test Scenarios](#integration-test-scenarios)
- [Manual Test Checklist](#manual-test-checklist)

## Socket Security Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| SSK-001 | Socket file permissions | Check socket file permissions | Permissions must be 0600 (owner read/write only) |
| SSK-002 | Socket ownership verification | Attempt connection from different user | Connection denied |
| SSK-003 | Socket directory ownership | Check parent directory ownership | Must be owned by user |
| SSK-004 | Socket cleanup on exit | Stop server, check socket | Socket file removed or unavailable |
| SSK-005 | Socket path permissions | Check `~/Library/Application Support/HostVault/` | Directory must be 0700 |

## Protocol Security Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| PRP-001 | UUID validation | Send request with invalid request_id | Request rejected with error |
| PRP-002 | Timestamp replay attack | Send request with timestamp > 5 min old | Request rejected (replay detected) |
| PRP-003 | Future timestamp | Send request with timestamp 10 min in future | Request rejected |
| PRP-004 | Missing version field | Send request without version | Request rejected with protocol error |
| PRP-005 | Invalid JSON | Send malformed JSON | Connection closed, error logged |
| PRP-006 | Secret name validation | Request secret with invalid chars (e.g., `SECRET!@#`) | Rejected with invalid_request |
| PRP-007 | Too many secrets | Request 11 secrets in one request | Rejected with invalid_request |
| PRP-008 | Request ID uniqueness | Send duplicate request_id | Second request rejected as duplicate |

## Authorization Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| AUT-001 | Biometric required for approval | Attempt to approve without biometric | Approval blocked, request denied |
| AUT-002 | Dialog shows command | Send request with command `-- node app.js` | Command displayed in dialog |
| AUT-003 | User can deny request | Click "Deny" after biometric auth | Request denied, exit code 2 |
| AUT-004 | Timeout handling | Wait 30 seconds without action | Request times out, exit code 3 |
| AUT-005 | Remember client option | Check "Remember this client" and approve | Client added to whitelist |
| AUT-006 | Remembered client still requires biometric | Send second request from whitelisted client | Biometric prompt still shown |
| AUT-007 | Command displayed for review | Send `hv API_KEY -- rm -rf /` | Full command shown in dialog before approval |

## Biometric Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| BIO-001 | Touch ID success | Approve with valid Touch ID | Request approved, secrets sent |
| BIO-002 | Touch ID failure | Fail Touch ID 3 times | Falls back to passcode prompt |
| BIO-003 | Passcode fallback | Use "Use Passcode Instead" | Passcode prompt shown |
| BIO-004 | Passcode success | Enter correct passcode | Request proceeds to dialog |
| BIO-005 | Passcode failure | Enter wrong passcode 3 times | Request denied, exit code 9 |
| BIO-006 | Cancel biometric | Press Cancel on biometric prompt | Request denied immediately, exit code 9 |
| BIO-007 | No biometric enrolled | Attempt auth with no Touch ID enrolled | Falls back to passcode |
| BIO-008 | Context invalidation | Wait 5 min after auth, try again | New biometric prompt shown |

## Allowlist Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| ALW-001 | Allowlist blocks non-listed | Request env var not in allowlist | Rejected with not_in_allowlist, exit code 8 |
| ALW-002 | Empty allowlist blocks all | Request any secret with empty allowlist | All requests blocked |
| ALW-003 | Partial allowlist match | Request 2 secrets, 1 in allowlist, 1 not | Partial response with blocked list |
| ALW-004 | Case sensitivity | Request `api_key_prod` when allowlist has `API_KEY_PROD` | Blocked (case sensitive) |
| ALW-005 | Add to allowlist | Add new env var name via GUI | Name appears in allowlist, can be requested |
| ALW-006 | Remove from allowlist | Delete env var from GUI | Future requests blocked |
| ALW-007 | Secret without value | Request env var in allowlist but no value set | Rejected with secret_not_found |
| ALW-008 | Whitespace in name | Add secret name with spaces | Rejected (invalid format) |

## Health Endpoint Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| HLT-001 | Health check fast | Send health check, measure response | Response < 100ms |
| HLT-002 | Health check no auth | Send health check | No biometric prompt shown |
| HLT-003 | Health check no logging | Send health check, check logs | No audit log entry for health check |
| HLT-004 | Health check rate limiting | Send 15 health checks rapidly | Last 5 rejected or rate-limited |
| HLT-005 | CLI health check short-circuit | Stop server, run `hv API_KEY -- cmd` | Exits immediately with code 1 (< 500ms) |
| HLT-006 | Health check response format | Send health check | Response matches [protocol spec](./PROTOCOL.md#health-check-endpoint) |

## Command Display Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| CMD-001 | Command in request | Send request with `-- node server.js` | Command included in JSON request |
| CMD-002 | Command displayed to user | Send complex command | Full command shown in auth dialog |
| CMD-003 | Command logged | Approve request with command | Command appears in audit log |
| CMD-004 | Long command handling | Send command with 500+ chars | Displayed properly (truncated or scrollable) |
| CMD-006 | Special characters in command | Send `hv KEY -- echo "hello; world"` | Command displayed correctly |
| CMD-007 | Command not executed by server | Send request, approve | Server does NOT execute command, only client does |

## Log Security Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| LOG-001 | Secret values not logged | Request secret, check logs | Secret values never appear in logs |
| LOG-002 | Failed attempts logged | Deny request, fail biometric | Logged as denied/authentication_failed |
| LOG-003 | Purge creates log entry | Purge logs older than 30 days | `log_purged` event logged |
| LOG-004 | Log file permissions | Check log file permissions | Append-only or 0600 |
| LOG-005 | Log rotation | Generate large log file | Automatic rotation occurs |
| LOG-006 | Export before purge | Purge with export option | JSON export created before purge |
| LOG-007 | Command in log | Approve request with command | Command field present in log entry |

## Bootstrap Proxy Security Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| BPS-001 | Binary busybox pattern | Run `ln -s hv hv-proxy && ./hv-proxy --version` | Shows proxy version, not client |
| BPS-002 | External socket exclusive bind | Start proxy, try second instance on same socket | Second instance fails with "address already in use" |
| BPS-003 | External socket default permissions | Check `/run/hostvault/` and `external.sock` after startup | Directory 0700, socket 0600 |
| BPS-004 | External socket configurable | Start with `--external-socket /tmp/test.sock` | Creates socket at specified path with correct permissions |
| BPS-005 | Internal IPC no filesystem presence | List `/proc/self/fd` of client process | Shows socket FD, no path in filesystem |
| BPS-006 | Rogue process external connect | Process tries to connect to external.sock directly | Connection refused or permission denied |
| BPS-007 | Proxy startup wins race | Start proxy and rogue simultaneously | Proxy obtains exclusive bind, rogue fails |
| BPS-008 | FD passing to client | Proxy spawns client or passes FD via SCM_RIGHTS | Client has valid socket FD for internal IPC |
| BPS-009 | Client context preserved | Request from container client | Host server sees original client hostname, PID, CWD |
| BPS-010 | Proxy health check aggregation | Stop host server, query via container client | Proxy detects host failure, returns exit code 1 to client |
| BPS-011 | Upstream connection retry | Start proxy without host server available | Proxy retries connection with exponential backoff |
| BPS-012 | Container mode auto-detection | Run client in container with proxy running | Client auto-detects container mode, uses internal IPC |
| BPS-013 | Exit code 12 - proxy not running | Stop proxy, run client | Client exits with code 12 |
| BPS-014 | Exit code 13 - internal IPC error | Simulate socketpair failure | Client exits with code 13 |
| BPS-015 | Exit code 14 - socket security violation | Modify external socket permissions to 0777 | Client exits with code 14 |
| BPS-016 | Proxy forwards with proxied_by field | Send request through proxy | Request includes `client_info.proxied_by` field |
| BPS-017 | External socket environment override | Set `HV_EXTERNAL_SOCKET=/custom/path.sock`, start proxy | Proxy uses custom path |
| BPS-018 | Upstream socket environment override | Set `HV_UPSTREAM_SOCKET=/custom/host.sock`, start proxy | Proxy connects to custom upstream path |

## Integration Test Scenarios

### End-to-End: Successful Execution

1. Start server
2. Add `API_KEY` to allowlist with value `secret123`
3. Run: `hv API_KEY -- node app.js`
4. **Verify**: Health check passes (< 500ms)
5. **Verify**: Biometric prompt appears
6. **Verify**: Auth dialog shows `node app.js` command
7. **Verify**: Approve with biometric
8. **Verify**: Node executes with `API_KEY=secret123`
9. **Verify**: Exit code 0
10. **Verify**: Audit log shows approval with command

### End-to-End: Server Down Short-Circuit

1. Stop server
2. Run: `hv API_KEY -- ./deploy.sh`
3. **Verify**: Exits within 500ms
4. **Verify**: Exit code 1
5. **Verify**: Error message mentions server not responding
6. **Verify**: Command was NOT executed

### End-to-End: Allowlist Violation

1. Start server
2. Allowlist is empty or doesn't contain `SECRET_VAR`
3. Run: `hv SECRET_VAR -- ./script.sh`
4. **Verify**: Health check passes
5. **Verify**: Request denied (not_in_allowlist)
6. **Verify**: Exit code 8
7. **Verify**: Error message mentions allowlist
8. **Verify**: Command was NOT executed
9. **Verify**: Audit log shows allowlist_violation event

### End-to-End: Biometric Failure

1. Start server
2. Add `API_KEY` to allowlist
3. Run: `hv API_KEY -- ./script.sh`
4. **Verify**: Health check passes
5. **Verify**: Biometric prompt appears
6. **Verify**: Fail or cancel biometric 3 times
7. **Verify**: Request denied immediately
8. **Verify**: Exit code 9
9. **Verify**: Command was NOT executed
10. **Verify**: Audit log shows authentication_failed

### End-to-End: Command Display and Review

1. Start server
2. Add `API_KEY` to allowlist
3. Run: `hv API_KEY -- rm -rf /important/data`
4. **Verify**: Auth dialog shows full command: `rm -rf /important/data`
5. **Verify**: User can review before approving
6. **Verify**: Deny prevents destructive operation
7. **Verify**: Log shows command for audit trail

### End-to-End: Container Mode Execution

1. Start host server on macOS
2. Configure SSH forwarding for `~/Library/Application Support/HostVault/hostvault.sock`
3. In devcontainer: Create `hv-proxy` symlink and start proxy
4. **Verify**: Proxy binds external socket at `/run/hostvault/external.sock` (0700/0600)
5. **Verify**: Proxy connects upstream successfully
6. Run: `hv API_KEY -- node app.js` in container
7. **Verify**: Health check passes through proxy (< 500ms)
8. **Verify**: Biometric prompt appears on macOS host (not in container)
9. **Verify**: Auth dialog shows original container client details (hostname, PID, CWD)
10. **Verify**: Approve, secrets received in container
11. **Verify**: Node executes with `API_KEY` set
12. **Verify**: Exit code 0
13. **Verify**: Audit log shows `proxied_by` field

### End-to-End: Bootstrap Proxy Failure

1. Start host server on macOS
2. Configure SSH forwarding
3. In devcontainer: Do NOT start proxy
4. Run: `hv API_KEY -- ./script.sh` in container
5. **Verify**: Exit code 12 (bootstrap proxy not running)
6. **Verify**: Error message mentions bootstrap proxy and external socket
7. **Verify**: Command NOT executed
8. **Verify**: No connection attempt to host server

### End-to-End: Rogue Process Isolation

1. Start host server on macOS
2. Configure SSH forwarding
3. In devcontainer: Start proxy
4. **Verify**: External socket has correct permissions (0700/0600)
5. Start rogue process attempting to connect to external.sock directly
6. **Verify**: Rogue connection fails (permission denied or connection refused)
7. Run legitimate client via internal IPC
8. **Verify**: Legitimate client succeeds, gets secrets after auth
9. **Verify**: Request goes through proxy, host sees original client context

## Manual Test Checklist

### GUI Usability

- [ ] Menu bar icon appears on launch
- [ ] Dropdown menu shows all options
- [ ] "Manage Secrets" opens configuration panel
- [ ] "View Logs" opens log viewer
- [ ] "Settings" opens preferences window
- [ ] "Quit" exits application cleanly
- [ ] Configuration panel allows adding/removing secrets
- [ ] Secret values are masked in UI
- [ ] Authorization dialog shows all required info:
  - [ ] Client hostname
  - [ ] Client user
  - [ ] Client PID
  - [ ] Client CWD
  - [ ] List of requested secrets
  - [ ] Command to be executed
  - [ ] Timeout countdown
- [ ] Biometric prompt appears before auth dialog
- [ ] "Remember this client" checkbox works
- [ ] Log viewer shows events chronologically
- [ ] Log purge with date range works
- [ ] Export logs works

### Real-World SSH Usage

- [ ] Can run `hv` over SSH forwarded socket
- [ ] Biometric prompt appears on macOS host (not SSH client)
- [ ] Secrets are available in SSH session after approval
- [ ] Command execution works remotely
- [ ] Health check works over SSH

### Cross-Platform Deployment

- [ ] Client works on ARM64 Linux
- [ ] Client works on x86_64 Linux
- [ ] Client works on ARM64 macOS
- [ ] Server works on macOS 10.15+
- [ ] Touch ID works on supported hardware
- [ ] Face ID works on supported Macs
- [ ] Passcode fallback works on all systems

### Bootstrap Proxy (Container Mode)

- [ ] `hv-proxy --version` works via symlink (busybox pattern)
- [ ] Proxy starts and binds external socket exclusively
- [ ] External socket has correct permissions (0700/0600)
- [ ] External socket path is configurable via `--external-socket`
- [ ] Upstream socket path is configurable via `--upstream-socket`
- [ ] Internal IPC uses socketpair (no filesystem path)
- [ ] Client auto-detects container mode
- [ ] Client connects via internal IPC (inherited FD)
- [ ] Client context (hostname, PID, CWD) preserved through proxy
- [ ] Host server sees `proxied_by` field in request
- [ ] Proxy health check aggregates upstream status
- [ ] Proxy handles upstream disconnection gracefully (retry with backoff)
- [ ] Rogue process cannot connect to external socket directly
- [ ] Exit code 12 when proxy not running
- [ ] Exit code 13 on internal IPC failure
- [ ] Exit code 14 on socket security violation

### Complex Command Scenarios

- [ ] Command with multiple arguments: `hv KEY -- cmd arg1 arg2 arg3`
- [ ] Command with flags: `hv KEY -- curl -H "Auth: $KEY" https://api.com`
- [ ] Shell command: `hv KEY -- bash -c 'echo $KEY'`
- [ ] Pipeline: `hv KEY -- cat file | grep pattern`
- [ ] Subshell: `(hv KEY -- echo $KEY)`

---

*See also: [Protocol Tests](./PROTOCOL.md#protocol-flow), [Security Mitigations](./SECURITY.md#security-mitigations-summary), [CLI Error Handling](./CLI.md#error-handling), and [macOS App Features](./MACOS-APP.md#core-features)*

---

*Document Version: 1.0*  
*Part of the [HostVault Product Requirements](./README.md)*
