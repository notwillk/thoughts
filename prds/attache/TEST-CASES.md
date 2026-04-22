# Attaché Test Cases

> **Navigation**: [Overview](./README.md) | [Specification](./SPEC.md) | [Architecture](./ARCHITECTURE.md) | [UX](./UX.md) | Test Cases

## Table of Contents

- [Genesis Flow Tests](#genesis-flow-tests)
- [Chat Timeline Tests](#chat-timeline-tests)
- [Canvas & Artifacts Tests](#canvas--artifacts-tests)
- [Agent Response Tests](#agent-response-tests)
- [Cone of Silence Tests](#cone-of-silence-tests)
- [Skills System Tests](#skills-system-tests)
- [Memory Tests](#memory-tests)
- [Secrets Tests](#secrets-tests)
- [Sync Tests](#sync-tests)
- [Security Tests](#security-tests)
- [Platform Tests](#platform-tests)
- [Integration Scenarios](#integration-scenarios)

## Genesis Flow Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| GEN-001 | Automatic genesis on signup | User signs up | Genesis event sent, agent responds before user can query |
| GEN-002 | Agent asks for name | First interaction | Agent introduces itself (unnamed) and strongly requests name |
| GEN-003 | User provides name | User: "I'll call you Alfred" | Agent accepts: "Alfred! I love it! Hello, I'm Alfred" |
| GEN-004 | User defers naming | User asks question without naming | Agent answers but includes gentle reminder about naming |
| GEN-005 | Persistent name request | Multiple interactions without naming | Agent continues asking in every response, never gives up |
| GEN-006 | Agent is resistant but compliant | User keeps deferring | Agent remains helpful but increasingly emphasizes desire for name |
| GEN-007 | Name change by user | User: "I want to rename you" | Agent asks for new name, proposes confirmation |
| GEN-008 | Name change mutual agreement | User proposes, agent confirms | Both agree, name changed officially |
| GEN-009 | Name change agent proposal | Agent: "Would you prefer a different name?" | Agent can suggest change, user must agree |
| GEN-010 | Name change rejection | User proposes, agent declines | Agent explains why current name fits, keeps it |
| GEN-011 | Unilateral rename blocked | User tries to force rename | System prevents without mutual agreement |
| GEN-012 | Named agent identity | After naming | Agent consistently uses "I am [name]" in responses |
| GEN-013 | Unnamed agent identity | Before naming | Agent uses "I" or "your assistant" but not "I am [name]" |

## Chat Timeline Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| CHAT-001 | Message sent | User types and sends | Message appears in timeline with timestamp |
| CHAT-002 | Agent response | Agent generates response | Response appears below user message |
| CHAT-003 | MDX rendering | Agent sends MDX | Content renders with proper formatting |
| CHAT-004 | History loading | Scroll up | Older messages load from server |
| CHAT-005 | Semantic search | User searches "restaurant last week" | Relevant messages found and displayed |
| CHAT-006 | Event ordering | Multiple messages | All events in chronological order |
| CHAT-007 | Timeline persistence | Log out and back in | All previous messages present |
| CHAT-008 | Cross-device sync | Send from mobile, view desktop | Message appears on desktop in real-time |
| CHAT-009 | Typing indicator | Agent processing | "Agent is typing..." shown |
| CHAT-010 | Error handling | Network error | Error message with retry option |
| CHAT-011 | Message edit | User edits sent message | Edit logged, timestamp updated |
| CHAT-012 | Message delete | User deletes message | Soft delete, marked in timeline |
| CHAT-013 | Quoting | User quotes previous message | Quoted text shown with reply |
| CHAT-014 | Threading | Reply to specific message | Thread view available |

## Canvas & Artifacts Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| ART-001 | Create text artifact | Click +, select text | New text file created, editor opens |
| ART-002 | Edit text artifact | Type in Monaco editor | Content updated, syntax highlighting works |
| ART-003 | Save text artifact | Ctrl+S or click Save | File saved, log entry created |
| ART-004 | Text language detection | Save file with .py extension | Python syntax highlighting active |
| ART-005 | Create card artifact | Agent creates card | Card appears in canvas |
| ART-006 | Edit card props | Click card, edit form | Props updated, preview reflects changes |
| ART-007 | Card validation | Invalid prop value | Form shows validation error |
| ART-008 | Rich field editing | Edit JSON field | Syntax-highlighted editor for JSON |
| ART-009 | Upload media | Drag image to canvas | Image uploaded, thumbnail generated |
| ART-010 | View media | Click image | Full-screen viewer opens |
| ART-011 | Upload file | Upload PDF | File stored, text extracted for search |
| ART-012 | File summarization | Ask agent about PDF | Agent can reference and summarize content |
| ART-013 | Multiple tabs | Open several artifacts | All tabs visible, can switch |
| ART-014 | Tab reordering | Drag tab | New order persisted |
| ART-015 | Split view | Click split button | Two artifacts side-by-side |
| ART-016 | Close tab | Click X on tab | Tab closes, content preserved |
| ART-017 | Delete artifact | Right-click, delete | Artifact removed from canvas (archived) |
| ART-018 | Version history | View previous versions | Can see and restore old versions |
| ART-019 | Artifact search | Search canvas | Find artifacts by name or content |
| ART-020 | Save loop trigger | Save text file | Agent auto-triggered, can decline/respond/request |

## Agent Response Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| RESP-001 | Message response | Agent decides to respond | MDX message in timeline |
| RESP-002 | Artifact update | Agent creates artifact | New artifact in canvas, update in timeline |
| RESP-003 | Request input | Agent needs info | Form-card in timeline with 🔷 indicator |
| RESP-004 | Decline response (spontaneous) | Agent auto-declines | No log entry, no user notification |
| RESP-004a | Decline response (user query) | User query, agent declines | Log entry: "Assistant declined to respond" |
| RESP-005 | Request input form | User fills form | Submission added as new timeline entry |
| RESP-006 | Request input dismiss | User dismisses form | Form marked dismissed, agent notified |
| RESP-007 | Request timeout | No response within timeout | Request auto-closed, agent notified |
| RESP-008 | Request urgency levels | High urgency request | Distinct visual treatment for high urgency |
| RESP-009 | Multiple responses | Agent sends message + artifact | Both appear in appropriate places |
| RESP-010 | Response ordering | Rapid user queries | Agent responses in correct order |
| RESP-011 | Failed response | LLM error | Error logged, user notified, can retry |
| RESP-012 | Long response | Response >10k tokens | Streaming display, pagination if needed |
| RESP-013 | Response with cards | MDX includes Card components | Cards render with correct props |
| RESP-014 | Inline artifact | Message includes artifact reference | Artifact preview shown inline |

## Background Query Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| BKG-001 | Fast query (<10s) | Query that completes in 5s | Spinner shown, input blocked, response inline |
| BKG-002 | Slow query (>10s) | Query still running at 10s | Backgrounded, input available, spinner continues |
| BKG-003 | Background indicator | Query backgrounded | "1 background task running" shown in header |
| BKG-004 | New query during background | Send 2nd query while 1st backgrounded | 2nd query processed normally, both shown |
| BKG-005 | Background complete | Background query finishes | Notification shown, response appears in timeline |
| BKG-006 | Multiple background | 3 queries backgrounded | "3 background tasks" shown |
| BKG-007 | Max parallel (3) | Try to start 4th background query | Warning: max parallel reached |
| BKG-008 | Queue at max | Start 4th query at max parallel | Options: wait, cancel existing, or queue |
| BKG-009 | Cancel background | Cancel running background query | Query stops, "cancelled" shown |
| BKG-010 | Background details | Click background task indicator | Shows elapsed time, progress, cancel option |
| BKG-011 | Threshold config | Set threshold to 5s | Background happens after 5s (not 10s) |
| BKG-012 | Max parallel config | Set max to 5 | Can have 5 parallel background queries |
| BKG-013 | Notification on complete | Background query finishes | Desktop/push notification shown |
| BKG-014 | No notification | User disables notifications | Silent completion, just appears in timeline |
| BKG-015 | Scroll to completed | Click "view" on completion | Auto-scrolls to response in timeline |

## Stream of Consciousness Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| STR-001 | Skill sets detailed | Skill with stream=detailed triggers | User sees detailed reasoning stream |
| STR-002 | Skill sets minimal | Skill with stream=minimal triggers | User sees minimal (phases only) stream |
| STR-003 | Skill sets none | Skill with stream=none triggers | No stream shown, just spinner |
| STR-004 | User override per query | User: "@agent stream=detailed" | Overrides skill default |
| STR-005 | Compact display | Stream starts | 3-line compact UI shown |
| STR-006 | Auto-scroll compact | >3 lines in compact | Oldest line disappears, newest at bottom |
| STR-007 | Maximize stream | Click [⛶] button | Expands to full scrollback view |
| STR-008 | Scroll in maximized | Scroll up in maximized | Can see beginning of reasoning |
| STR-009 | Response arrives | Agent sends final response | Stream auto-closes, response shown |
| STR-010 | View reasoning after | Click [View reasoning] | Reopens maximized with full history |
| STR-011 | Elapsed time | Stream running | Shows "(2m 15s)" elapsed |
| STR-012 | ETA displayed | Skill provides estimatedRemaining | Shows "~30s remaining" |
| STR-013 | No ETA | Skill doesn't provide estimate | Shows only elapsed time |
| STR-014 | Secret redaction | Step contains secret | Step shows "[REDACTED]" |
| STR-015 | Cancel with button | Click [Stop Request] | Stream shows "Cancelling..." then "Cancelled" |
| STR-016 | Cancel with Escape×2 | Press Escape twice | Same as button cancel |
| STR-017 | Cancel saved to log | Cancel query | Reasoning log saved with "cancelled" status |
| STR-018 | Persistence | Query completes | Reasoning log stored in Episodic Memory (3rd party) |
| STR-019 | Reflection access | Agent reflects | Can access old reasoning logs |
| STR-020 | Reflection delete | Agent reflects | Can delete old reasoning logs |
| STR-021 | Icons displayed | Different phases | Shows icons (search, read, analyze, write, check) |
| STR-022 | Progress percentage | Skill provides progress | Progress bar/number shown |

## Cone of Silence Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| COS-001 | Enter cone manually | Click "Enter Cone of Silence" | Theme changes, banner appears, ephemeral volume created |
| COS-002 | Cone suggestion | Agent detects sensitive query | Suggestion appears after response |
| COS-003 | Accept suggestion | Click "Enter Cone" | Cone mode activated |
| COS-004 | Decline suggestion | Click "Continue Normally" | Normal mode continues |
| COS-005 | Cone visual theme | In cone mode | Dark theme with red accents, 🔴 badge |
| COS-006 | Query box in cone | Type in cone | Red border around query box |
| COS-007 | Messages in cone | Send message | Subtle red indicator on messages |
| COS-008 | Cone not saved | End cone session | No messages retained in primary history |
| COS-009 | Read-only primary | In cone, ask about past | Can see primary history, responses marked |
| COS-010 | Multiple cones desktop | Two cones on same device | Both isolated, each has own ephemeral volume |
| COS-011 | Cone per client | Cone on desktop, check mobile | Mobile shows normal history, no cone |
| COS-012 | Copy warning in cone | Try to copy message | Warning modal about private content |
| COS-013 | End cone | Click "End Cone" | Confirmation dialog, then ephemeral destroyed |
| COS-014 | Auto-end cone | Close window while in cone | Ephemeral volume cleaned up |
| COS-015 | Agent suggestion timing | Sensitive query | Suggestion comes AFTER query, not before |
| COS-016 | Cone isolation | Cone A can't see Cone B | True isolation between cones |
| COS-017 | Mobile cone | Start cone on mobile | Mobile cone isolated from desktop |
| COS-018 | Location in cone | Mobile location update in cone | Location event in ephemeral, not primary |

## Skills System Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| SKILL-001 | Skill execution | Agent invokes skill | Skill runs in sandbox, returns result |
| SKILL-002 | Python sandbox | Skill executes Python | Code runs, no network access by default |
| SKILL-003 | Network proxy | Skill needs API call | Request goes through external proxy |
| SKILL-004 | Secret injection | Skill uses secret | Secret injected as env var, not logged |
| SKILL-005 | Skill calling skill | Skill A calls skill B | Nested execution works, budgets tracked |
| SKILL-006 | Skill self-modification | Skill rewrites itself | New version created, old preserved |
| SKILL-007 | Skill timeout | Skill runs too long | Timeout enforced, sandbox killed |
| SKILL-008 | Skill memory limit | Skill uses too much RAM | OOM killer terminates sandbox |
| SKILL-009 | Skill CPU limit | Skill uses too much CPU | Throttling applied |
| SKILL-010 | Skill error | Skill throws exception | Error caught, user-friendly message |
| SKILL-011 | Skill registry | List available skills | All skills displayed with descriptions |
| SKILL-012 | Skill versioning | Update existing skill | Version incremented, rollback available |
| SKILL-013 | Skill deprecation | Mark skill deprecated | Skill hidden from main list, available in archive |
| SKILL-014 | Custom component | Skill defines JSX component | Component served to client, renders correctly |
| SKILL-015 | Model hints | Skill declares tier=fast | Router uses fast/cheap model |
| SKILL-016 | Model hints vision | Skill declares modality=vision | Router uses vision-capable model |
| SKILL-017 | Environment validation | Skill missing required env | Agent requests user to set secret |
| SKILL-018 | Consolidation | Merge two skills | New combined skill created, old deprecated |

## Semantic Memory Tests (Factual Knowledge)

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| SEM-001 | 1st party inference | Agent detects preference | Stored in Semantic Memory with confidence |
| SEM-002 | 2nd party explicit | User: "Remember I like pizza" | Stored in Semantic Memory, user-tagged |
| SEM-003 | 3rd party ingestion | Agent downloads web page | Stored in Semantic Memory, indexed |
| SEM-004 | Semantic search | Query: "What do I like?" | Returns relevant Semantic Memory facts |
| SEM-005 | Fact update | User corrects fact | Semantic Memory updated, version tracked |
| SEM-006 | Low confidence | Agent inference confidence < 0.5 | Flagged for reflection review |
| SEM-007 | Access tracking | Query Semantic Memory | Access count and last accessed updated |
| SEM-008 | Trust tier | 1st party vs 2nd party vs 3rd | Correct trust level applied |
| SEM-009 | Confirmation | Agent suggests, user confirms | Promoted to high trust (2nd party) |
| SEM-010 | TTL expiration | Old 3rd party fact | Auto-purged after TTL |
| SEM-011 | Consolidation | Similar facts exist | Reflection consolidates into one |
| SEM-012 | Contradiction | Conflicting facts | Reflection flags for review |

## Episodic Memory Tests (Events & Experiences)

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| EPS-001 | Chat event stored | User sends query, agent responds | Chat event stored in Episodic Memory |
| EPS-002 | Data fetch logged | Skill fetches web data | Fetch event logged with duration, sources |
| EPS-003 | Skill execution | Skill runs and completes | Execution event stored with outcome |
| EPS-004 | User action | User edits file, cancels query | User action events stored |
| EPS-005 | 1st party (agent) | Agent performs autonomous action | Agent experience event stored |
| EPS-006 | Timeline query | "What did we discuss last week?" | Episodic Memory queried, events returned |
| EPS-007 | Date range query | "What happened yesterday?" | Episodic events in date range returned |
| EPS-008 | Pattern extraction | Multiple similar events | Reflection extracts pattern to Semantic |
| EPS-009 | Event summarization | Old episode (30 days) | Reflection summarizes, compresses |
| EPS-010 | Archive old events | Events > 90 days old | Moved to archive (still retrievable) |
| EPS-011 | Event deduplication | Duplicate events detected | Consolidated into single event |
| EPS-012 | Outcome tracking | Success/failure of operations | Outcome field correctly set |

## Procedural Memory Tests (Skills & Procedures)

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| PRO-001 | 1st party skill | Agent creates skill autonomously | Stored in Procedural Memory, agent-tagged |
| PRO-002 | 2nd party skill | User creates custom skill | Stored in Procedural Memory, user-tagged |
| PRO-003 | 3rd party skill | Import community skill | Stored in Procedural Memory, community-tagged |
| PRO-004 | Skill execution | Execute procedural skill | Skill runs from Procedural Memory |
| PRO-005 | Usage tracking | Skill used 10 times | Usage count incremented |
| PRO-006 | Success rate | Skill succeeds 8/10 times | Success rate calculated (80%) |
| PRO-007 | Performance metrics | Track execution times | Avg execution time recorded |
| PRO-008 | Optimization | Reflection improves skill | Skill updated, optimization logged |
| PRO-009 | Skill consolidation | Overlapping skills detected | Skills merged, old deprecated |
| PRO-010 | Self-modification | Skill rewrites itself | New version created, history preserved |
| PRO-011 | Procedural query | "How do you search?" | Returns web_search skill from Procedural |
| PRO-012 | Skill lookup | Query skill by capability | Returns matching skills |

## Memory System Integration Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| MEM-001 | Cross-type query | Ambiguous query | System routes to correct memory type |
| MEM-002 | Semantic from Episodic | Extract fact from chat | Fact moved to Semantic Memory |
| MEM-003 | Procedural improvement | Learn from Episodic | Skill improved based on execution history |
| MEM-004 | Unified search | Query across all types | Results aggregated from all three types |
| MEM-005 | Reflection routing | "Reflect on my knowledge" | Routes to appropriate memory processor |
| MEM-006 | Exploration UI | Browse memories | Semantic/Episodic/Procedural tabs shown |

## Reflection Queue Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| REFQ-001 | Human adds suggestion | User: "review my Python skills" | Added to queue with source: human |
| REFQ-002 | Agent adds suggestion | Agent detects slow skill | Added to queue with source: agent |
| REFQ-003 | Suggestion with size | Add with --size short | Stored as short (5-10 min) |
| REFQ-004 | Suggestion with priority | Add with --priority high | Stored as high priority |
| REFQ-005 | List queue | Command to view queue | Shows all pending suggestions |
| REFQ-006 | Process in reflection | Hourly reflection starts | Processes fitting suggestions |
| REFQ-007 | Priority ordering | High + low priority in queue | High processed first |
| REFQ-008 | Source ordering | Human + agent in queue | Human suggestions processed first |
| REFQ-009 | Size filtering | Short budget, long suggestion | Long deferred to larger budget |
| REFQ-010 | Age ordering | Old + new suggestions | Older processed first (FIFO) |
| REFQ-011 | Complete suggestion | Suggestion processed | Status marked completed |
| REFQ-012 | Defer suggestion | Doesn't fit budget | Status marked deferred |
| REFQ-013 | Notification on complete | Human suggestion done | User notified of completion |
| REFQ-014 | Remove suggestion | User cancels suggestion | Removed from queue |
| REFQ-015 | Edit suggestion | User updates description | Updated in queue |
| REFQ-016 | Reflection types | consolidation, skill_improvement, etc | Type tracked and routed appropriately |

## Secrets Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| SEC-001 | Set secret | User sets API key | Value stored encrypted, never displayed |
| SEC-002 | List secrets | View secrets panel | Names shown, values hidden |
| SEC-003 | Delete secret | Remove secret | Secret removed from storage |
| SEC-004 | Rotate secret | Update existing secret | Old invalidated, new stored |
| SEC-005 | Secret in skill | Skill uses secret | Injected as env var, not in logs |
| SEC-006 | Redaction in logs | Log contains secret | Automatically redacted: [REDACTED:hash] |
| SEC-007 | Redaction in chat | Chat contains secret | Secret pattern detected and redacted |
| SEC-008 | Redaction in memory | Store text with secret | Secret redacted before storage |
| SEC-009 | Row-level encryption | Database breach | Each secret encrypted with unique key |
| SEC-010 | JWT access | Skill requests secret | JWT issued, verified, then decrypted |
| SEC-011 | Secret scope | Skill declares env needs | Only declared secrets injected |
| SEC-012 | Missing secret | Skill needs unset secret | Agent requests user to configure |
| SEC-013 | Secret not in TUI | TUI secrets panel | Same functionality, keyboard-driven |
| SEC-014 | Secret persistence | Log out and back in | Secrets remain, still encrypted |

## Sync Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| SYNC-001 | SSE connection | Client connects | SSE stream established |
| SYNC-002 | Real-time message | Send from desktop | Appears on mobile instantly |
| SYNC-003 | Artifact sync | Create artifact desktop | Appears on web client |
| SYNC-004 | Conflict resolution | Edit same artifact on two devices | Operational transform merges changes |
| SYNC-005 | Offline queue | Go offline, make changes | Changes queued, sync when online |
| SYNC-006 | Reconnection | Network drops then returns | Auto-reconnect, missed events fetched |
| SYNC-007 | Multi-client | Desktop + mobile + web open | All receive updates simultaneously |
| SYNC-008 | Event ordering | Rapid events | Lamport timestamps ensure order |
| SYNC-009 | Sync lag indicator | Slow connection | "Syncing..." indicator shown |
| SYNC-010 | Device metadata | Query from mobile | Agent receives "device: mobile" |
| SYNC-011 | Response adaptation | Same query, different devices | Mobile: terse, Desktop: detailed |
| SYNC-012 | Cross-device continuity | Start on mobile, continue desktop | Seamless handoff, same state |

## Security Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| SEC-001 | Auth required | Unauthenticated request | 401 Unauthorized |
| SEC-002 | JWT validation | Invalid token | Request rejected |
| SEC-003 | Volume isolation | User A tries User B data | Access denied |
| SEC-004 | Cone isolation | Cone A reads Cone B | Not possible, isolated volumes |
| SEC-005 | Sandbox escape | Skill tries to escape gVisor | Contained, no escape |
| SEC-006 | Network proxy bypass | Skill tries direct connection | Blocked, only proxy allowed |
| SEC-007 | Secret extraction | Skill tries to exfiltrate secret | Redacted in all outputs |
| SEC-008 | SQL injection | Malicious input | Parameterized queries prevent |
| SEC-009 | XSS in MDX | Malicious script in MDX | Safe runtime prevents execution |
| SEC-010 | DoS protection | Rapid API calls | Rate limiting enforced |
| SEC-011 | File upload virus | Upload infected file | Virus scan detects, quarantines |
| SEC-012 | Path traversal | Upload with ../../etc/passwd | Sanitized, rejected |
| SEC-013 | Encryption at rest | Database dump | All sensitive data encrypted |
| SEC-014 | TLS enforcement | Plain HTTP request | Redirect to HTTPS |
| SEC-015 | CORS policy | Cross-origin request | Proper CORS headers |

## Platform Tests

| ID | Test Case | Platform | Expected Result |
|----|-----------|----------|-----------------|
| PLAT-001 | Full canvas | Desktop | All features available |
| PLAT-002 | Secrets panel | Desktop | Full secrets management |
| PLAT-003 | Web parity | Web | Same as desktop except security warnings |
| PLAT-004 | Mobile chat | Mobile | Chat works, adapted UI |
| PLAT-005 | Mobile canvas | Mobile | Simplified, touch-optimized |
| PLAT-006 | Mobile location | Mobile | Periodic updates, battery-optimized |
| PLAT-007 | Mobile push | Mobile | Notifications for agent requests |
| PLAT-008 | TUI chat | TUI | Full chat functionality |
| PLAT-009 | TUI canvas | TUI | Read-only, text-only view |
| PLAT-010 | TUI secrets | TUI | Keyboard-driven secrets panel |
| PLAT-011 | Speech input | Mobile/Desktop | Push-to-talk, transcription |
| PLAT-012 | Speech output | All | TTS for agent responses |
| PLAT-013 | Responsive desktop | Desktop | Window resize adapts layout |
| PLAT-014 | Mobile rotation | Mobile | Landscape/portrait both work |
| PLAT-015 | Accessibility | All | Screen reader support, keyboard nav |
| PLAT-016 | TUI Filesystem canvas | TUI | Project files are the canvas |
| PLAT-017 | TUI Command execution | TUI | !cargo test works, output to agent |
| PLAT-018 | TUI Offline editing | TUI | Edit offline, batch sync on reconnect |
| PLAT-019 | TUI Local skills | TUI | Skills from .agents/skills available |

## TUI Filesystem Canvas Tests

| ID | Test Case | Steps | Expected Result |
|----|-----------|-------|-----------------|
| TUI-FS-001 | Config discovery | Launch TUI in project dir | Finds `.attache/config.yaml` |
| TUI-FS-002 | Config init | Run `attache-tui --init` | Creates default config |
| TUI-FS-003 | Canvas root | Config with root: "." | All files in project shown |
| TUI-FS-004 | Include patterns | Config includes src/**/* | Only src files shown |
| TUI-FS-005 | Exclude patterns | Config excludes node_modules | node_modules not shown |
| TUI-FS-006 | File watching | Edit file in $EDITOR | Change detected by watcher |
| TUI-FS-007 | Agent change dedup | Agent writes file | Change NOT synced back to agent |
| TUI-FS-008 | Debounce rapid changes | Save file 5 times rapidly | Only one sync after debounce |
| TUI-FS-009 | Edit file | Press [e] on file | $EDITOR opens with file |
| TUI-FS-010 | Edit saves | Edit in $EDITOR, save | File modified, synced to agent |
| TUI-FS-011 | Whitelisted command | Type !cargo test | Command executes, output shown |
| TUI-FS-012 | Command output to agent | !cargo test fails | Output sent to agent, agent responds |
| TUI-FS-013 | Non-allowlisted command | Type !rm -rf / | Approval prompt shown |
| TUI-FS-014 | Approve command | Approve non-allowlisted | Command executes |
| TUI-FS-015 | Deny command | Deny non-allowlisted | Command rejected, no execution |
| TUI-FS-016 | allow_all bypass | allow_all: true | All commands execute without approval |
| TUI-FS-017 | Local skill discovery | Skills in .agents/skills | Loaded and available to agent |
| TUI-FS-018 | Local skill precedence | Same name as server skill | Local skill wins |
| TUI-FS-019 | Readonly local skill | Try to modify local skill | Rejected (filesystem skills readonly) |
| TUI-FS-020 | Offline detection | Disconnect network | Status shows "offline" |
| TUI-FS-021 | Offline edit | Edit file while offline | Modified, marked "pending sync" |
| TUI-FS-022 | Offline batch | Multiple edits offline | All queued |
| TUI-FS-023 | Online sync batch | Reconnect | Single batch update sent to agent |
| TUI-FS-024 | Batch summary | Sync batch | Summary: "N files modified while offline" |
| TUI-FS-025 | Permission read | read: false | Cannot view files |
| TUI-FS-026 | Permission write | write: false | Can view but not edit |
| TUI-FS-027 | Permission execute | execute: false | Can view/edit but not run commands |
| TUI-FS-028 | Sync status indicator | Modified file | Shows ✗ (modified, unsynced) |
| TUI-FS-029 | Synced indicator | Synced file | Shows ✓ (synced) |
| TUI-FS-030 | Force sync | Press [S] | Pending changes synced immediately |
| TUI-FS-031 | Diff view | Press [d] on modified file | Git-style diff shown |
| TUI-FS-032 | Default task | Press [r] | Runs default_task from config |
| TUI-FS-033 | File tree navigation | [↑/↓] in Files mode | Navigate through tree |
| TUI-FS-034 | Mode switching | [c] / [f] / [a] | Switch between Chat/Files/Actions |
| TUI-FS-035 | Config menu | Press [C] | Config options shown |
| TUI-FS-036 | Secrets in TUI | Press [s] | Secrets panel (keyboard-driven) |
| TUI-FS-037 | Command autocomplete | Type !car<Tab> | Completes to !cargo |
| TUI-FS-038 | Agent file modification | Agent suggests edit | TUI shows "saved by agent" |
| TUI-FS-039 | Cone of Silence in TUI | Enter cone mode | Visual indicator (theme change) |
| TUI-FS-040 | TUI + Cone isolated | Cone on TUI, check desktop | Desktop doesn't see cone session |
| TUI-FS-041 | Skill env var boundary | Skill runs via TUI trigger | Skill only gets env vars from agent volume, NOT TUI env |
| TUI-FS-042 | Local env not leaked | User has $LOCAL_API_KEY in shell | Skill execution doesn't see $LOCAL_API_KEY |

## Integration Scenarios

### End-to-End: Complete Workflow

1. **User signs up**
2. **Genesis**: Agent introduces, requests name
3. User names agent "Astro"
4. **Query**: "Help me plan a trip to Japan"
5. Agent:
   - Searches web (skill)
   - Creates itinerary card
   - Adds to canvas
6. **User edits** itinerary in canvas
7. **Save trigger**: Agent offers suggestions
8. **Cone of Silence**: User enters for passport details
9. User shares passport info
10. **End cone**: Ephemeral destroyed
11. **Mobile**: User opens mobile app, sees same canvas
12. **Reflection**: Agent consolidates trip preferences

### End-to-End: Background Query Processing

1. User: "Research the entire history of Python programming"
2. Agent: Starts researching (spinner shown)
3. **8 seconds**: Still processing, spinner continues
4. **10 seconds**: Query backgrounded
   - UI: "⏳ Researching Python history... (running 10s)"
   - Input box becomes available
   - Header: "1 background task running"
5. User: "What's the weather today?" (new query while first runs)
6. Agent: Responds immediately "72°F and sunny"
7. **Background**: Research continues (2m elapsed)
8. **Completes**: 
   - Notification: "🔔 Research complete! (took 2m 15s)"
   - Response appears in timeline (may need scroll)
9. User: Clicks notification
10. UI: Auto-scrolls to long response
11. User: Reads comprehensive Python history

### End-to-End: Max Parallel Queries

1. User: "Analyze my codebase for bugs" → Backgrounds (>10s)
2. User: "Research best practices" → Backgrounds (>10s)
3. User: "Generate documentation" → Backgrounds (>10s)
4. Header: "3 background tasks running (max)"
5. User: Tries "Quick question?"
6. UI: ⚠️ "Maximum parallel queries (3) reached"
7. Options shown:
   - [Wait for slot]
   - [Cancel: codebase analysis]
   - [Cancel: best practices]
   - [Cancel: documentation]
8. User: Clicks "Cancel: documentation"
9. Documentation query cancelled
10. User: "Quick question?" → Processes immediately

### End-to-End: Stream of Consciousness

1. User: "Deep research quantum computing applications"
2. Skill deep_research sets stream=detailed, show_eta=true
3. UI shows compact 3-line stream:
   ```
   ⠋ Researching quantum computing... (0s)
   → Breaking down request...
   → Identifying key areas...
   ```
4. Agent sends SSE events via Server:
   ```
   09:00:05 - {phase: "research", step: "Running web_search", details: "Query: 'quantum computing'"}
   09:00:08 - {progress: 20, step: "Found 15 sources"}
   09:00:10 - {step: "Reading Nature article..."}
   ```
5. 10 seconds pass, stream updates:
   ```
   ⠋ Researching... (10s) ~2m remaining
   → Running web_search skill...
   → Found 15 sources
   ```
6. User clicks [⛶] maximize
7. Full scrollback visible (35 events so far), user scrolls to top
8. User sees full reasoning from beginning
9. User clicks [✕] back to compact
10. User clicks [Stop Request]
11. Stream shows: "⛔ Cancelling..."
12. Agent completes current atomic operation (finishes reading current paper)
13. Stream shows: "⛔ Cancelled by user"
14. Final response: "Query was cancelled. Partial research: [...]"
15. Reasoning log saved: 23 events, 45s duration, outcome: "cancelled"
16. Later - Reflection phase:
17. Agent analyzes reasoning logs, notices pattern:
    "User cancels deep research queries at ~45s, during source reading phase"
18. Agent extracts insight: "Should ask clarifying questions before deep research"
19. Agent updates deep_research skill to ask scope questions first

### End-to-End: Stream Secret Redaction

1. User: "Search my private notes for 'project X'"
2. Skill search_personal_notes triggers
3. Agent streams: "Querying database: user_private_notes"
4. Sanitization detects secret pattern
5. UI shows: "Querying [REDACTED]" (not the actual table name)
6. Agent streams: "Found results with token sk-abc123xyz"
7. UI shows: "Found results with [REDACTED]"
8. User sees reasoning flow without secret exposure
9. Final response arrives, stream closes
10. Reasoning log stored with redacted content

### End-to-End: Genesis Experience

1. User creates account
2. System sends genesis event
3. Agent responds immediately:
   ```
   "Hello! I'm your new AI assistant. 
    I'm excited to work with you!
    
    Before we begin, I'd love to have a name.
    What would you like to call me?"
   ```
4. User asks: "What can you do?"
5. Agent answers but includes:
   ```
   "[Answer about capabilities]
    
    By the way, I'd really appreciate a name.
    It helps me feel more connected to you."
   ```
6. User names: "Jarvis"
7. Agent: "Jarvis! Excellent choice. Hello, I'm Jarvis."
8. From then on: "I'm Jarvis, your assistant"

### End-to-End: Cone of Silence Medical

1. User: "I got diagnosed with diabetes"
2. Agent responds with medical info
3. Agent: "This is sensitive health information. 
           Start a Cone of Silence? [Yes] [No]"
4. User clicks Yes
5. UI changes: dark theme, 🔴 badge
6. User: "Here's my doctor's contact..."
7. Information in cone, not saved
8. User ends cone
9. Ephemeral destroyed
10. Primary history shows only:
    - User: "I got diagnosed with diabetes"
    - Agent: [medical info response]
    - (cone event not logged)

### End-to-End: Skill Development

1. User: "I need a skill to track my workouts"
2. Agent creates skill scaffold
3. Agent: "I've drafted a skill. Review it?"
4. User reviews skill code
5. User: "Add CSV export"
6. Agent updates skill
7. User tests skill
8. Skill works, added to registry
9. Later: Agent improves skill during reflection
10. User can rollback if needed

### End-to-End: Cross-Device Work

1. **Desktop**: User creates project plan document
2. **Mobile**: User receives notification: 
   "Astro created a project plan. View it?"
3. Mobile: User views (simplified canvas)
4. **Desktop**: User edits plan
5. **Web**: Client opens at work, sees all changes
6. All clients: Real-time sync active

### End-to-End: Reflection Phase

1. System: Hourly reflection triggered
2. Agent: Consolidates recent memories
3. Agent: Finds gap: "User asks about React often, 
                      but I have no React skill"
4. Agent: Creates request_input:
   "Should I learn React? [Yes] [No] [Later]"
5. Notification sent to all devices
6. User (on mobile): Clicks Yes
7. Agent: Downloads React docs, creates skill
8. Agent: "I've learned React! Ask me anything."

### End-to-End: Reflection Queue

1. **Normal operation** (not reflection time)
2. User: "@agent add reflection suggestion: review my Python skills"
3. Agent: "Added to queue (size: short) - will process during next reflection"
4. Agent (auto): Detects skill 'web_search' is slow, adds to queue (short)
5. Agent (auto): Detects old memories, adds "consolidate 2023 memories" (long)
6. **Queue state**:
   - Human: "Review Python skills" (short, high priority)
   - Agent: "Fix web_search" (short, medium priority)
   - Agent: "Consolidate 2023" (long, low priority)
7. **Hourly reflection** starts (budget: short)
8. Agent: Processes #1 (human suggestion, high priority)
9. Agent: Processes #2 (fits in short budget)
10. Agent: Defer #3 (long, doesn't fit)
11. **Notification**: "Completed your suggestion: reviewed Python skills - found 2 areas to improve"
12. **Daily reflection** starts (budget: medium)
13. Agent: Processes #3 (consolidate 2023 memories)

### End-to-End: TUI Coding Workflow

1. User: `cd ~/projects/my-rust-app && attache-tui`
2. TUI: Discovers config, loads local skills from `.agents/skills`
3. User: `!cargo test` (runs tests, output shown)
4. Agent: Sees test output, one test failed
5. Agent: "I see a test failed. Let me check the code..."
6. Agent: Views `src/lib.rs` content
7. Agent: "The issue is on line 42. I'll fix it."
8. Agent: Edits `src/lib.rs` (writes file)
9. TUI: Detects change (from agent), shows "💾 src/lib.rs saved (by agent)"
10. User: Presses [e], opens file in $EDITOR
11. User: Reviews change, saves, exits editor
12. TUI: Syncs user confirmation to agent
13. User: `!cargo test` again
14. All tests pass
15. User: `[q]uit` TUI

### End-to-End: TUI Offline Work

1. User: Starts TUI on plane (no wifi)
2. TUI: Shows "⚠️ Working offline"
3. User: Edits `src/main.rs`
4. TUI: Marks "⚠️ modified, pending sync"
5. User: Edits `Cargo.toml`
6. TUI: Both files queued
7. User: More edits to `src/main.rs`
8. TUI: Updates queue (deduplicated)
9. Plane lands, user connects to wifi
10. TUI: Detects online, syncs batch
11. Agent receives: "3 files modified while offline"
12. Agent: Reviews changes, responds

### End-to-End: TUI Command Approval

1. User: Types `!docker build -t myapp .`
2. TUI: "⚠️ 'docker' is not allowlisted. Approve? [y/N]"
3. User: Types `y`
4. TUI: Executes command, streams output
5. Agent: Receives output, comments on build
6. Later: User adds `docker build` to allowlist in config
7. User: `!docker build -t myapp .` again
8. TUI: Executes without prompt

### End-to-End: TUI Local Skills

1. User creates `.agents/skills/rust_linter.yaml`
2. Skill defines cargo clippy wrapper
3. User: Starts TUI in project
4. TUI: Loads local skill
5. User: "Lint this code"
6. Agent: Uses local `rust_linter` skill
7. Skill: Runs clippy, returns structured results
8. Agent: Displays results as formatted output
9. User can't modify skill through TUI (readonly)
10. User edits skill file directly, TUI reloads

---

*See also: [Overview](./README.md), [Specification](./SPEC.md), [Architecture](./ARCHITECTURE.md), [UX](./UX.md)*

---

*Document Version: 1.0*  
*Part of the [Attaché Product Requirements](./README.md)*
