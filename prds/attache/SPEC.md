# Attaché Specification

> **Navigation**: [Overview](./README.md) | [Specification](./SPEC.md) | [Architecture](./ARCHITECTURE.md) | [UX](./UX.md) | [Test Cases](./TEST-CASES.md)

## Table of Contents

- [Artifacts](#artifacts)
- [TUI Configuration](#tui-configuration)
- [Filesystem Canvas](#filesystem-canvas)
- [MDX/JSX Runtime](#mdxjsx-runtime)
- [Agent Response Types](#agent-response-types)
- [Skills System](#skills-system)
- [Memory Model](#memory-model)
- [Secrets System](#secrets-system)
- [Cone of Silence](#cone-of-silence)
- [Reflection System](#reflection-system)
- [Storage Architecture](#storage-architecture)

## Artifacts

### Artifact Types

#### Text Artifacts
```typescript
interface TextArtifact {
  id: string;
  type: "text";
  path: string;           // Virtual path in workspace
  content: string;
  language?: string;      // For syntax highlighting
  version: number;        // For change tracking
  lastModified: Date;
  modifiedBy: "human" | "agent";
}
```

**Features**:
- Full text editor with syntax highlighting
- Monaco/CodeMirror integration
- Language detection from file extension
- Line numbers, search, find/replace
- Auto-save with debounce
- Diff view for agent-suggested changes

**Supported Languages**:
- markdown, json, yaml, toml
- javascript, typescript, python, rust, go
- html, css, sql, shell
- Custom language definitions

#### Card Artifacts
```typescript
interface CardArtifact {
  id: string;
  type: "card";
  component: string;      // Component name from skill
  props: Record<string, any>;
  skillId: string;        // Source skill
  version: number;
  editable: boolean;      // Can user edit props?
}
```

**Card Types**:
- **FormCard**: Input fields, buttons, actions
- **DisplayCard**: Read-only data display
- **ListCard**: Sortable, filterable lists
- **ChartCard**: Data visualization
- **MediaCard**: Image/video with metadata
- **Custom**: Skill-defined components

**Editing**:
- Form-based UI for props
- Rich fields (JSON) use syntax-highlighted editor
- Validation via JSON Schema
- Real-time preview

#### Media Artifacts
```typescript
interface MediaArtifact {
  id: string;
  type: "media";
  url: string;            // Local or remote
  mimeType: string;
  metadata: {
    width?: number;
    height?: number;
    duration?: number;
    size: number;
  };
  thumbnail?: string;
}
```

**Supported Formats**:
- Images: jpeg, png, gif, webp, svg, heic
- Video: mp4, webm, mov
- Audio: mp3, wav, ogg, m4a

**Features**:
- Lazy loading
- Thumbnail generation
- Full-screen view
- Zoom/pan for images
- Playback controls for video/audio

#### File Artifacts
```typescript
interface FileArtifact {
  id: string;
  type: "file";
  name: string;
  mimeType: string;
  storageRef: string;     // Reference to stored blob
  size: number;
  uploadedAt: Date;
  extractedText?: string; // For searchable PDFs/docs
}
```

**Supported Uploads**:
- PDFs
- Office documents (docx, xlsx, pptx)
- Text files
- Archives (zip, tar.gz) - extracted automatically

**Processing**:
- Text extraction for indexing
- Thumbnail generation where possible
- Virus scanning on upload

### Artifact Operations

```typescript
interface ArtifactOperations {
  // Create
  createText(path: string, content: string, language?: string): TextArtifact;
  createCard(component: string, props: object): CardArtifact;
  uploadFile(file: File): Promise<FileArtifact>;
  
  // Read
  getArtifact(id: string): Artifact;
  listArtifacts(): Artifact[];
  
  // Update
  updateText(id: string, content: string): TextArtifact;
  updateCardProps(id: string, props: object): CardArtifact;
  
  // Delete
  deleteArtifact(id: string): void;
  
  // Canvas operations
  addToCanvas(artifact: Artifact, tab?: string): void;
  removeFromCanvas(id: string): void;
  reorderTabs(order: string[]): void;
}
```

### Canvas Save Loop

When user edits an artifact and hits save:

1. **Artifact Update**:
   - Content written to storage
   - Version incremented
   - `lastModified` and `modifiedBy` updated

2. **Log Entry**:
   - "Artifact saved: `<path>`" added to chat timeline
   - Multiple consecutive saves collapsed into one entry if no other history in between

3. **Agent Trigger**:
   - Automatic request sent to agent
   - Agent can:
      - **Decline**: No action, no log entry (spontaneous decline)
      - **Respond**: Message or artifact update added to timeline
      - **Request Input**: Form-card added to timeline

## TUI Configuration

TUI uses a local configuration file (`.attache/config.yaml`) to define the filesystem canvas, permissions, and command allowlist.

### Config File Location

Configuration is discovered in this priority order:
1. `--project <path>` flag (explicit directory)
2. Current working directory (where `attache-tui` is launched)
3. Parent directories (stop at `$HOME`)

### Config Schema

```yaml
version: "1.0"  # Required: config format version

# Canvas filesystem configuration
canvas:
  root: "."              # Root directory (relative to config file)
  watch: true            # Enable filesystem watching (inotify/kqueue)
  
  # File inclusion patterns (glob)
  include:
    - "src/**/*"
    - "tests/**/*"
    - "*.md"
    - "*.rs"
    - "*.py"
    - "*.js"
    - "*.ts"
    - "Cargo.toml"
    - "package.json"
  
  # File exclusion patterns
  exclude:
    - "node_modules/**"
    - ".git/**"
    - "target/**"
    - "__pycache__/**"
    - "*.log"
    - "*.tmp"
  
  # File permissions
  permissions:
    read: true           # Can read files
    write: true          # Can write/modify files
    execute: true        # Can execute commands

# Command allowlist (when execute: true)
commands:
  # Allowlisted commands (no approval needed)
  allowlist:
    - "cargo test"
    - "cargo build"
    - "cargo run"
    - "cargo check"
    - "cargo clippy"
    - "npm test"
    - "npm run build"
    - "npm install"
    - "python -m pytest"
    - "python"
    - "python3"
    - "rustc"
    - "git status"
    - "git diff"
    - "git log"
    - "git add"
    - "git commit"
    - "ls"
    - "ll"
    - "cat"
    - "echo"
    - "pwd"
  
  # Commands outside allowlist require approval
  require_approval: true
  
  # DANGEROUS: Disable allowlist (allow all commands)
  allow_all: false

# Default task (run with 'r' key)
default_task: "cargo test"

# Skills discovery from filesystem (agentskills.io format)
# Searches for SKILL.md files in subdirectories: .agents/skills/<name>/SKILL.md
skills:
  # Search paths (relative to config file)
  locations:
    - ".agents/skills"
    - "../.agents/skills"
    # - "~/.config/attache/skills"  # Global skills (optional)
  
  # Filesystem skills override server skills for this instance
  local_precedence: true

# Sync behavior
sync:
  # Batch changes when offline
  offline_batching: true
  
  # Maximum batch window (seconds)
  batch_window: 30
  
  # Auto-sync on file save
  auto_sync: true
  
  # Sync debounce (ms)
  debounce_ms: 500

# Editor configuration
editor:
  # Override $EDITOR
  # command: "vim"
  
  # Wait for editor close before continuing
  wait: true
  
  # Terminal editor detection
  prefer_terminal: true  # Use terminal editor vs GUI editor
```

### Config Initialization

```bash
# Create default config in current directory
attache-tui --init

# Creates .attache/config.yaml with sensible defaults for detected project type
```

## Filesystem Canvas

TUI uses the local filesystem as the canvas, rather than virtual artifacts. This enables local-first coding workflows.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     FILESYSTEM CANVAS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  .attache/config.yaml        ← Configuration                    │
│                                                                 │
│  Project Files (Canvas):                                       │
│  ├── src/                    ← Source code                    │
│  │   ├── main.rs                                             │
│  │   └── lib.rs                                              │
│  ├── tests/                  ← Tests                          │
│  ├── Cargo.toml              ← Config                         │
│  └── README.md               ← Documentation                  │
│                                                                 │
│  .agents/                    ← Local skills (agentskills.io)   │
│  └── skills/                                                  │
│      ├── rust-linter/                                        │
│      │   ├── SKILL.md                                        │
│      │   └── scripts/                                        │
│      │       └── lint.py                                     │
│      └── test-runner/                                        │
│          ├── SKILL.md                                        │
│          └── scripts/                                        │
│              └── run_tests.py                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### File Watching

TUI watches the filesystem for changes:

```rust
pub struct FilesystemWatcher {
    watcher: RecommendedWatcher,
    debounce: Duration,
    agent_changes: Arc<RwLock<HashSet<PathBuf>>>,
}

impl FilesystemWatcher {
    /// Start watching configured paths
    pub fn watch(&mut self, config: &CanvasConfig) -> Result<(), WatcherError>;
    
    /// Handle filesystem event
    pub fn on_event(&self, event: DebouncedEvent) -> Option<FileChange>;
    
    /// Mark change as coming from agent (to ignore)
    pub fn mark_agent_change(&self, path: &Path);
    
    /// Check if change is from agent
    pub fn is_agent_change(&self, path: &Path) -> bool;
}
```

**Change Detection**:
- Uses `notify` crate (inotify on Linux, kqueue on macOS, ReadDirectoryChanges on Windows)
- Debounce rapid successive changes (500ms)
- Track changes made by agent to prevent echo loops
- Only sync changes that pass include/exclude filters

### Change Deduplication

Prevent infinite loops when agent modifies files:

```rust
pub struct ChangeTracker {
    // Changes made by agent (ignore these in watcher)
    agent_changes: Arc<RwLock<HashMap<PathBuf, SystemTime>>>,
    
    // Recent changes we've processed (prevent duplicates)
    recent_changes: Arc<RwLock<HashMap<PathBuf, SystemTime>>>,
    
    // Debounce timer
    debounce: Duration,
}

impl ChangeTracker {
    /// Called before agent writes file
    pub fn pre_agent_write(&self, path: &Path) {
        self.agent_changes.write().insert(path.to_owned(), SystemTime::now());
    }
    
    /// Called when watcher detects change
    pub fn should_process(&self, path: &Path) -> bool {
        // Check if this is an agent change
        if let Some(time) = self.agent_changes.read().get(path) {
            if time.elapsed().unwrap_or(Duration::MAX) < Duration::from_secs(5) {
                return false;  // Ignore - it's an agent change
            }
        }
        
        // Check for recent duplicate
        if let Some(time) = self.recent_changes.read().get(path) {
            if time.elapsed().unwrap_or(Duration::MAX) < self.debounce {
                return false;  // Debounce
            }
        }
        
        true
    }
    
    /// Mark change as processed
    pub fn mark_processed(&self, path: &Path) {
        self.recent_changes.write().insert(path.to_owned(), SystemTime::now());
        // Clean up old agent change markers
        self.agent_changes.write().retain(|_, time| {
            time.elapsed().unwrap_or(Duration::MAX) < Duration::from_secs(30)
        });
    }
}
```

### Command Execution

TUI can execute shell commands and send output to agent:

```rust
pub struct CommandExecutor {
    allowlist: Vec<String>,
    require_approval: bool,
    allow_all: bool,
    working_dir: PathBuf,
}

impl CommandExecutor {
    pub async fn execute(&self, cmd: &str) -> Result<CommandResult, ExecutionError> {
        // Check allowlist
        if !self.allow_all && !self.is_allowlisted(cmd) {
            if self.require_approval {
                // Prompt user for approval
                let approved = self.prompt_approval(cmd).await?;
                if !approved {
                    return Err(ExecutionError::NotApproved);
                }
            } else {
                return Err(ExecutionError::NotAllowlisted);
            }
        }
        
        // Execute
        let output = tokio::process::Command::new("sh")
            .arg("-c")
            .arg(cmd)
            .current_dir(&self.working_dir)
            .output()
            .await?;
        
        Ok(CommandResult {
            command: cmd.to_string(),
            stdout: String::from_utf8_lossy(&output.stdout).to_string(),
            stderr: String::from_utf8_lossy(&output.stderr).to_string(),
            exit_code: output.status.code().unwrap_or(-1),
            duration: Duration::from_millis(0),  // Track actual duration
        })
    }
    
    /// Send command output to agent
    pub async fn send_to_agent(&self, result: &CommandResult, agent: &AgentClient) {
        let message = format!(
            "Command executed: `{}`\n\nExit code: {}\n\n**Output:**\n```\n{}\n```",
            result.command,
            result.exit_code,
            result.stdout
        );
        
        agent.send_message(message).await;
    }
}
```

**Command Syntax in TUI**:
```
> !cargo test                    # Run tests
> !npm run build                 # Build project
> !python -m pytest              # Run Python tests
> !git status                    # Check git status
> !ls -la                        # List files
```

### Offline Support

Batch changes when offline, sync when reconnected:

```rust
pub struct OfflineManager {
    state: ConnectionState,
    pending_batch: Vec<FileChange>,
    batch_timer: Option<Instant>,
    batch_window: Duration,
}

impl OfflineManager {
    /// Detect connectivity
    pub async fn check_connection(&mut self) -> ConnectionState {
        // Ping server
        match self.ping_server().await {
            Ok(_) => {
                if self.state == ConnectionState::Offline {
                    // Just came back online
                    self.sync_batch().await;
                }
                self.state = ConnectionState::Online;
            }
            Err(_) => {
                self.state = ConnectionState::Offline;
            }
        }
        self.state
    }
    
    /// Queue change when offline
    pub fn queue_change(&mut self, change: FileChange) {
        // Check if similar change already queued
        let existing = self.pending_batch.iter_mut()
            .find(|c| c.path == change.path);
        
        if let Some(existing) = existing {
            // Update existing (newer timestamp)
            *existing = change;
        } else {
            self.pending_batch.push(change);
        }
        
        self.batch_timer.get_or_insert_with(Instant::now);
    }
    
    /// Sync batch when online
    pub async fn sync_batch(&mut self, agent: &AgentClient) -> Result<(), SyncError> {
        if self.pending_batch.is_empty() {
            return Ok(());
        }
        
        // Take all pending changes
        let batch = std::mem::take(&mut self.pending_batch);
        self.batch_timer = None;
        
        // Send as single batch update
        let update = BatchUpdate {
            timestamp: Utc::now(),
            changes: batch,
            summary: format!("{} files modified while offline", batch.len()),
        };
        
        agent.send_batch_update(update).await?;
        Ok(())
    }
}
```

**Batch Update Message**:
```json
{
  "type": "filesystem_batch_update",
  "timestamp": "2024-01-15T10:30:00Z",
  "changes": [
    {
      "path": "src/main.rs",
      "type": "modified",
      "diff": "@@ -1,5 +1,5 @@...",
      "timestamp": "2024-01-15T10:25:00Z"
    },
    {
      "path": "Cargo.toml",
      "type": "modified",
      "diff": "@@ -10,3 +10,4 @@...",
      "timestamp": "2024-01-15T10:27:00Z"
    }
  ],
  "summary": "2 files modified while offline"
}
```

### Local Skills

TUI can load skills from the filesystem, enabling project-specific capabilities. Skills follow the [agentskills.io](https://agentskills.io/) specification (source of truth for skill format).

```markdown
<!-- .agents/skills/rust-linter/SKILL.md -->
---
name: rust-linter
description: Lint Rust code using cargo clippy. Use when working with Rust projects, before committing code, or when the user mentions clippy, lints, or Rust errors.
license: MIT
compatibility: Requires cargo and clippy installed
metadata:
  attache:
    id: rust_linter_v1
    category: code-quality
    tier: fast
    modality: text
    estimated-cost: 0.0001
---

## Available Scripts

- `scripts/lint.py` - Run clippy and parse output

## Usage

Run the linter on the entire project:

```bash
python3 scripts/lint.py
```

Or on specific files:

```bash
python3 scripts/lint.py src/main.rs src/lib.rs
```
```

```python
# .agents/skills/rust-linter/scripts/lint.py
# /// script
# dependencies = []
# ///

import subprocess
import json
import sys

def main(files=None):
    cmd = ["cargo", "clippy", "--message-format=json"]
    if files:
        cmd.extend(["--package", files[0].split('/')[0]])
    
    result = subprocess.run(cmd, capture_output=True, text=True)
    
    # Parse JSON output
    issues = []
    for line in result.stdout.split('\n'):
        if line:
            try:
                msg = json.loads(line)
                if msg.get("reason") == "compiler-message":
                    issues.append({
                        "file": msg["message"]["spans"][0]["file_name"],
                        "line": msg["message"]["spans"][0]["line_start"],
                        "message": msg["message"]["message"],
                        "level": msg["message"]["level"]
                    })
            except:
                pass
    
    return {
        "issues": issues,
        "count": len(issues)
    }

if __name__ == "__main__":
    files = sys.argv[1:] if len(sys.argv) > 1 else None
    result = main(files)
    print(json.dumps(result))
```

**Skill Discovery**:
```rust
pub async fn discover_local_skills(locations: &[PathBuf]) -> Vec<Skill> {
    let mut skills = Vec::new();
    
    for location in locations {
        if !location.exists() {
            continue;
        }
        
        let entries = tokio::fs::read_dir(location).await?;
        
        while let Some(entry) = entries.next_entry().await? {
            let path = entry.path();
            
            // Look for SKILL.md in subdirectories
            if path.is_dir() {
                let skill_md = path.join("SKILL.md");
                if skill_md.exists() {
                    let content = tokio::fs::read_to_string(&skill_md).await?;
                    let skill = parse_skill_md(&content)?;
                    
                    // Mark as filesystem skill
                    let skill = skill.with_source(SkillSource::Filesystem {
                        path: path.clone(),
                        readonly: true,
                    });
                    
                    skills.push(skill);
                }
            }
        }
    }
    
    skills
}
```

**Skill Precedence**:
1. Filesystem skills (from `.agents/skills/<name>/SKILL.md`) - **READONLY for this session**
2. Server skills (from skill registry) - **READWRITE**
3. If name collision: filesystem skill wins

## MDX/JSX Runtime

### Safe Component Model

Components run in isolated environment with strict constraints:

```typescript
interface SafeComponent {
  // Props-only input
  render(props: Record<string, any>): VNode;
  
  // Local state only (no global access)
  state?: {
    initial: Record<string, any>;
    schema: JSONSchema;
  };
  
  // No external access
  // No DOM manipulation
  // No shared state
}
```

### Security Constraints

| Constraint | Implementation |
|------------|----------------|
| No external API access | Network requests proxied through agent |
| No DOM escape | Render to virtual DOM only |
| No global state | Each component instance isolated |
| Props-only input | Parent controls all data flow |
| No eval/new Function | Static analysis + sandbox |

### Rendering Architecture

```
Skill Component (JSX)
    ↓
Build Step (Vite/esbuild)
    ↓
Component Bundle (ESM)
    ↓
Safe Runtime (iframe/Web Worker)
    ↓
Virtual DOM
    ↓
Render to HTML/CSS
    ↓
Client Display
```

### Built-in Components

```typescript
// Form input components
<Input type="text|number|email|password" />
<TextArea rows={number} />
<Select options={Array<{label, value}>} />
<Checkbox label="string" />
<RadioGroup options={Array} />
<DatePicker />
<FileUpload accept="string" />

// Display components
<Markdown content="string" />
<CodeBlock language="string" code="string" />
<Table data={Array} columns={Array} />
<Image src="string" alt="string" />
<Video src="string" controls={boolean} />

// Layout components
<Card title="string" actions={Array} />
<Accordion items={Array} />
<Tabs tabs={Array} />
<SplitPane left={VNode} right={VNode} />

// Action components
<Button variant="primary|secondary|danger" onClick={handler} />
<ButtonGroup buttons={Array} />
<Dropdown items={Array} />
```

### Custom Components from Skills

Skills can register custom components via `references/components.md`:

```markdown
<!-- .agents/skills/data-visualizer/references/components.md -->
# Components

## ChartCard

Renders interactive charts (line, bar, pie).

Props:
- `type` (enum: "line", "bar", "pie") - Chart type
- `data` (array) - Data points
- `title` (string) - Chart title
```

Components are bundled and served as HTML/CSS to the frontend.

## Agent Response Types

### Message Response
```typescript
interface MessageResponse {
  type: "message";
  id: string;
  timestamp: Date;
  mdx: string;           // MDX content
  artifacts?: Artifact[]; // Inline artifacts
}
```

Renders as standard chat message with MDX rendering.

### Artifact Update Response
```typescript
interface ArtifactUpdateResponse {
  type: "artifact_update";
  id: string;
  timestamp: Date;
  changes: Array<{
    artifactId: string;
    operation: "create" | "update" | "delete";
    previous?: Artifact;
    current?: Artifact;
    patch?: JSONPatch;    // For text diffs
  }>;
  message?: string;       // Optional explanation
}
```

Applies changes to canvas and adds entry to timeline.

### Request Input Response
```typescript
interface RequestInputResponse {
  type: "request_input";
  id: string;
  timestamp: Date;
  form: CardArtifact;    // Form-card component
  timeout?: number;      // Seconds until auto-close
  urgency: "low" | "medium" | "high";
}
```

Distinct visual treatment:
- Highlighted border
- Clear call-to-action
- Input fields within the card
- Cannot be edited, only submitted or dismissed

**Lifecycle**:
1. Agent creates request_input
2. Added to timeline and canvas
3. User fills form and submits
4. Response added as new timeline entry
5. Request marked complete (closed by human or agent)

### Query Handling & Background Processing

**Problem**: Some queries take a long time (research, complex analysis, multiple skill calls). User shouldn't be blocked waiting.

**Solution**: Progressive disclosure with background processing.

```typescript
interface QueryConfig {
  // Threshold for backgrounding (seconds)
  background_threshold: number;  // Default: 10
  
  // Maximum parallel queries
  max_parallel: number;          // Default: 3
  
  // Visual indicator
  spinner_style: "dots" | "pulse" | "progress";
}
```

**UX Flow**:

```
User sends query: "Research the history of Rust programming"
    ↓
Agent starts processing (< 10 seconds)
    ↓
[Spinner shown] "Assistant is researching..."
    ↓
10 seconds elapsed
    ↓
Query backgrounded
    ↓
UI state changes:
  - Message shows with spinner in background
  - Input box becomes available for new query
  - "1 background task running" indicator shown
    ↓
User can send new query: "What's the weather?"
    ↓
Agent responds to weather query immediately
    ↓
Background query completes
    ↓
Notification: "Research complete!"
    ↓
Response appears in timeline (may be above current view)
```

**Parallel Query Limit**:

```
Scenario: User sends 3 long-running queries in quick succession

Query 1: "Analyze my codebase" (>10s) → Backgrounded ✓
Query 2: "Research topic X" (>10s) → Backgrounded ✓  
Query 3: "Generate report" (>10s) → Backgrounded ✓
Query 4: "Quick question" → 
    [Max parallel reached]
    Options:
    - Wait for slot (queue)
    - Cancel one background task
    - Or: Query 4 is fast (<10s), process immediately
```

**Visual States**:

| State | Indicator | Input Available? |
|-------|-----------|------------------|
| Processing (<10s) | Spinner inline | No (blocked) |
| Background (>10s) | Spinner + "running" badge | Yes |
| Complete | Full response | Yes |
| Max parallel | Warning + queue | Depends |

**Background Task Indicator**:

```
┌─────────────────────────────────────────────────────────────────┐
│ 💬 Chat - 2 background tasks running                     [⚙️] │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ [Response to query 4 - immediate]                              │
│                                                                 │
│ [Response to query 3 - completed, may need scroll up]        │
│                                                                 │
│ ⏳ Researching Rust history... (running 2m)                    │
│ ⏳ Analyzing codebase... (running 45s)                       │
│                                                                 │
│ > New query available here                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Clicking Background Task**:
- Shows details: start time, elapsed time, estimated completion
- Option to cancel (if cancelable)
- Jump to message in timeline

**Configuration**:

```yaml
# .attache/config.yaml or user settings
query_handling:
  background_threshold_seconds: 10  # When to allow new input
  max_parallel_queries: 3             # Max concurrent background tasks
  show_progress_bar: true             # Visual progress indication
  notify_on_complete: true            # Desktop notification when done
```

### Decline Response

```typescript
interface DeclineResponse {
  type: "decline";
  id: string;
  reason?: string;        // Optional explanation (not shown to user)
  context: "user_query" | "auto_trigger";  // Why decline happened
}
```

**Behavior**:
- **Context = "user_query"**: User sent a message, agent declined → **Log entry created**: "Assistant declined to respond"
- **Context = "auto_trigger"**: Agent was auto-triggered (e.g., file save), declined → **No log entry** (silent)

This distinction ensures users know when their direct query was declined, while spontaneous declines don't clutter the timeline.

### Stream of Consciousness (Reasoning Stream)

A skill-controlled, real-time display of agent reasoning to maintain user engagement during processing.

#### Purpose
Long operations (research, multi-step analysis) leave users waiting. The reasoning stream shows what's happening, keeps users engaged, and provides transparency.

#### Stream Events

```typescript
interface StreamEvent {
  type: "thinking";
  queryId: string;
  timestamp: Date;
  sequence: number;        // For ordering
  
  // Content
  phase: "research" | "analysis" | "synthesis" | "complete";
  step: string;           // User-visible description (sanitized)
  details?: string;       // Additional context
  
  // Progress
  progress?: number;      // 0-100
  estimatedRemaining?: number;  // Seconds (optional)
  
  // State
  status: "running" | "completed" | "failed" | "cancelled";
  icon?: "search" | "read" | "analyze" | "write" | "check" | "clock";
  
  // Privacy
  containsSecrets: boolean;  // If true, step/details redacted
}
```

**Event Sequence Example**:
```
1. {phase: "research", step: "Breaking down request...", status: "running"}
2. {phase: "research", step: "Running web_search skill", 
    details: "Query: 'Rust history'", status: "running", icon: "search"}
3. {phase: "research", step: "Found 12 sources", 
    progress: 20, status: "completed", icon: "check"}
4. {phase: "research", step: "Reading Wikipedia...", 
    status: "running", icon: "read"}
5. {phase: "analysis", step: "Comparing sources...", 
    progress: 60, status: "running", icon: "analyze"}
6. {phase: "synthesis", step: "Compiling final answer...", 
    progress: 90, status: "running", icon: "write", estimatedRemaining: 30}
7. {phase: "complete", step: "Done!", 
    progress: 100, status: "completed"}
```

#### Skill Configuration

Skills control default stream behavior:

```yaml
# deep_research.yaml
name: "deep_research"
description: "Multi-step research with full reasoning"

stream_config:
  default_level: "detailed"  # none | minimal | normal | detailed
  show_progress: true
  show_eta: true           # This skill calculates ETAs
  
  # Per-operation defaults
  operation_defaults:
    quick_lookup: "minimal"
    multi_source_research: "detailed"
```

**Verbosity Levels**:

| Level | What's Shown | Use Case |
|-------|---------------|----------|
| **none** | Just spinner | Simple queries |
| **minimal** | Phase names only | Quick operations |
| **normal** | Key milestones + skill names | Standard work |
| **detailed** | All reasoning + tool calls + results | Complex research |

#### UI Presentation

**Compact State (Default - 3 Lines)**:
```
┌─────────────────────────────────────────────────────────────────┐
│ ⠋ Researching Rust history... (2m 15s) ~30s remaining            │
│   → Running web_search skill...                                  │
│   → Found 12 sources, reading Wikipedia...                  [⛶] │
└─────────────────────────────────────────────────────────────────┘
```

- Auto-scrolls: Newest at bottom, oldest disappears
- [⛶] = Maximize button
- Elapsed time shown
- ETA shown if skill provides `estimatedRemaining`

**Maximized State**:
```
┌─────────────────────────────────────────────────────────────────┐
│ ⏳ Reasoning Stream - ~30s remaining                    [⛶] [✕] │
├─────────────────────────────────────────────────────────────────┤
│ • 09:00:00 ⠋ Starting research phase                          │
│ • 09:00:02   Breaking down request...                          │
│ • 09:00:05 ⠋ Running web_search skill                         │
│              Query: "Rust programming language history"         │
│ • 09:00:08 ✓ Found 12 sources                                 │
│ • 09:00:10 ⠋ Reading: Wikipedia - Rust (language)             │
│              Extracting: timeline, authors, milestones        │
│ • 09:00:25 ✓ Sources complete (3/3 priority)                 │
│ • 09:00:26 ⠋ Analysis phase: Comparing sources...            │
│ • 09:00:30 ⠋ Synthesizing timeline...                         │
│ • 09:00:35 ⠋ Compiling final answer... (~30s remaining)       │
│                                                                 │
│ [Stop Request]                                                  │
└─────────────────────────────────────────────────────────────────┘
```

- Full scrollback visible
- Can scroll to beginning
- [Stop Request] button

**When Response Arrives**:
```
Stream auto-collapses, response shown:
┌─────────────────────────────────────────────────────────────────┐
│ You: Research Rust programming history                        │
│                                                                 │
│ [Response content appears here]                                │
│                                                                 │
│ (2m 15s reasoning, 12 sources, detailed) [View reasoning]    │
└─────────────────────────────────────────────────────────────────┘
```

#### Cancel / Stop Functionality

| Method | Action |
|--------|--------|
| Stop button | Click [Stop Request] in maximized view |
| Escape ×2 | Double-tap Escape key |

**Cancel Behavior**:
1. Sends cancel signal to agent
2. Agent stops gracefully (completes current atomic operation)
3. Stream shows: "⛔ Cancelled by user"
4. Final response: "Query was cancelled"
5. Reasoning log preserved with "cancelled" status

#### Privacy & Secret Redaction

```typescript
function sanitizeEvent(event: StreamEvent): StreamEvent {
  if (containsSecrets(event.step) || containsSecrets(event.details)) {
    return {
      ...event,
      step: redactSecrets(event.step),  // e.g., "[REDACTED]"
      details: "[REDACTED]",
      // Keep phase, progress, icon visible
    };
  }
  return event;
}
```

**Examples**:
```
Before: "Calling API with token sk-abc123xyz"
After:  "Calling API with [REDACTED]"

Before: "Querying database: user_passwords"
After:  "Querying [REDACTED]"
```

#### Per-Query Toggle

Users can override per query:

```bash
# Override for this query
User: "@agent stream=detailed Research quantum computing"
Agent: "Starting with detailed reasoning stream..."

# Or mid-stream
User: "Make stream minimal"
Agent: "Switching to minimal display..."
```

#### Persistence for Reflection

```typescript
interface ReasoningLog {
  queryId: string;
  timestamp: Date;
  events: StreamEvent[];
  duration: number;
  outcome: "success" | "cancelled" | "failed";
  config: StreamConfig;
  
  // Reflection metadata
  priority: "low";        // Always low priority for memory
  accessedCount: number;
  lastAccessed?: Date;
}
```

**Storage**: Uses same memory mechanisms as everything else (3rd party memory tier).

**Reflection Usage**:
- Agent analyzes logs to improve skills
- Identify patterns: "User cancels at phase X", "Common failure points"
- Can delete: "Reflection: Deleted 100 old reasoning logs"
- Can extract patterns: "Extract common research flows to 3rd party memory"

#### Configuration

```yaml
# .attache/config.yaml
stream_of_consciousness:
  # User default (overridable per query)
  default_level: "normal"
  
  # UI behavior
  max_compact_lines: 3
  show_elapsed_time: true
  show_eta_when_available: true
  
  # Cancel
  enable_cancel: true
  cancel_keys: ["Escape", "Escape"]  # Double-tap
```

## Skills System

> **Source of Truth**: Skills follow the [agentskills.io specification](https://agentskills.io/specification). This section documents Attaché-specific extensions.

### Skill Definition

Skills are directories containing a `SKILL.md` file with YAML frontmatter and Markdown instructions:

```
web-search/
├── SKILL.md              # Required: metadata + instructions
├── scripts/              # Optional: executable code
│   └── search.py
├── references/           # Optional: documentation
│   └── components.md     # Attaché: JSX component definitions
└── assets/               # Optional: templates, resources
```

**SKILL.md Format**:

```markdown
---
name: web-search
description: Search the web for current information. Use when the user needs up-to-date facts, current events, or information not in your training data.
license: MIT
metadata:
  attache:
    id: web_search_v2
    category: research
    tier: fast
    modality: text
    estimated-cost: 0.001
---

## Available Scripts

- `scripts/search.py` - Execute web search using SERP API

## Usage

Run the search script with your query:

```bash
python3 scripts/search.py "your search query"
```

## Components

See [references/components.md](references/components.md) for UI components.
```

### agentskills.io Standard Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Lowercase, hyphens, max 64 chars. Must match directory name. |
| `description` | Yes | Max 1024 chars. Describe what AND when to use. |
| `license` | No | License name or file reference. |
| `compatibility` | No | Environment requirements (e.g., "Requires Python 3.11+"). |
| `metadata` | No | Arbitrary key-value pairs for agent-specific extensions. |

### Attaché Extensions (in `metadata.attache`)

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique skill identifier (e.g., "web_search_v2"). |
| `category` | string | Skill category for organization. |
| `tier` | enum | Model routing hint: `fast`, `balanced`, `advanced`. |
| `modality` | enum | Input type: `text`, `vision`, `multilateral`. |
| `estimated-cost` | number | USD cost per invocation (for routing optimization). |

### Skill Execution Environment

**Sandbox**:
- gVisor or Firecracker container
- No network access by default
- All external calls through proxy outside VPC
- Resource limits: CPU, memory, disk
- Timeout enforcement

**Script Execution**:
- Python, Bash, JavaScript, or any executable
- Scripts declare dependencies inline (PEP 723 for Python, etc.)
- Skill can call other skills via API
- Skill can rewrite itself (self-modification)

**Execution Flow**:
```
Agent receives task
    ↓
Skill selected based on metadata.attache routing hints
    ↓
Environment variables injected (from Agent Secrets volume ONLY)
    ↓
Sandbox spawned
    ↓
Script executes
    ↓
Result returned to agent
    ↓
Response formatted (MDX + artifacts)
```

**Important Security Boundary**: Skills NEVER receive environment variables from the TUI or local filesystem. Even when triggered via TUI, skills only receive env vars from the secure agent volume. This prevents accidental leakage of local environment variables to the agent.

### Skill API

```python
# Available in skill runtime
from attache import (
    call_skill,      # Call another skill
    create_artifact, # Create canvas artifact
    log,             # Structured logging
    request_input,   # Request human input
    search_memory,   # Query memory store
    llm_call,        # Direct LLM call
)

def main(query: str, max_results: int = 10) -> dict:
    """Main entry point"""
    # Search implementation
    results = search_api(query, max_results)
    
    # Create artifact
    artifact = create_artifact(
        type="card",
        component="SearchResultsCard",
        props={"query": query, "results": results}
    )
    
    return {
        "message": f"Found {len(results)} results",
        "artifacts": [artifact]
    }
```

### Skill Storage

Skills are stored as directories in the filesystem:
- `SKILL.md` - Metadata + instructions
- `scripts/` - Executable code
- `references/` - Documentation and component definitions
- `assets/` - Static resources

**Operations**:
- Create new skill (write directory)
- Update existing skill (modify files, version tracked in `id` field)
- Deprecate skill (moved to archive, not deleted)
- Consolidate skills (merge related skills)
- Rewrite skill (self-modification)

All operations logged. No special version control system - just files on disk.

### References Directory

The `references/` directory contains optional documentation:

```
web-search/
├── SKILL.md
├── scripts/
│   └── search.py
└── references/
    ├── components.md     # JSX component definitions
    ├── REFERENCE.md      # Detailed technical reference
    └── examples.md       # Usage examples
```

**components.md** (Attaché-specific):
```markdown
# Components

## SearchResultsCard

Display web search results with clickable links.

Props:
- `query` (string): The search query
- `results` (array): Array of {title, url, snippet} objects
```

## Memory Model

Cognitive-inspired architecture with three memory types, each with 1st/2nd/3rd party sources:

| Memory Type | Purpose | Content | Source Tiers |
|-------------|---------|---------|--------------|
| **Semantic** | Factual knowledge (what you know) | Facts, concepts, preferences | 1st (inferred), 2nd (explicit), 3rd (external) |
| **Episodic** | Event experiences (what happened) | Chat logs, data fetches, interactions | 1st (agent experiences), 2nd (user events), 3rd (external events) |
| **Procedural** | Skills & procedures (how to do it) | Skills, execution patterns, tools | 1st (agent-created), 2nd (user-created), 3rd (community/imported) |

All share storage, indexing, and retrieval; differ by content type and access patterns.

### Semantic Memory (Factual Knowledge)

**Purpose**: Store facts, concepts, user preferences, learned knowledge

```typescript
interface SemanticMemory {
  id: string;
  type: "semantic";
  
  // Content classification
  category: "preference" | "fact" | "concept" | "inference";
  content: string;           // The knowledge/fact
  confidence: number;        // 0-1 (for inferred knowledge)
  
  // Source (1st/2nd/3rd party)
  source: {
    tier: "1st" | "2nd" | "3rd";
    origin: "inferred" | "explicit" | "external";
    createdBy: "agent" | "user" | "import";
  };
  
  // Metadata
  createdAt: Date;
  lastAccessed: Date;
  accessCount: number;
  vector: number[];          // For semantic search
}
```

**Examples by Source**:
- **1st party (inferred)**: "User prefers concise responses", "User likes Python"
- **2nd party (explicit)**: "My favorite restaurant is Joe's Pizza", "Remind me Sundays"
- **3rd party (external)**: "Rust was created in 2010", "From Wikipedia article"

**Reflection Operations**:
- Consolidate similar facts
- Resolve contradictions
- Update confidence scores
- Archive low-confidence inferences

**Query Examples**:
- "What do you know about me?" → Semantic Memory search
- "What facts have you learned?" → Semantic Memory query

### Episodic Memory (Event Experiences)

**Purpose**: Store timeline of events, interactions, experiences

```typescript
interface EpisodicMemory {
  id: string;
  type: "episodic";
  
  // Event classification
  eventType: "chat" | "data_fetch" | "skill_execution" | 
             "user_action" | "agent_action" | "reflection";
  timestamp: Date;
  content: string;           // What happened
  
  // Context
  context: {
    query?: string;        // User's query (for chats)
    response?: string;     // Agent's response
    artifacts?: string[];  // Related artifacts
    duration?: number;      // Event duration (seconds)
    outcome?: "success" | "cancelled" | "failed";
  };
  
  // Source
  source: {
    tier: "1st" | "2nd" | "3rd";
    actor: "agent" | "user" | "system" | "external";
  };
  
  // For retrieval
  vector: number[];        // Embedding
  tags: string[];           // ["rust", "research", "2024-01"]
}
```

**Examples by Source**:
- **1st party (agent)**: "Agent researched Rust for 5m, found 12 sources"
- **2nd party (user)**: "User cancelled query during research phase"
- **3rd party (external)**: "External API returned 503 error at 14:23"

**Reflection Operations**:
- Summarize old episodes (compress)
- Extract patterns ("User often cancels at phase X")
- Archive by date ranges
- Extract to Semantic Memory ("User likes quantum computing")

**Query Examples**:
- "What did we discuss last week?" → Episodic Memory query
- "What was I working on yesterday?" → Episodic Memory search
- "When did I ask about Rust?" → Episodic Memory timeline

### Procedural Memory (Skills & Procedures)

**Purpose**: Store skills, how to perform tasks, execution patterns

```typescript
interface ProceduralMemory {
  id: string;
  type: "procedural";
  
  // Skill definition
  name: string;
  description: string;
  implementation: {
    type: "python" | "jsx" | "composite";
    code: string;
    version: string;
  };
  
  // Source
  source: {
    tier: "1st" | "2nd" | "3rd";
    createdBy: "agent" | "user" | "community";
  };
  
  // Performance metrics
  usageCount: number;
  successRate: number;
  avgExecutionTime: number;
  
  // Reflection metadata
  lastOptimized: Date;
  optimizationHistory: OptimizationRecord[];
}
```

**Examples by Source**:
- **1st party (agent-created)**: "web_search" skill (agent inferred need)
- **2nd party (user-created)**: User's "analyze_codebase" custom skill
- **3rd party (community)**: Imported "weather_api" from skill marketplace

**Reflection Operations**:
- Optimize based on usage patterns
- Consolidate overlapping skills
- Improve success rates
- Self-modify based on feedback

**Query Examples**:
- "How do you search the web?" → Procedural Memory (skill lookup)
- "What skills do you have for Rust?" → Procedural Memory query
- "Improve your research skill" → Procedural Memory reflection

### Memory Operations

```typescript
interface MemoryAPI {
  // Store to specific memory type
  storeSemantic(memory: SemanticMemory): Promise<void>;
  storeEpisodic(memory: EpisodicMemory): Promise<void>;
  storeProcedural(memory: ProceduralMemory): Promise<void>;
  
  // Cross-type search (queries appropriate type based on query)
  search(query: string, type?: MemoryType, limit?: number): Promise<Memory[]>;
  
  // Type-specific queries
  querySemantic(filters: SemanticFilters): Promise<SemanticMemory[]>;
  queryEpisodic(filters: EpisodicFilters): Promise<EpisodicMemory[]>;
  queryProcedural(filters: ProceduralFilters): Promise<ProceduralMemory[]>;
  
  // Retrieve by ID (any type)
  get(id: string): Promise<Memory>;
  
  // Update
  update(id: string, updates: Partial<Memory>): Promise<void>;
  
  // Delete (soft - moved to archive)
  delete(id: string): Promise<void>;
}
```

### Memory Type Selection

**Agent automatically routes queries**:

| User Query | Memory Type | Example |
|------------|-------------|---------|
| "What do you know about X?" | Semantic | Facts about X |
| "What did we discuss Y?" | Episodic | Chat about Y |
| "How do you do Z?" | Procedural | Skill for Z |
| "What was I working on?" | Episodic | Recent activities |
| "What are my preferences?" | Semantic | User profile |

### Vector Indexing

All three memory types indexed for semantic retrieval:
- **Embedding model**: OpenAI text-embedding-3 or equivalent
- **Vector DB**: Qdrant or pgvector
- **Strategy**: Type-specific chunking
  - Semantic: Paragraph-level
  - Episodic: Event-level with context
  - Procedural: Skill-level with documentation

### Exploration Interface

```
Explore
├── 🔮 Semantic Memory      (Knowledge & Facts)
│   ├── By Category: Preferences, Facts, Concepts
│   ├── By Source: 1st Party, 2nd Party, 3rd Party
│   └── Search: "What do I know?"
│
├── 📅 Episodic Memory      (History & Events)
│   ├── Timeline: Chronological view
│   ├── By Type: Chats, Data Fetches, Actions
│   └── Search: "What happened?"
│
└── 🛠️ Procedural Memory    (Skills & Procedures)
    ├── By Category: Research, Analysis, Tools
    ├── By Source: Agent-created, User-created, Community
    └── Search: "How do I?"
```

## Secrets System

### Storage

**Encrypted Store**:
- Row-level encryption with unique keys
- AES-256-GCM
- Keys derived from master key + row salt
- Stored in dedicated database

**Access Control**:
```typescript
interface SecretAccess {
  secretId: string;
  skillId?: string;       // If scoped to skill
  userId: string;
  issuedAt: Date;
  expiresAt: Date;
  jwt: string;            // Signed JWT with claims
}
```

### Redaction System

All storage systems implement automatic redaction:

```typescript
interface RedactionEngine {
  // Scan content for secrets
  scan(content: string): SecretMatch[];
  
  // Redact found secrets
  redact(content: string): string;
  // Example: "api_key: sk-1234" → "api_key: [REDACTED:hash]"
}
```

**Applied to**:
- Chat history logs
- Memory storage
- Skill logs
- Debug output
- Error messages

### Secrets Panel (Desktop/TUI)

```bash
# Add secret
attache secrets set SERP_API_KEY

# List secret names (values never shown)
attache secrets list
# SERP_API_KEY (set 2024-01-15)
# GITHUB_TOKEN (set 2024-01-10)

# Delete secret
attache secrets delete SERP_API_KEY

# Rotate secret
attache secrets rotate SERP_API_KEY
```

**UI Panel**:
- Table of secret names (no values)
- Set (prompts for value)
- Delete (confirmation required)
- Rotate (generate new, invalidate old)
- Never a "view" button

### Skill Access

Skills declare required environment variables:

```yaml
env:
  - name: "API_KEY"
    required: true
```

At runtime:
1. System checks if secret exists in **Agent Secrets volume**
2. If yes: injects into sandbox environment
3. If no: agent requests user to set secret via Secrets panel
4. Secret value never logged or stored in skill

**Security Boundary**: Skills NEVER receive environment variables from:
- The TUI process or its environment
- The local filesystem or shell
- User's local `.env` files or shell profile
- Parent process environment

All environment variables for skills come exclusively from the encrypted Agent Secrets volume. This ensures:
- No accidental leakage of local env vars to the agent
- Consistent environment across all clients (desktop, web, TUI)
- Centralized secret management with proper auditing
- Skills work identically regardless of which client triggered them

## Cone of Silence

### Volume Model

**Primary Volume**:
- All persistent data
- Read-write for normal operation
- Read-only during cone sessions

**Cone Volume** (ephemeral):
- Created when cone starts
- Stores only cone session data
- Has read-only access to primary volume history
- Destroyed when cone ends

```typescript
interface ConeSession {
  id: string;
  clientId: string;       // Which device initiated
  startedAt: Date;
  endedAt?: Date;
  primaryVolume: VolumeRef;    // Read-only access
  ephemeralVolume: VolumeRef;  // Read-write, temporary
}
```

### Lifecycle

**Starting a Cone**:
1. User invokes cone mode (explicit action)
2. New ephemeral volume created
3. Primary volume mounted read-only
4. Chat histories merged for display
5. Visual theme changes to indicate cone mode

**During Cone**:
- All interactions tagged with `cone: true`
- Visual distinction: dark theme, warning banner
- Agent can suggest cone mode (triggers after suggestion)
- Multiple cones can be active (isolated per client)

**Ending a Cone**:
1. User ends session or closes client
2. Ephemeral volume destroyed
3. Primary volume history restored (no cone entries)
4. Visual theme reverts

### Visual Indicators

**Cone Mode UI**:
- Dark/reduced theme
- "Cone of Silence" badge in header
- Query box has warning border
- All messages have subtle cone indicator

**Copy Warning**:
- Normal copy buttons: work as expected
- Cone copy buttons: show warning modal
  - "This content is from a private session"
  - "It will not be saved to your history"
  - Confirm to proceed

### Suggestion Flow

When agent detects potentially sensitive query:

```
User: "I need to discuss my medical diagnosis"
Agent: Message response (normal)
Agent: Suggests cone mode (after response)
UI: "Start private session? [Yes] [No]"
If Yes: Cone starts (new volume, merged view)
If No: Continue normally
```

## Reflection System

### Budget Model

```typescript
interface ReflectionBudget {
  // Total allocation
  monthlyTokens: number;     // e.g., 10M tokens
  monthlyDurationMs: number; // e.g., 1 hour compute
  
  // Distribution
  hourlyBudget: number;      // Light folding of recent data
  dailyBudget: number;       // Deep processing of day
  extensiveBudget: number;   // Major reorganization (triggered by threshold)
}
```

**Phases**:

| Phase | Trigger | Scope | Budget |
|-------|---------|-------|--------|
| **Hourly** | Scheduled | Last hour of data | Small |
| **Daily** | Scheduled | Last 24 hours | Medium |
| **Extensive** | Learning threshold | All data | Large |

### Operations

**Consolidation**:
- Merge redundant memories
- Summarize long conversations
- Extract key facts

**Rebalancing**:
- Adjust memory weights based on access patterns
- Promote frequently accessed
- Deprecate unused

**Purge**:
- Remove low-confidence 1st party inferences
- Archive old 3rd party data
- Soft delete (recoverable from archive)

**Reindex**:
- Update vector embeddings
- Optimize search performance
- Rechunk documents

**Skill Improvement**:
- Analyze skill usage patterns
- Rewrite inefficient skills
- Consolidate overlapping skills
- Generate new examples

### Gap Detection

During reflection, agent identifies:
- Missing knowledge areas
- Frequently asked unanswered questions
- Incomplete skill coverage

**Action**:
- Create `request_input` for user
- Suggest skill development
- Queue 3rd party data ingestion

### User-Requested Reflection

Skill-based invocation:

```yaml
# reflect_now.yaml
name: "reflect_now"
description: "Request immediate reflection on specific topics"
requires_approval: true  # Human must approve
```

User can request:
- Scope: "all", "skills", "memory", "recent"
- Priority: normal or high

UI shows progress but doesn't block.

### Reflection Queue (Non-Reflection Time)

**Problem**: Agent or human identifies issues during normal operation, but reflection isn't running right now.

**Solution**: A queue where either party can add suggestions for the next reflection.

```typescript
interface ReflectionSuggestion {
  id: string;
  timestamp: Date;
  source: "human" | "agent";
  type: "consolidation" | "rebalance" | "skill_improvement" | "gap_detection" | "other";
  reflection_size: "short" | "medium" | "long" | "extensive";
  description: string;
  context?: string;        // Additional details
  priority: "low" | "medium" | "high";
  status: "pending" | "in_progress" | "completed" | "deferred";
}
```

**Who Can Add**:

| Source | How | Examples |
|--------|-----|----------|
| **Human** | Command or card | "Review my Rust skills", "Consolidate travel memories" |
| **Agent** | Auto-detection | "Skill 'web_search' is slow", "Memory X hasn't been accessed in 30 days" |

**Adding Suggestions**:

```bash
# Human adds via TUI
attache suggest-reflection --size short "Review my Python skills"

# Human adds via chat
User: "@agent queue a medium reflection to consolidate my project notes"
Agent: "Added to reflection queue (medium) - will process during next scheduled reflection."
```

**Reflection Size Indicators**:

| Size | Duration | Tokens | When to Use |
|------|----------|--------|-------------|
| **Short** | 5-10 min | ~50K | Quick consolidation, single skill tweak |
| **Medium** | 15-30 min | ~200K | Rebalance recent memories, improve 1-2 skills |
| **Long** | 1-2 hours | ~1M | Deep reorganization, multiple skill improvements |
| **Extensive** | 3+ hours | ~5M+ | Major system overhaul, triggered by threshold |

**During Reflection**:

1. Agent checks reflection queue
2. Selects suggestions fitting current budget
3. Prioritizes by:
   - Priority level (high > medium > low)
   - Source (human requests first)
   - Age (older suggestions first)
   - Size (smaller fits better in limited budgets)
4. Executes suggestions
5. Updates status to "completed" or "deferred" (if budget exhausted)

**Example Queue**:

```
Reflection Queue (3 pending):

1. [HIGH] Human: "Review Rust skills" (short)
   Added: 2024-01-15 09:00
   
2. [MEDIUM] Agent: "Skill 'web_search' slow" (short)
   Added: 2024-01-15 08:30
   
3. [LOW] Agent: "Consolidate old travel memories" (long)
   Added: 2024-01-14 14:00

Next reflection (hourly, budget: short) will process: #1, #2
```

**Notification**: When human suggestion is completed:
- "Completed your reflection suggestion: 'Review Rust skills' - found 3 areas for improvement"

## Storage Architecture

### Per-User Volume

Each user has isolated storage:

```
/volumes/{user_id}/
  ├── primary/
  │   ├── chat/           # PostgreSQL (or SQLite)
  │   ├── artifacts/      # Filesystem
  │   ├── vectors/        # Qdrant data
  │   ├── skills/         # Filesystem (regular files, like artifacts)
  │   ├── memory/         # JSON/Parquet files
  │   └── config.sqlite   # User settings
  │
  └── cones/
      └── {cone_id}/      # Ephemeral, destroyed after use
          ├── chat/       # Session-only history
          └── artifacts/  # Session-only files
```

**Benefits**:
- Easy backup (single volume)
- Easy migration (move volume)
- Strong isolation (per-user)
- Cone of Silence isolation (separate ephemeral volume)

### Database Types

| Data | Store | Reason |
|------|-------|--------|
| Chat timeline | PostgreSQL | Relational, time-series |
| Artifacts | Filesystem | Binary files, Git-like versioning |
| Vectors | Qdrant | Semantic search optimized |
| Skills | Git repo | Version control, diffs |
| Memory | Parquet/JSON | Columnar for analytics |
| Secrets | Encrypted SQLite | Security, row-level keys |
| Config | SQLite | Simple key-value |

### Backup & Migration

**Backup**:
```bash
# Export user volume
attache admin export-volume {user_id} > user_backup.tar.gz

# Import to new instance
attache admin import-volume user_backup.tar.gz
```

**Migration**:
- Volume is self-contained
- Move file, update DNS, done
- No database migrations needed

### Scaling

**Horizontal**:
- User volumes distributed across servers
- Consistent hashing by user_id
- No cross-user queries

**Vertical**:
- Large users can have dedicated volume servers
- Hot volumes cached in memory
- Cold volumes archived to object storage

---

*See also: [Overview](./README.md), [Architecture](./ARCHITECTURE.md), [UX](./UX.md), [Test Cases](./TEST-CASES.md)*

---

*Document Version: 1.0*  
*Part of the [Attaché Product Requirements](./README.md)*
