# FactForge Test Cases

> **Navigation**: [Overview](./README.md) | [Specification](./SPEC.md) | [Architecture](./ARCHITECTURE.md) | Test Cases

## Table of Contents

- [Config Parsing Tests](#config-parsing-tests)
- [Fact Collection Tests](#fact-collection-tests)
- [Templating Tests](#templating-tests)
- [Variant Resolution Tests](#variant-resolution-tests)
- [Action Execution Tests](#action-execution-tests)
- [Dependency Tests](#dependency-tests)
- [Integration Test Scenarios](#integration-test-scenarios)
- [Edge Cases](#edge-cases)

## Config Parsing Tests

| ID | Test Case | Input | Expected Result |
|----|-----------|-------|-----------------|
| CFG-001 | Valid minimal config | `facts: []\nsteps: []` | Parse success |
| CFG-002 | Missing facts field | `steps: []` | Parse error (missing facts) |
| CFG-003 | Missing steps field | `facts: []` | Parse error (missing steps) |
| CFG-004 | Valid intrinsic fact | `facts: [{name: os, type: intrinsic}]` | Parse success |
| CFG-005 | Valid env fact | `facts: [{name: home, type: env}]` | Parse success |
| CFG-006 | Valid env fact with explicit variable | `facts: [{name: shell, type: env, variable: SHELL}]` | Parse success |
| CFG-007 | Valid command fact | `facts: [{name: has_brew, type: command, run: "which brew"}]` | Parse success |
| CFG-008 | Command fact with output field | `facts: [{name: has_brew, type: command, run: "which brew", output: status}]` | Parse success |
| CFG-009 | Command fact with type conversion | `facts: [{name: node_ver, type: command, run: "node --version", as: semver}]` | Parse success |
| CFG-010 | Invalid fact type | `facts: [{name: x, type: invalid}]` | Parse error |
| CFG-011 | Step without variants | `steps: [{name: test}]` | Parse error (missing variants) |
| CFG-012 | Variant without when | `steps: [{name: test, variants: [{actions: []}]}]` | Parse error (missing when) |
| CFG-013 | Variant without actions | `steps: [{name: test, variants: [{when: {}}]}]` | Parse error (missing actions) |
| CFG-014 | Valid step with needs | `steps: [{name: a}, {name: b, needs: [a], variants: [...]}]` | Parse success |
| CFG-015 | Custom saved facts path | `save_facts_to: ~/.cache/facts.json` | Parse success, path stored |
| CFG-016 | Empty YAML | `""` | Parse error |
| CFG-017 | Invalid YAML syntax | `facts: [` | Parse error with location |
| CFG-018 | Duplicate step names | Two steps named "install" | Parse error (duplicate) |

## Fact Collection Tests

| ID | Test Case | Setup | Expected Result |
|----|-----------|-------|-----------------|
| FCT-001 | Collect intrinsic os | Run on Linux | `os` = "linux" |
| FCT-002 | Collect intrinsic arch | Run on ARM64 | `arch` = "arm64" |
| FCT-003 | Collect intrinsic user | Current user "alice" | `user` = "alice" |
| FCT-004 | Collect intrinsic uid | Current UID 1000 | `uid` = 1000 (number) |
| FCT-005 | Collect env fact | `$HOME=/home/alice` | `home` = "/home/alice" |
| FCT-006 | Collect env fact with explicit var | Variable `SHELL=/bin/bash` | `shell` = "/bin/bash" |
| FCT-007 | Missing env var | Variable `UNDEFINED_VAR` not set | Error: env var not found |
| FCT-008 | Command fact stdout | `run: "echo hello"`, output: stdout | `cmd.stdout` = "hello", `cmd` = "hello" |
| FCT-009 | Command fact stderr | `run: "echo error >&2"`, output: stderr | `cmd.stderr` = "error", `cmd` = "error" |
| FCT-010 | Command fact status | `run: "exit 42"`, output: status | `cmd.status` = 42, `cmd` = 42 |
| FCT-011 | Command fact all outputs | Any command | Creates `cmd.stdout`, `cmd.stderr`, `cmd.status` |
| FCT-012 | Command fact trim | `run: "echo '  hello  '"` | `cmd.stdout` = "hello" (trimmed) |
| FCT-013 | Command fact as number | `run: "echo 42"`, as: number | `cmd` = 42 (number type) |
| FCT-014 | Command fact as semver | `run: "echo 1.2.3"`, as: semver | `cmd` = Semver(1.2.3) |
| FCT-015 | Invalid semver | `run: "echo not-a-version"`, as: semver | Error: invalid semver |
| FCT-016 | Command fact fails | `run: "false"` | `cmd.status` = 1, other fields empty |
| FCT-017 | Factforge version | Always | `factforge_version` = current version |
| FCT-018 | Fact caching | Same command declared twice | Command runs once, cached |
| FCT-019 | Fact order | Declare env before command | Env collected before command |
| FCT-020 | Missing fact reference | Template uses `{{undefined}}` | Error: fact not found |

## Templating Tests

| ID | Test Case | Template | Facts/Env | Expected Result |
|----|-----------|----------|-----------|-----------------|
| TMP-001 | Simple fact interpolation | `"{{os}}"` | `os=linux` | "linux" |
| TMP-002 | Multiple facts | `"{{os}}-{{arch}}"` | `os=linux, arch=arm64` | "linux-arm64" |
| TMP-003 | Env var interpolation | `"${HOME}/bin"` | `HOME=/home/alice` | "/home/alice/bin" |
| TMP-004 | Combined interpolation | `"${HOME}/{{tool}}"` | `HOME=/home, tool=ripgrep` | "/home/ripgrep" |
| TMP-005 | Nested braces escape | `"{{'{{'}}os{{'}}'}}"` | | "{{os}}" |
| TMP-006 | Missing fact | `"{{missing}}"` | | Error: fact not found |
| TMP-007 | Missing env var | `"${UNDEFINED}"` | | Error: env var not found |
| TMP-008 | Empty fact value | `"{{empty}}"` | `empty=""` | "" |
| TMP-009 | URL template | `"https://example.com/{{arch}}"` | `arch=x86_64` | "https://example.com/x86_64" |
| TMP-010 | Path template | `"${HOME}/.config/{{app}}"` | `HOME=/home, app=nvim` | "/home/.config/nvim" |
| TMP-011 | Command substitution (not supported) | `"$(uname)"` | | Error: invalid syntax |
| TMP-012 | Semver in template | `"{{node_version}}"` | `node_version=Semver(20.0.0)` | "20.0.0" |
| TMP-013 | Number in template | `"{{count}}"` | `count=42` | "42" |
| TMP-014 | Complex nested | `"${XDG_CONFIG_HOME:-${HOME}/.config}/{{tool}}"` | | Support default values? |

## Variant Resolution Tests

| ID | Test Case | Facts | Variants | Expected Result |
|----|-----------|-------|----------|-----------------|
| RES-001 | Exact match | `os=linux` | `[{when: {os: linux}}]` | Match variant 1 |
| RES-002 | No match (warning) | `os=macos` | `[{when: {os: linux}}]` | No match, warning issued, step skipped |
| RES-003 | Multiple matches (error) | `os=linux, arch=arm64` | `[{when: {os: linux}}, {when: {arch: arm64}}]` | Error: MultipleMatchingVariants |
| RES-004 | AND logic match | `os=linux, arch=arm64` | `[{when: {os: linux, arch: arm64}}]` | Match variant 1 |
| RES-005 | AND logic no match | `os=linux, arch=x86_64` | `[{when: {os: linux, arch: arm64}}]` | No match, step skipped |
| RES-006 | Number match (shorthand) | `status=0` | `[{when: {status: 0}}]` | Match |
| RES-007 | Number mismatch | `status=1` | `[{when: {status: 0}}]` | No match, step skipped |
| RES-007a | Number greater than | `count=10` | `[{when: {count: ">5"}}]` | Match |
| RES-007b | Number greater than false | `count=3` | `[{when: {count: ">5"}}]` | No match |
| RES-007c | Number greater than equal | `count=5` | `[{when: {count: ">=5"}}]` | Match |
| RES-007d | Number less than | `count=3` | `[{when: {count: "<5"}}]` | Match |
| RES-007e | Number less than equal | `count=5` | `[{when: {count: "<=5"}}]` | Match |
| RES-007f | Number explicit equality | `count=42` | `[{when: {count: "==42"}}]` | Match |
| RES-007g | Number invalid expression | `count=5` | `[{when: {count: ">>5"}}]` | Error: invalid number expression |
| RES-008 | Missing fact in criteria | `os=linux` (no arch) | `[{when: {os: linux, arch: arm64}}]` | No match (arch missing), step skipped |
| RES-009 | Semver exact match (shorthand) | `version=Semver(1.2.3)` | `[{when: {version: "1.2.3"}}]` | Match |
| RES-009a | Semver explicit equality | `version=Semver(1.2.3)` | `[{when: {version: "==1.2.3"}}]` | Match |
| RES-010 | Semver >= match | `version=Semver(20.0.0)` | `[{when: {version: ">=18.0.0"}}]` | Match |
| RES-010a | Semver >= exact match | `version=Semver(18.0.0)` | `[{when: {version: ">=18.0.0"}}]` | Match |
| RES-011 | Semver >= no match | `version=Semver(16.0.0)` | `[{when: {version: ">=18.0.0"}}]` | No match, step skipped |
| RES-011a | Semver > match | `version=Semver(20.0.0)` | `[{when: {version: ">18.0.0"}}]` | Match |
| RES-011b | Semver > no match (equal) | `version=Semver(18.0.0)` | `[{when: {version: ">18.0.0"}}]` | No match |
| RES-011c | Semver <= match | `version=Semver(16.0.0)` | `[{when: {version: "<=18.0.0"}}]` | Match |
| RES-011d | Semver < match | `version=Semver(16.0.0)` | `[{when: {version: "<18.0.0"}}]` | Match |
| RES-011e | Semver < no match (equal) | `version=Semver(18.0.0)` | `[{when: {version: "<18.0.0"}}]` | No match |
| RES-012 | Semver ^ match | `version=Semver(18.5.0)` | `[{when: {version: "^18.0.0"}}]` | Match |
| RES-013 | Semver ^ no match | `version=Semver(19.0.0)` | `[{when: {version: "^18.0.0"}}]` | No match, step skipped |
| RES-014 | Semver ~ match | `version=Semver(18.5.0)` | `[{when: {version: "~18.0.0"}}]` | Match |
| RES-015 | Empty when clause | `os=linux` | `[{when: {}}]` | Always matches |
| RES-016 | Invalid semver expression | `version=Semver(1.0.0)` | `[{when: {version: ">>1.0.0"}}]` | Error: invalid semver expr |
| RES-017 | Case sensitivity | `os=Linux` | `[{when: {os: linux}}]` | No match (case sensitive), step skipped |
| RES-018 | String vs number | `count="42"` | `[{when: {count: 42}}]` | No match (type mismatch), step skipped |
| RES-019 | Zero matches with dependencies | Step A matches, Step B (needs A) has no match | Step A runs, Step B skipped with warning, no cascade |
| RES-020 | All steps no match | 3 steps, none match | All skipped, warnings issued, exit code 0 |

## Action Execution Tests

| ID | Test Case | Action | Setup | Expected Result |
|----|-----------|--------|-------|-----------------|
| ACT-001 | Download success | `download` url, dest | Valid URL | File downloaded |
| ACT-002 | Download with SHA256 | `download` + sha256 | Valid hash | Success |
| ACT-003 | Download wrong SHA256 | `download` + wrong hash | | Error: ChecksumFailed (exit 6) |
| ACT-004 | Download with GPG | `download` + gpg key | Valid sig | Success |
| ACT-005 | Download wrong GPG | `download` + wrong key | | Error: GpgFailed (exit 7) |
| ACT-006 | Download idempotent exists | `download`, file exists | | Skipped |
| ACT-007 | Download idempotent different | `download` + sha256, file differs | | Re-download |
| ACT-008 | Shell success | `shell` run: "echo hi" | | Success |
| ACT-009 | Shell failure | `shell` run: "exit 1" | | Error: NonZeroExit |
| ACT-010 | Shell with cwd | `shell` run, cwd: "/tmp" | | Runs in /tmp |
| ACT-011 | Shell with env | `shell` run, env: {KEY: val} | | $KEY set |
| ACT-012 | Extract tar.gz | `extract` src, dest | Valid archive | Extracted |
| ACT-013 | Extract zip | `extract` src: "file.zip" | Valid zip | Extracted |
| ACT-014 | Extract strip components | `extract` with strip_components: 1 | Archive with prefix | Prefix stripped |
| ACT-015 | Extract idempotent | `extract`, files exist | | Skipped |
| ACT-016 | Write file | `write_file` path, content | | File created |
| ACT-017 | Write file with mode | `write_file` + mode: "0600" | | File with perms |
| ACT-018 | Write file idempotent | `write_file`, file exists with same content | | Skipped |
| ACT-019 | Write file overwrite | `write_file`, file exists with diff content | | Overwritten |
| ACT-020 | Path validation blocked | `download` dest: "/usr/bin/tool" | | Error: PathValidation (exit 8) |
| ACT-021 | Path validation allowed | `download` dest: "${HOME}/bin/tool" | | Success |
| ACT-022 | Template in URL | `download` url: "https://{{arch}}" | `arch=x86_64` | URL resolved |
| ACT-023 | Template in dest | `download` dest: "${HOME}/{{tool}}" | `HOME=/home, tool=rg` | Path resolved |

## Dependency Tests

| ID | Test Case | Steps | Dependencies | Expected Order |
|----|-----------|-------|--------------|--------------|
| DEP-001 | Simple dependency | A, B needs A | B→A | A, B |
| DEP-002 | Chain dependency | A, B needs A, C needs B | C→B→A | A, B, C |
| DEP-003 | Diamond dependency | A, B needs A, C needs A, D needs B,C | D→(B,C)→A | A, B, C, D |
| DEP-004 | Multiple dependencies | A, B needs [A, C], C | B→(A,C) | A, C, B |
| DEP-005 | Unknown dependency | A needs B (B undefined) | | Error: UnknownDependency (exit 1) |
| DEP-006 | Circular dependency | A needs B, B needs A | | Error: CircularDependency (exit 9) |
| DEP-007 | Self dependency | A needs A | | Error: CircularDependency (exit 9) |
| DEP-008 | Indirect cycle | A→B→C→A | | Error: CircularDependency (exit 9) |
| DEP-009 | Dependency failure propagation | A (fails), B needs A | | A: Failed, B: NotAttempted |
| DEP-010 | Independent failure | A (fails), B (no dep), C needs A | | A: Failed, B: Success, C: NotAttempted |
| DEP-011 | No dependencies | A, B, C (none) | | Any order (A,B,C) |

## Integration Test Scenarios

### End-to-End: Install Tool on macOS with Homebrew

**Config**:
```yaml
facts:
  - name: os
    type: intrinsic
  - name: has_brew
    type: command
    run: "which brew"
    output: status
    as: number

steps:
  - name: install_ripgrep
    variants:
      - when:
          os: macos
          has_brew: 0
        actions:
          - type: shell
            run: "brew install ripgrep"
      - when:
          os: linux
        actions:
          - type: shell
            run: "apt-get install -y ripgrep"
```

**Steps**:
1. Run on macOS with Homebrew installed
2. **Verify**: Fact collection succeeds (os=macos, has_brew=0)
3. **Verify**: Variant resolution picks macOS variant
4. **Verify**: Shell action executes `brew install ripgrep`
5. **Verify**: Report shows: 1 successful
6. **Verify**: Facts saved to default location
7. **Verify**: Exit code 0

### End-to-End: No Matching Variant

**Config**:
```yaml
facts:
  - name: os
    type: intrinsic

steps:
  - name: install_windows_tool
    variants:
      - when:
          os: windows
        actions:
          - type: shell
            run: "echo windows"
```

**Steps**:
1. Run on macOS
2. **Verify**: Fact collection succeeds (os=macos)
3. **Verify**: Warning: No matching variant for "install_windows_tool"
4. **Verify**: Report shows: 1 skipped (no matching variant)
5. **Verify**: Exit code 0 (success, nothing to do on this platform)
6. **Verify**: Facts still saved (successful run)

### End-to-End: Multiple Matching Variants

**Config**:
```yaml
facts:
  - name: os
    type: intrinsic
  - name: arch
    type: intrinsic

steps:
  - name: install_tool
    variants:
      - when:
          os: macos
        actions: [...]
      - when:
          arch: arm64
        actions: [...]
```

**Steps**:
1. Run on macOS ARM64
2. **Verify**: Both variants match (os=macos AND arch=arm64)
3. **Verify**: Error: Multiple matching variants (2) for "install_tool"
4. **Verify**: Exit code 3

### End-to-End: Dependency Chain

**Config**:
```yaml
facts:
  - name: os
    type: intrinsic

steps:
  - name: install_homebrew
    variants:
      - when:
          os: macos
        actions:
          - type: shell
            run: "/bin/bash -c '$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)'"
  - name: install_ripgrep
    needs: [install_homebrew]
    variants:
      - when:
          os: macos
        actions:
          - type: shell
            run: "brew install ripgrep"
```

**Steps**:
1. Run on macOS without Homebrew
2. **Verify**: install_homebrew runs first
3. **Verify**: install_ripgrep runs after (depends on first)
4. **Verify**: Report shows: 2 successful
5. **Verify**: Exit code 0

### End-to-End: Dependency Failure

**Config**:
```yaml
steps:
  - name: step_a
    variants:
      - when: {}
        actions:
          - type: shell
            run: "exit 1"
  - name: step_b
    needs: [step_a]
    variants:
      - when: {}
        actions:
          - type: shell
            run: "echo success"
```

**Steps**:
1. Run config
2. **Verify**: step_a fails (exit 1)
3. **Verify**: step_b not attempted (dependency failed)
4. **Verify**: Report shows: 1 failed, 1 not attempted
5. **Verify**: Exit code 4

### End-to-End: Check Command with Changes

**Setup**:
1. Run factforge on Linux x86_64
2. Facts saved with os=linux, arch=x86_64
3. Switch to macOS ARM64 machine (or simulate)

**Steps**:
1. Run `factforge check config.yaml`
2. **Verify**: Detects changes (os: linux→macos, arch: x86_64→arm64)
3. **Verify**: Prompts: "Rerun with new facts? [Y/n]"
4. Select "n"
5. **Verify**: Message: "Skipped. Run `factforge run config.yaml` to update."
6. **Verify**: Exit code 0

### End-to-End: Check Command --warn-only

**Setup**: Same as above

**Steps**:
1. Run `factforge check --warn-only config.yaml`
2. **Verify**: Outputs warning about changed facts
3. **Verify**: Shows command to run
4. **Verify**: No interactive prompt
5. **Verify**: Exit code 0

### End-to-End: Plan Command

**Config**:
```yaml
facts:
  - name: os
    type: intrinsic

steps:
  - name: install_tool
    variants:
      - when:
          os: macos
        actions:
          - type: download
            url: "https://example.com/tool-macos"
            dest: "${HOME}/bin/tool"
```

**Steps**:
1. Run `factforge plan config.yaml` on macOS
2. **Verify**: Shows resolved facts (os=macos)
3. **Verify**: Shows selected variant (when: os=macos)
4. **Verify**: Shows resolved actions (URL and path expanded)
5. **Verify**: Does NOT download file
6. **Verify**: Exit code 0

### End-to-End: From Stdin

**Steps**:
1. Run: `cat config.yaml | factforge run --from-stdin`
2. **Verify**: Parses YAML from stdin
3. **Verify**: Executes normally
4. **Verify**: Exit code 0

## Edge Cases

| ID | Test Case | Description | Expected Behavior |
|----|-----------|-------------|-------------------|
| EDG-001 | Empty config | `facts: []\nsteps: []` | Success (no-op) |
| EDG-002 | Very long command output | Command outputs 1MB | Handle gracefully, possibly truncate |
| EDG-003 | Binary output in command | Command outputs binary | Store as string (may be lossy) |
| EDG-004 | Unicode in facts | Fact values with emoji/unicode | Preserve correctly |
| EDG-005 | Spaces in paths | `${HOME}/My Documents/tool` | Handle correctly with quotes |
| EDG-006 | Special chars in URLs | URL with `?&=` | Encode properly or preserve |
| EDG-007 | Concurrent runs | Two factforge instances | Lock file or error |
| EDG-008 | Interrupted download | SIGINT during download | Clean up temp file |
| EDG-009 | Full disk | Download to full disk | Error with clear message |
| EDG-010 | Network timeout | Unreachable URL | Error: connection timeout |
| EDG-011 | Template injection | Fact value: `"{{other}}"` | Don't re-template (one-pass) |
| EDG-012 | Recursive template | Fact depends on template of itself | Error: circular reference |
| EDG-013 | Very deep dependency chain | 1000 steps in chain | Handle without stack overflow |
| EDG-014 | Many variants | Step with 1000 variants | Efficient matching |
| EDG-015 | Large config | 10MB YAML file | Parse efficiently |
| EDG-016 | Permission denied | Can't write to dest | Error: permission denied |
| EDG-017 | Symlink in path | Dest is a symlink | Follow or error? (configurable?) |
| EDG-018 | Relative paths in config | `dest: "./tool"` | Resolve relative to config file |
| EDG-019 | Home directory expansion | `~` vs `${HOME}` | Both work |
| EDG-020 | Windows paths | `C:\\Users\\...` | Error: Windows not supported (yet, exit 1) |

---

*See also: [Overview](./README.md), [Specification](./SPEC.md), [Architecture](./ARCHITECTURE.md)*

---

*Document Version: 1.0*  
*Part of the [FactForge Product Requirements](./README.md)*
