---
name: source-completeness
description: Validates that all user-facing surfaces discovered in source code are captured in the behavioral specs. Catches coverage gaps where the analysis pipeline missed features, tools, env vars, CLI flags, or other behavioral interfaces. Run AFTER Layer 3 specs are written, BEFORE sanitization.
---

# Source-to-Spec Completeness Validation

## Why This Exists

The analysis pipeline (Layers 1-3) reads source code and produces behavioral specs. But it can miss things:
- Registered commands or tools defined in source but not in the tool catalog
- Environment variables read by the code but not documented
- CLI flags defined in the argument parser but not cataloged
- Subcommands registered dynamically or conditionally
- Config keys accessed in settings but not listed in the config spec
- Events dispatched but not inventoried
- Error categories defined but not documented

The fidelity validation (Layer 7) only checks raw→clean. This skill checks **source→raw** — it goes back to the actual source code and verifies that every discoverable user-facing surface was captured in the specs.

## When to Run

Run AFTER Layer 3 deep documentation is complete, ideally as part of Gate 1 or as a pre-Gate-1 completeness check. If gaps are found, dispatch deep-dive agents to cover the missing areas before proceeding.

Also run AFTER Layer 7 fidelity validation to catch anything the fidelity check couldn't find (because it was never in the raw specs).

## The Checks

Search patterns depend on target shape. For a single-file bundle, grep the bundle directly. For a source tree, grep recursively across the source root. Patterns also depend on the target language's idioms for registering each surface — the examples below give common patterns for several languages; adjust to the actual target.

### Check 1: Tool / Command Registration Completeness

Applies to targets that expose a registered command, tool, or handler surface (CLIs with subcommands, plugin hosts, RPC servers with method registration).

**Source extraction:** grep for the registration pattern used by the target. Examples:

```bash
# Named-object pattern (common in JS/TS/Go handler registration)
grep -roE 'name:\s*"[A-Z][a-zA-Z]+"' <source-root> | sort -u

# Decorator-based registration (Python)
grep -roE '@(command|tool|handler|register)\(' <source-root>

# Macro-based registration (Rust)
grep -roE '#\[(command|tool|handler)' <source-root>

# Assigned-constant pattern (JS/TS)
grep -roE 'var [a-zA-Z0-9_]+="[A-Z][a-zA-Z]+"' <source-root> | grep -v 'Exception\|Error\|Element'
```

**Spec extraction:** every registered name in the tool/command catalog section of the spec.

**Diff:** every registered name in source must appear in the spec. Missing names are P0-CRITICAL gaps.

### Check 2: Environment Variable Completeness

**Source extraction:** patterns by language:

```bash
# Node/TypeScript
grep -roE 'process\.env\.[A-Z_][A-Z0-9_]*' <source-root> | sed 's/.*process\.env\.//' | sort -u
# Python
grep -roE 'os\.environ(\.get)?\(?"[A-Z_][A-Z0-9_]*"' <source-root> | grep -oE '[A-Z_][A-Z0-9_]+' | sort -u
# Go
grep -roE 'os\.Getenv\("[A-Z_][A-Z0-9_]*"\)' <source-root> | grep -oE '[A-Z_][A-Z0-9_]+' | sort -u
# Rust
grep -roE 'env::var\("[A-Z_][A-Z0-9_]*"\)' <source-root> | grep -oE '[A-Z_][A-Z0-9_]+' | sort -u
```

**Spec extraction:** all env var names from the environment contract spec.

**Diff:** categorize missing env vars:
- User-facing (`DATABASE_*`, `AWS_*`, `GCP_*`, `LOG_*`, and similar) → P0: must document
- Internal-only (test vars, build vars) → P2: note but don't require
- Standard (`HOME`, `PATH`, `SHELL`, `TERM`) → ignore

### Check 3: CLI Flag Completeness

**Source extraction:**

```bash
# Quoted flag strings — works across most CLI frameworks
grep -roE '"--[a-z][a-z0-9-]+"' <source-root> | sort -u

# Framework-specific definitions:
#   Rust/clap:    .arg("--flag-name")
#   Python/argparse: add_argument("--flag-name")
#   Go/flag:      flag.StringVar(&v, "--flag-name", ...)
grep -roE '\.arg\("--[a-z-]+"|add_argument\("--[a-z-]+"|flag\.\w+Var\([^,]+,\s*"--[a-z-]+"' <source-root>
```

**Spec extraction:** every `--flag` name in the CLI contract spec.

**Diff:** every flag must appear. Hidden flags should be documented as hidden.

### Check 4: Subcommand Completeness

Applies to CLIs or REPLs with a subcommand surface.

**Source extraction:** target-dependent. For subcommand CLIs, grep for the subcommand registration or dispatch table (clap subcommands, Cobra, argparse subparsers). For REPL-style interfaces, grep for the command-prefix pattern (e.g., `/help`, `:quit`).

**Spec extraction:** every subcommand from the CLI interface spec.

**Diff:** every registered subcommand must appear.

### Check 5: Event Dispatch Completeness

Applies to targets that dispatch named events — common shapes include observer patterns, signals, message buses, and webhook emission.

**Source extraction:** grep for event dispatch calls and event-type string constants. Common shapes: `emit("event-name", ...)`, `dispatch(EventType.X)`, `pubsub.publish("...", ...)`, signal/slot registrations, `raise EventName(...)`.

**Spec extraction:** every event type documented in the extensibility or eventing spec.

**Diff:** every dispatched event type must be documented.

### Check 6: Config Key Completeness

**Source extraction:** Search for settings property accesses and config key strings.

**Spec extraction:** Extract all config keys from the configuration spec.

**Diff:** User-configurable keys must appear. Internal state keys are P2.

### Check 7: Error Category Completeness

**Source extraction:** Search for error class definitions, error type enums, and error message constants.

**Spec extraction:** Extract all documented error categories.

**Diff:** User-visible errors must be documented.

## Report Format

```markdown
# Source-to-Spec Completeness Report

## Summary
| Check | Source Count | Spec Count | Missing | Extra |
|-------|-------------|------------|---------|-------|
| Registered commands/tools | N | N | N | N |
| Env Vars | N | N | N | N |
| CLI Flags | N | N | N | N |
| Subcommands | N | N | N | N |
| Events dispatched | N | N | N | N |
| Config Keys | N | N | N | N |
| Error Categories | N | N | N | N |

## P0-CRITICAL Gaps (must fix)
[List all missing user-facing surfaces]

## P1 Gaps (should fix)
[List missing non-critical surfaces]

## P2 Gaps (advisory)
[List internal-only surfaces]

## Verdict: PASS | FAIL
PASS requires zero P0 gaps.
```

## Agent Dispatch Pattern

Dispatch as `greenfield:analyzer` since it needs to read both source code and specs. Can be parallelized:
- One agent per check (7 agents), or
- Batch checks 1-3 (commands/tools, env vars, CLI flags) and checks 4-7 (subcommands, events, config keys, errors) into 2 agents

## Relationship to Other Validations

```
Source Code → [Layer 1-3 analysis] → Raw Specs → [Layer 5 sanitization] → Output Specs
                                          ↑                                        ↑
                                    source-completeness                     fidelity-validation
                                    (source → raw)                      (raw → clean)
                                          ↑                                        ↑
                                    "Did we capture it?"                   "Did we preserve it?"
```

Together these two validations form a complete chain: source-completeness ensures nothing was missed during analysis, and fidelity-validation ensures nothing was lost during sanitization.

## Integration with Public Documentation

When public documentation is available (docs mode was active), cross-reference the source findings against official docs:
- Features documented in official docs but missing from specs → P0 gap
- Features in source but not in official docs → likely internal, lower priority
- Features in official docs but not in source → version mismatch or deprecated

This cross-reference is especially valuable because official documentation represents the **intended** user-facing surface, while source extraction finds the **actual** surface. Discrepancies in either direction are informative.
