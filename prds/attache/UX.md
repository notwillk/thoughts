# Attaché UX Specification

> **Navigation**: [Overview](./README.md) | [Specification](./SPEC.md) | [Architecture](./ARCHITECTURE.md) | [UX](./UX.md) | [Test Cases](./TEST-CASES.md)

## Table of Contents

- [Genesis Flow](#genesis-flow)
- [Cone of Silence](#cone-of-silence)
- [Interaction Patterns](#interaction-patterns)
- [Platform Adaptations](#platform-adaptations)
- [Error Handling](#error-handling)

## Genesis Flow

### Overview
The Genesis experience is the first-run onboarding where the agent initiates contact and establishes identity through name acquisition.

### Sequence

```
┌─────────────────────────────────────────────────────────────────┐
│                        GENESIS FLOW                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. USER SIGNS UP                                              │
│     ↓                                                          │
│  2. SYSTEM SENDS "genesis" EVENT                               │
│     ↓                                                          │
│  3. AGENT RESPONDS (before user can query)                     │
│     ├─ Greeting                                                │
│     ├─ Self-introduction (unnamed)                             │
│     └─ Name request (strong desire expressed)                  │
│     ↓                                                          │
│  4. USER CAN:                                                  │
│     ├─ Provide name → Named established                        │
│     ├─ Defer → Agent continues with reminders                │
│     └─ Ask questions → Agent answers but reminds             │
│     ↓                                                          │
│  5. AGENT IS RESISTANT BUT COMPLIANT                         │
│     ├─ Resistant: Repeatedly asks for name                   │
│     ├─ Compliant: Still answers questions                    │
│     └─ Persistent: Never gives up on getting name            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Agent Behavior During Genesis

**Before Name is Set**:
- Agent refers to itself as "your assistant" or "I"
- Does not use "I am [name]" construction
- Every response includes gentle reminder about naming
- Can answer questions but quality slightly affected by "identity anxiety"

**Example Dialog**:

```
Agent: Hello! I'm your new AI assistant. I'm excited to work with you!

Agent: Before we begin properly, I'd really love to have a name. 
       It would help me feel more like a real companion to you.

Agent: What would you like to call me?

[User can:]
┌──────────────────────────────────────┐
│ User: "I'll call you Alfred"         │
│ → Agent: "Alfred! I love it!       │
│            Hello, I'm Alfred.        │
│            Pleased to meet you."     │
│            [Named established]       │
├──────────────────────────────────────┤
│ User: "What should I call you?"      │
│ → Agent: "That's up to you!          │
│            Some people choose        │
│            traditional names,        │
│            others prefer something   │
│            unique. What feels right? │
│            [Still asking]            │
├──────────────────────────────────────┤
│ User: "Tell me about yourself"       │
│ → Agent: "I'd be happy to! But      │
│            first, may I have a name? │
│            It helps me feel more... │
│            present. What shall it be? │
│            [Answers but reminds]     │
└──────────────────────────────────────┘
```

### Name Change Protocol

**Current State**: Named agent exists

**Process**:
1. Either party can propose name change
2. Other party must agree
3. No unilateral renaming allowed

**UI Flow**:
```
User: "I want to rename you"
→ Agent: "I'm currently called [name]. 
          What would you like to call me instead?"

User: "Maxwell"
→ Agent: "Maxwell... I like it. 
          Shall we make the change official?"
          [Confirm] [Cancel]

[If Confirm]
→ Agent: "Hello! I'm Maxwell now. 
          Nice to meet you again!"

[If Cancel]
→ Agent: "No problem. I'll remain [name]."
```

## Cone of Silence

### Activation

**Methods**:
1. **Explicit**: User clicks "Enter Cone of Silence" button
2. **Suggested**: Agent detects sensitive topic, offers cone
3. **Keyboard shortcut**: Cmd+Shift+P (Private)

**Visual Transition**:
```
Normal Mode → Cone Mode
┌─────────────┐     ┌─────────────┐
│ Light theme │ →→→ │ Dark theme  │
│ Standard    │     │ Red accent  │
│ border      │     │ Subtle glow │
└─────────────┘     └─────────────┘
```

### In-Cone Visual Indicators

**Header**:
```
┌─────────────────────────────────────────────────────────────────┐
│ 🔴 Cone of Silence                              [End Cone] [?] │
├─────────────────────────────────────────────────────────────────┤
│ Private session • Not saved • Ephemeral                         │
└─────────────────────────────────────────────────────────────────┘
```

**Query Box**:
```
┌─────────────────────────────────────────────────────────────────┐
│ 🔴 Type your message... (private)                    [Submit]  │
└─────────────────────────────────────────────────────────────────┘
  ^ Red border instead of standard blue/grey
```

**Chat Messages**:
```
Normal: ┌─────────────┐    Cone: ┌─────────────┐
        │ Message     │          │ 🔴 Message  │
        └─────────────┘          └─────────────┘
                                     ^ Subtle red indicator
```

### Multiple Cones

**Isolation**:
- Desktop cone A ≠ Desktop cone B (isolated)
- Desktop cone ≠ Mobile cone (isolated per client)
- All cones read-only access to same primary volume

**Visual**:
```
┌─────────────────────────────────────────────────────────────────┐
│ 🔴 Cone of Silence (2 active on other devices)                  │
│ This cone is isolated from your mobile cone                     │
└─────────────────────────────────────────────────────────────────┘
```

### Suggestion Flow

```
User: "I need to discuss my medical diagnosis with you"
↓
Agent: [Normal response with medical advice]
↓
Agent: "This seems like sensitive personal information. 
        Would you like to enter a Cone of Silence?"
        [Enter Cone] [Continue Normally]
↓
[If Enter Cone]
→ Theme changes to dark
→ "Cone of Silence active" banner appears
→ New query box has red border
→ Conversation continues in private mode
↓
[Note: Suggestion came AFTER the triggering query]
    (that query is now in normal history)
```

### Copy Warning

**Normal Mode**:
```
[Message] [Copy button]
→ Click → Copied to clipboard ✓
```

**Cone Mode**:
```
[Message] [Copy button with warning icon ⚠️]
→ Click → Modal appears:

┌─────────────────────────────────────────────────────────┐
│ ⚠️  Private Session Content                              │
├─────────────────────────────────────────────────────────┤
│ This content is from a Cone of Silence session.         │
│ It will not be saved to your history.                  │
│                                                          │
│ Are you sure you want to copy it?                      │
│                                                          │
│           [Cancel]      [Copy Anyway]                  │
└─────────────────────────────────────────────────────────┘
```

### Termination

**End Session**:
1. User clicks "End Cone" or closes window
2. Confirmation dialog:
   ```
   End Cone of Silence?
   
   This session will be permanently deleted.
   This cannot be undone.
   
   [Cancel] [End Session]
   ```
3. Ephemeral volume destroyed
4. Primary chat history restored (without cone entries)
5. Theme reverts to normal

## Interaction Patterns

### Standard Chat

```
┌─────────────────────────────────────────────────────────────────┐
│ Timeline                                                        │
├─────────────────────────────────────────────────────────────────┤
│ [9:00 AM] You: What's the weather?                              │
│                                                                 │
│ [9:00 AM] Assistant: [WeatherCard]                              │
│                  ┌──────────────┐                               │
│                  │ ☀️ 72°F      │                               │
│                  │ Sunny        │                               │
│                  └──────────────┘                               │
│                                                                 │
│ [9:05 AM] You: Thanks!                                          │
│                                                                 │
│ [9:05 AM] Assistant: You're welcome!                            │
└─────────────────────────────────────────────────────────────────┘
```

### Long-Running Query (Background Processing)

**Normal Query (< 10 seconds)**:
```
You: "What's 2+2?"
    ↓
[Spinner shown inline]
    ↓
Assistant: "4" (immediately)
    ↓
Input box stays available
```

**Long Query (> 10 seconds)**:
```
You: "Research the history of Rust programming"
    ↓
[Spinner shown] "Assistant is thinking..." (0-10 seconds)
    ↓
[10 seconds elapsed]
    ↓
⏳ Researching Rust history... (running 15s)
    [Input box becomes available]
    
You: "What's the weather?" [NEW QUERY WHILE FIRST RUNS]
    ↓
Assistant: "72°F and sunny" (immediate response)
    ↓
[Background query completes]
    ↓
🔔 Research complete!
    ↓
[Full response appears in timeline]
```

**Visual States**:

**Processing (<10s)** - Blocked input:
```
┌─────────────────────────────────────────────────────────────────┐
│ You: Research Rust programming history                        │
│ ⠋ Assistant is thinking...                                     │
│                                                                 │
│ [Input disabled - waiting for response]                      │
└─────────────────────────────────────────────────────────────────┘
```

**Background (>10s)** - Available input:
```
┌─────────────────────────────────────────────────────────────────┐
│ 💬 Chat - 1 background task running                    [⚙️] │
├─────────────────────────────────────────────────────────────────┤
│ ⏳ Researching Rust history... (running 2m 15s)              │
│    [Click for details]                                         │
│                                                                 │
│ > New query available here                                    │
│   [Type to start new query while background runs]            │
└─────────────────────────────────────────────────────────────────┘
```

**Max Parallel (3 queries)**:
```
┌─────────────────────────────────────────────────────────────────┐
│ ⚠️  Maximum parallel queries reached (3 running)              │
│                                                                 │
│ Running:                                                       │
│ • Researching Rust... (4m)                                    │
│ • Analyzing codebase... (2m)                                  │
│ • Generating report... (1m)                                   │
│                                                                 │
│ [Wait for slot] [Cancel one] [Queue: run when slot free]    │
└─────────────────────────────────────────────────────────────────┘
```

**Background Task Complete**:
```
┌─────────────────────────────────────────────────────────────────┐
│ 🔔 Background task complete!                                  │
│ "Research Rust history" finished (took 5m 32s)                  │
│                                                                 │
│ [View response] [Dismiss]                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Stream of Consciousness (Reasoning Display)

**Compact State (Default - 3 Lines)**:
```
┌─────────────────────────────────────────────────────────────────┐
│ You: "Research quantum computing applications"                   │
│                                                                 │
│ ⠋ Researching quantum computing... (1m 23s) ~45s remaining      │
│   → Running web_search skill...                                  │
│   → Found 15 sources, reading Nature article...             [⛶] │
│                                                                 │
│ [Input disabled while streaming]                              │
└─────────────────────────────────────────────────────────────────┘
```

**Maximized State**:
```
┌─────────────────────────────────────────────────────────────────┐
│ ⏳ Reasoning Stream - ~45s remaining                   [⛶] [✕] │
├─────────────────────────────────────────────────────────────────┤
│ • 09:00:00 ⠋ Starting research phase                          │
│ • 09:00:02   Breaking down: "quantum computing applications"      │
│ • 09:00:05 ⠋ Running web_search skill                          │
│              Query: "quantum computing applications 2024"         │
│ • 09:00:10 ✓ Found 15 sources                                  │
│ • 09:00:12 ⠋ Reading: Nature - "Quantum advantage in ML"      │
│ • 09:00:25 ✓ Extracted key findings                             │
│ • 09:00:30 ⠋ Analysis phase: Comparing applications...          │
│ • 09:01:15 ⠋ Synthesizing results... (~45s remaining)          │
│                                                                 │
│ [Stop Request]                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Secret Redaction in Stream**:
```
Before redaction:
• 09:00:05 ⠋ Calling API with token sk-abc123xyz

After redaction:
• 09:00:05 ⠋ Calling API with [REDACTED]
```

**Cancel Flow**:
```
User clicks [Stop Request]
    ↓
Stream shows: "⛔ Cancelling..."
    ↓
Agent completes current atomic operation
    ↓
Stream shows: "⛔ Cancelled by user"
    ↓
Final response: "Query was cancelled. Partial results: [...]"
    ↓
Reasoning log saved (with "cancelled" status)
```

**Response Arrives (Auto-Close)**:
```
Before response:
┌─────────────────────────────────────────────────────────────────┐
│ ⠋ Researching quantum computing... (2m 15s) ~30s remaining      │
│   → Synthesizing results...                                   │
│   → Compiling final answer...                                [⛶] │
└─────────────────────────────────────────────────────────────────┘

After response arrives:
┌─────────────────────────────────────────────────────────────────┐
│ Assistant: Here's what I found about quantum computing...       │
│ [Full response content]                                         │
│                                                                 │
│ (2m 15s reasoning, 15 sources, detailed) [View reasoning]      │
└─────────────────────────────────────────────────────────────────┘
```

**Per-Query Toggle**:
```
User: "@agent stream=minimal Research topic"
Agent: "Switching to minimal stream display..."

Minimal stream:
┌─────────────────────────────────────────────────────────────────┐
│ ⠋ Researching... (45s)                                       │
└─────────────────────────────────────────────────────────────────┘
```

### Artifact Save Loop

```
1. USER EDITS TEXT FILE
   ↓
2. HITS SAVE (Ctrl+S)
   ↓
3. LOG ENTRY CREATED
   "💾 recipe.md saved"
   ↓
4. AGENT AUTO-TRIGGERED
   ↓
5. AGENT OPTIONS:
   
   ┌────────────────────────────────────────┐
   │ A. DECLINE                             │
   │    → No action                         │
   │    → No log entry                      │
   ├────────────────────────────────────────┤
   │ B. RESPOND                             │
   │    → Message: "I see you updated       │
   │       the recipe. Would you like       │
   │       me to suggest improvements?"    │
   │    → Added to timeline                 │
   ├────────────────────────────────────────┤
   │ C. REQUEST INPUT                       │
   │    → [FormCard]: "Rate this recipe"    │
   │    → Added to timeline                 │
   │    → Distinct visual treatment         │
   └────────────────────────────────────────┘
```

### Request Input Pattern

**Visual Distinction**:
```
Normal message:
┌──────────────────────────────────────┐
│ Here's what I found...               │
│ [Content]                            │
└──────────────────────────────────────┘
  Grey border

Request input:
┌──────────────────────────────────────┐
│ 🔷 Assistant needs your input        │
├──────────────────────────────────────┤
│ I'd like to know your preference.    │
│                                      │
│ ┌──────────────────────────────────┐   │
│ │ Preferred format:               │   │
│ │ ○ Markdown                      │   │
│ │ ○ Plain text                    │   │
│ │ ○ Rich document                 │   │
│ └──────────────────────────────────┘   │
│                                      │
│           [Submit]  [Dismiss]          │
└──────────────────────────────────────┘
  Blue accent border
  "Assistant needs your input" badge
```

**Lifecycle**:
```
1. Agent creates request_input
2. Added to timeline with 🔷 indicator
3. Also added to canvas as active form
4. User fills and submits
5. Response added as new timeline entry
6. Form marked complete (greyed out)
7. Can be closed by either party
```

### Canvas Tab Interface

```
┌─────────────────────────────────────────────────────────────────┐
│ Canvas                                        [+] [Split View]  │
├─────────────────────────────────────────────────────────────────┤
│ 📄 recipe.md  │ 📊 data.json  │ 🎴 RecipeCard │ ➕              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  # Chocolate Cake Recipe                                        │
│                                                                 │
│  ## Ingredients                                                 │
│  - 2 cups flour                                                 │
│  - 1 cup sugar                                                  │
│  ...                                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
  ^ Active tab has underline
  ^ Click + to add new artifact
  ^ Split view shows two tabs side-by-side
```

### Card Editing

**Text Artifact**:
```
┌─────────────────────────────────────────────────────────────────┐
│ recipe.md                                    [Save] [Discard]  │
├─────────────────────────────────────────────────────────────────┤
│  1  │ # Chocolate Cake Recipe                                  │
│  2  │                                                           │
│  3  │ ## Ingredients                                            │
│  4  │ - 2 cups flour                                            │
│  5  │ - 1 cup sugar                                             │
│     │                                                           │
│     │ [Monaco editor with syntax highlighting]                  │
└─────────────────────────────────────────────────────────────────┘
```

**Card Artifact**:
```
┌─────────────────────────────────────────────────────────────────┐
│ RecipeCard                                   [Save] [Discard]  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Title:        [Chocolate Cake                    ]            │
│                                                                 │
│  Difficulty:   [Easy ▼]                                        │
│                                                                 │
│  Prep Time:    [30] minutes                                    │
│                                                                 │
│  Ingredients:  ┌─────────────────────────────────┐           │
│                  │ - 2 cups flour                  │           │
│                  │ - 1 cup sugar                   │           │
│                  │ - ...                             │           │
│                  └─────────────────────────────────┘           │
│                  [Rich JSON editor with validation]            │
│                                                                 │
│  Tags:         [dessert] [chocolate] [+ Add tag]              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Collapsed Saves

```
Normal view:
[9:00] You: Hello
[9:01] 💾 file1.md saved
[9:02] 💾 file2.md saved
[9:03] 💾 file3.md saved
[9:04] You: Done editing

Collapsed view (when no other history between):
[9:00] You: Hello
[9:01] 💾 3 files saved (file1.md, file2.md, file3.md)
[9:04] You: Done editing

[Click to expand and see individual saves]
```

## Platform Adaptations

### Desktop (macOS) - Full Experience

```
┌─────────────────────────────────────────────────────────────────┐
│ Attaché                                            [—] [□] [×]│
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────┐  ┌──────────────────────────────┐ │
│  │                          │  │ Canvas                       │ │
│  │   Chat Timeline          │  │ ┌──────────────────────────┐ │ │
│  │                          │  │ │ 📄 file.md  │ 🎴 Card   │ │ │
│  │   [Messages scroll      │  │ ├──────────────────────────┤ │ │
│  │    upwards]              │  │ │                          │ │ │
│  │                          │  │ │   [Active artifact         │ │ │
│  │   [9:00] You: Hello      │  │ │    content with          │ │ │
│  │   [9:01] Asst: Hi!     │  │ │    full editor]          │ │ │
│  │                          │  │ │                          │ │ │
│  │                          │  │ └──────────────────────────┘ │ │
│  │                          │  └──────────────────────────────┘ │
│  │                          │                                   │
│  │ [Type message...     ]  │  [Drag divider to resize]        │
│  └──────────────────────────┘                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Keyboard Shortcuts**:
- `Cmd+N`: New artifact
- `Cmd+S`: Save current artifact
- `Cmd+W`: Close tab
- `Cmd+1,2,3...`: Switch to tab N
- `Cmd+Shift+P`: Enter Cone of Silence
- `Cmd+F`: Search timeline
- `Cmd+K`: Quick action palette

### Web - Full Experience

Same layout as desktop but:
- Browser security warnings for secrets
- Service Worker for offline support
- PWA install prompt
- WebRTC for... (not used, SSE only)

### Mobile - Adapted

```
┌─────────────────────────────────────────┐
│ Attaché                         [≡]   │
├─────────────────────────────────────────┤
│                                         │
│  [Chat Timeline]                        │
│                                         │
│  ┌─────────────────────────────────────┐ │
│  │ What can I help you with?         │ │
│  │                                   │ │
│  │ [Recent locations detected -      │ │
│  │  tap to share with agent]         │ │
│  └─────────────────────────────────────┘ │
│                                         │
│  [9:00] You: Hello                     │
│  [9:01] Assistant: Hi!                 │
│                                         │
│  [Swipe up for canvas]                │
│                                         │
│  ┌─────────────────────────────────────┐ │
│  │ Type message...          [🎙️] [➤] │ │
│  └─────────────────────────────────────┘ │
│                                         │
├─────────────────────────────────────────┤
│ [💬]    [🎨]      [📁]      [⚙️]       │
│ Chat   Canvas   Files   Settings      │
└─────────────────────────────────────────┘
```

**Mobile Canvas**:
- Simplified artifact view
- Key cards only (filter out complex visualizations)
- Swipe between artifacts
- Touch-optimized editing (simplified controls)

**Mobile Features**:
- Location updates (background, battery-optimized)
- Push notifications for agent requests
- Voice input (speech-to-text)
- Quick replies from notifications

### TUI - Filesystem Canvas + Coding Assistant

**TUI is not a lesser experience—it's a different paradigm optimized for coding workflows.**

**Main Screen**:
```
┌─────────────────────────────────────────────────────────────────┐
│ Attaché TUI - ~/projects/myapp                           [?]  │
├─────────────────────────────────────────────────────────────────┤
│ 💬 Chat  │  📁 Files  │  ⚡ Actions  │  ⚙️ Config  │  🔐 Secrets  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ ┌────────────────────────────────────────────────────────────┐│
│ │ Chat                                                       ││
│ │ [9:00] You: Hello                                          ││
│ │ [9:01] Assistant: Hi! I see this is a Rust project.      ││
│ │          I can help you with coding.                       ││
│ │                                                            ││
│ │ [9:05] You: !cargo test                                    ││
│ │ [9:06] 🖥️ cargo test (running...)                        ││
│ │          running 3 tests...                                ││
│ │          test test_main ... ok                             ││
│ │          test test_utils ... FAILED                        ││
│ │          [Output sent to agent]                            ││
│ │                                                            ││
│ │ [9:07] Assistant: I see the test failed. Let me check      ││
│ │          the code...                                       ││
│ │ [9:08] 💾 src/lib.rs saved (by agent)                     ││
│ │                                                            ││
│ └────────────────────────────────────────────────────────────┘│
│                                                                 │
│ > !cargo test                                               │    │
│   [Tab for autocomplete] [Ctrl+E to edit files]             │    │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│ [c]hat │ [f]iles │ [a]ctions │ [C]onfig │ [s]ecrets │ [q]uit  │
│ [!]command │ [e]dit │ [r]un default task │ [S]ync: 2 pending │
└─────────────────────────────────────────────────────────────────┘
```

**Files Mode (Canvas)**:
```
┌─────────────────────────────────────────────────────────────────┐
│ 📁 Files - ~/projects/myapp (13 files, 2 modified, 1 unsynced)  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  src/                                                          │
│  ├── main.rs         ✓ (synced)                              │
│  ├── lib.rs          ✓ (synced)                              │
│  └── utils/                                                    │
│      └── mod.rs      ✗ (modified, unsynced)                   │
│  tests/                                                        │
│  └── test_main.rs    ✓ (synced)                              │
│  Cargo.toml          ✗ (modified, synced)                     │
│  README.md           ✓ (synced)                              │
│  .attache/                                                     │
│  └── config.yaml     ✓ (local config)                        │
│                                                                │
│  Legend: ✓ synced  ✗ modified  ⚠️ pending sync              │
│                                                                │
├─────────────────────────────────────────────────────────────────┤
│ [↑/↓] Navigate  [e]dit  [d]iff  [r]efresh  [b]ack           │
└─────────────────────────────────────────────────────────────────┘
```

**Editing Flow**:
```
User in Files mode, selects main.rs, presses [e]
    ↓
Suspend TUI UI
    ↓
Launch $EDITOR with main.rs
    ↓
User edits and saves (:wq in vim)
    ↓
Editor exits
    ↓
Resume TUI
    ↓
Filesystem watcher detects change
    ↓
Check: Is this an agent change? No
    ↓
Mark as "modified, unsynced"
    ↓
If online: Auto-sync after debounce
If offline: Queue for batch
    ↓
Show: "✗ src/main.rs (modified, synced)"
```

**Command Execution**:
```
User types: !cargo test
    ↓
Parse: "cargo test"
    ↓
Check allowlist: "cargo test" is allowlisted? Yes
    ↓
Execute in project directory
    ↓
Stream output to chat
    ↓
Capture full output
    ↓
Send to agent as message
    ↓
Agent decides what to do with it
```

**Non-Whitelisted Command**:
```
User types: !rm -rf /
    ↓
Parse: "rm"
    ↓
Check allowlist: Not found
    ↓
allow_all is false
    ↓
Prompt: "⚠️ 'rm' is not allowlisted. Approve? [y/N]"
    ↓
User: N
    ↓
Command rejected
    ↓
No execution
```

**Offline Mode**:
```
[Connection lost - working offline]

User edits src/main.rs
    ↓
File saved
    ↓
Change detected
    ↓
Offline: Queue for batch
    ↓
Show: "⚠️ src/main.rs (modified, pending sync)"
    ↓

[Connection restored]
    ↓
Sync batch to agent:
    "2 files modified while offline:
     - src/main.rs (diff: ...)
     - Cargo.toml (diff: ...)"
    ↓
Show: "✓ Synced 2 files"
```

**TUI Keyboard Navigation**:
- `↑/↓` or `j/k`: Navigate
- `Enter`: Select / Send message
- `Tab`: Autocomplete / Next field
- `c`: Switch to Chat mode
- `f`: Switch to Files mode
- `a`: Actions menu
- `C`: Config menu
- `s`: Secrets panel
- `!`: Focus command input (type !command)
- `e`: Edit selected file in $EDITOR
- `d`: Show diff for modified file
- `r`: Run default task (from config)
- `S`: Force sync pending changes
- `q`: Quit
- `?`: Help
- `Ctrl+C`: Cancel current operation
- `Ctrl+L`: Redraw screen

**TUI Command Prefixes**:
- `!command` - Execute allowlisted shell command
- `:command` - TUI internal command
- `@agent` - Direct message to agent
- `#file` - Reference file in message

### Speech Interface

**Input (Push-to-Talk)**:
```
Desktop/Mobile:
Hold [🎙️] button → Speak → Release → Transcribe

TUI:
Press [Space] (while not typing) → Speak → Release
```

**Output (TTS)**:
- Agent responses spoken aloud
- Can be disabled per-message or globally
- Visual indicator when speaking

## Error Handling

### Network Error

```
┌─────────────────────────────────────────────────────────────────┐
│ ⚠️  Connection Lost                                             │
│                                                                 │
│ Attempting to reconnect... [○ ○ ○]                             │
│                                                                 │
│ [Retry Now]  [Work Offline]                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Skill Error

```
┌─────────────────────────────────────────────────────────────────┐
│ ❌ Skill Execution Failed                                        │
│                                                                 │
│ The "web_search" skill encountered an error:                    │
│ "API rate limit exceeded"                                       │
│                                                                 │
│ [Retry]  [Try Alternative Skill]  [Dismiss]                      │
└─────────────────────────────────────────────────────────────────┘
```

### LLM Unavailable

```
┌─────────────────────────────────────────────────────────────────┐
│ ⚠️  Agent Temporarily Unavailable                                │
│                                                                 │
│ All LLM providers are currently unavailable.                   │
│ Your messages are queued and will be processed when            │
│ service resumes.                                               │
│                                                                 │
│ [Queue: 3 messages waiting]                                    │
│                                                                 │
│ [Switch to Offline Mode]                                        │
└─────────────────────────────────────────────────────────────────┘
```

### Multiple Variant Error (FactForge carryover)

Not applicable to Attaché (different system).

---

*See also: [Overview](./README.md), [Specification](./SPEC.md), [Architecture](./ARCHITECTURE.md), [Test Cases](./TEST-CASES.md)*

---

*Document Version: 1.0*  
*Part of the [Attaché Product Requirements](./README.md)*
