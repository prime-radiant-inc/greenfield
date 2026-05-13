---
name: provenance-methodology
description: Cross-cutting provenance discipline. Citation format, confidence levels, source hierarchy, session capture. Loaded by every analysis agent.
---

# Provenance Methodology

Every behavioral claim you write MUST have a citation. No exceptions.

## The Citation Rule

A "behavioral claim" is any assertion about what the target system does, how it responds, what data it accepts or produces, what errors it raises, what limits it enforces, or how it transitions between states.

**Cite as you go.** Do NOT batch citations at the end of your analysis. Every time you write a behavioral claim, the very next thing you write is the citation.

## Citation Format

Inline HTML comment, immediately after the claim:

```markdown
- Sessions expire after 30 minutes of inactivity
  <!-- cite: source=official-docs, ref=https://docs.example.com/sessions#timeout, confidence=confirmed, agent=doc-researcher, corroborated_by=runtime-observation -->
```

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `source` | enum | The type of source (see Source Types below) |
| `ref` | string | Specific location: URL, `workspace/` file path with optional `:line`, or session timestamp |
| `confidence` | enum | `confirmed`, `inferred`, or `assumed` (see Confidence Levels below) |
| `agent` | string | Your agent name |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `corroborated_by` | comma-separated list | Other source types that independently confirm this claim |
| `session` | string | Agent ID for session log in `workspace/provenance/sessions/` |

### Placement Rules

- The `<!-- cite: -->` comment MUST appear on the line immediately following the claim it supports, or on the same line after the claim text.
- If a single claim is supported by multiple independent sources, use a single citation with `corroborated_by` listing the additional sources.
- If a paragraph contains multiple claims, each claim gets its own citation. Break compound sentences into separate cited items.
- Block-level claims (tables, code blocks, decision trees) place the citation comment immediately after the closing block.

## Source Types (Strongest to Weakest)

| Rank | Source Type | When to Use | Origin |
|------|-----------|-------------|-------|
| 1 | `official-docs` | Published documentation, README, man pages, API references, changelogs | PUBLIC |
| 2 | `public-api` | Observed behavior of public API endpoints, CLI commands | PUBLIC |
| 3 | `sdk-analysis` | Published SDK, client library, or plugin source code | PUBLIC |
| 4 | `community-knowledge` | Stack Overflow, blog posts, conference talks, third-party tutorials | PUBLIC |
| 5 | `runtime-observation` | Behavior observed by running the product in a container | RAW |
| 6 | `source-code` | Proprietary source code, bundles, minified JS | RAW |
| 7 | `binary-analysis` | Disassembly, decompilation, binary instrumentation | RAW |
| 8 | `inferred` | Reasoning, convention, analogy. No direct observation. | N/A |

### Agent Source Type Guide

| Agent | Primary Source Type |
|-------|-------------------|
| `doc-researcher` | `official-docs` or `community-knowledge` |
| `sdk-analyzer`, `integration-test-miner` | `sdk-analysis` |
| `bundle-splitter`, `chunk-analyzer`, `function-analyzer`, `targeted-extractor` | `source-code` |
| `cli-explorer`, `web-ui-explorer`, `behavior-observer`, `ux-documenter` | `runtime-observation` |
| `binary-surveyor`, `binary-deep-analyzer` | `binary-analysis` |
| Layer 2 synthesis agents | Cite upstream agent output files |
| Layer 3 documentation agents | Cite all supporting evidence from any source |

## Confidence Levels

### `confirmed`

Two or more independent sources agree, OR a single runtime observation with reproducible steps.

**Use when:**
- Official docs state X AND source code confirms X
- Runtime observation shows X AND SDK client handles X
- Two independent community sources describe X consistently
- A single reproducible runtime observation (documented input, steps, output)

### `inferred`

One authoritative source, no contradictions.

**Use when:**
- Official docs state X but no second source confirms it
- Source code clearly implements X but no docs mention it
- A single well-regarded community source describes X, consistent with known behaviors

### `assumed`

Convention, pattern matching, or reasoning. No direct source.

**Use when:**
- Following a common convention (e.g., "REST API probably returns JSON") but no source confirms
- A pattern in one module is assumed to apply in another
- Evidence is partial and the claim is the most plausible interpretation
- Filling a spec gap where behavior must exist for the system to function

## How to Determine Confidence

```dot
digraph confidence_determination {
    rankdir=TB;

    "Determine confidence level" [shape=doublecircle];
    "How many independent sources?" [shape=diamond];
    "Is the source authoritative?" [shape=diamond];
    "Any contradicting evidence?" [shape=diamond];
    "Use confirmed" [shape=box];
    "Use inferred" [shape=box];
    "Use assumed" [shape=box];
    "STOP: Record the contradiction" [shape=octagon, style=filled, fillcolor=red, fontcolor=white];

    "Determine confidence level" -> "How many independent sources?";
    "How many independent sources?" -> "Any contradicting evidence?" [label="2 or more"];
    "How many independent sources?" -> "Is the source authoritative?" [label="1"];
    "How many independent sources?" -> "Use assumed" [label="0"];
    "Is the source authoritative?" -> "Use inferred" [label="yes"];
    "Is the source authoritative?" -> "Use assumed" [label="no"];
    "Any contradicting evidence?" -> "STOP: Record the contradiction" [label="yes"];
    "Any contradicting evidence?" -> "Use confirmed" [label="no"];
}
```

**Authoritative sources:** official docs, source code, runtime observation, specific community content. **Not authoritative:** your own reasoning (always `assumed` unless corroborated).

## Confidence Escalation

Confidence can be upgraded but never downgraded without new contradicting evidence.

| From | To | Trigger |
|------|----|---------|
| `assumed` | `inferred` | A single authoritative source confirms the claim |
| `assumed` | `confirmed` | Two independent sources found, OR reproducible runtime observation |
| `inferred` | `confirmed` | A second independent source found, OR reproducible runtime observation |

When escalating, update the `<!-- cite: -->` comment with the new confidence level and add the confirming source to `corroborated_by`.

## Handling Contradictions

```dot
digraph contradiction_handling {
    rankdir=TB;

    "New source contradicts existing claim" [shape=ellipse];
    "Record both claims with citations" [shape=box];
    "Does runtime observation contradict docs?" [shape=diamond];
    "Does source code contradict docs?" [shape=diamond];
    "Higher-ranked source wins" [shape=box];
    "Runtime observation wins (docs may be stale)" [shape=box];
    "Source code wins for implementation details" [shape=box];
    "Preserve all conflicts in citations for audit" [shape=box];

    "New source contradicts existing claim" -> "Record both claims with citations";
    "Record both claims with citations" -> "Does runtime observation contradict docs?";
    "Does runtime observation contradict docs?" -> "Runtime observation wins (docs may be stale)" [label="yes"];
    "Does runtime observation contradict docs?" -> "Does source code contradict docs?" [label="no"];
    "Does source code contradict docs?" -> "Source code wins for implementation details" [label="yes"];
    "Does source code contradict docs?" -> "Higher-ranked source wins" [label="no"];
    "Higher-ranked source wins" -> "Preserve all conflicts in citations for audit";
    "Runtime observation wins (docs may be stale)" -> "Preserve all conflicts in citations for audit";
    "Source code wins for implementation details" -> "Preserve all conflicts in citations for audit";
}
```

## Common Mistakes

**Citing everything as "confirmed" because you feel confident.**
`confirmed` requires 2+ independent sources. Your confidence in your own analysis is not a second source. One source = `inferred`. Period.

**Using `source=inferred` for everything you cannot directly cite.**
`source=inferred` means the claim was derived by reasoning. If you read it in docs, `source=official-docs`. If you saw it in code, `source=source-code`. Source type = where the evidence came from. Confidence = how sure you are.

**Omitting the `ref` field or using vague references.**
"ref=docs" is useless. "ref=source code" is useless. Use specific references:
- `ref=https://docs.example.com/auth#tokens` (specific URL)
- `ref=workspace/raw/source/analysis/chunk-023.md:45` (file:line)
- `ref=provenance/sessions/cli-explorer-abc123.jsonl:2026-03-01T10:30:00Z` (session:timestamp)

**Not recording assumed claims.**
Unrecorded assumptions look like confirmed facts to downstream agents and the implementer. Always cite assumed claims with `confidence=assumed`. List them in an Assumptions section. This is honesty, not weakness.

**Batching citations at the end.**
You forget sources. You guess at confidence. Citations become decoration instead of evidence. Cite as you go.

## Session Capture

After each subagent completes, the orchestrator:

1. Copies the session JSONL to `workspace/provenance/sessions/{agent-name}-{agent-id}.jsonl`
2. Records the agent run in `workspace.json`
3. Appends to the audit log at `workspace/provenance/audit-log.md`
4. Creates a git commit containing all output files, session log, and updated metadata

The session log contains every tool call, search result, and file reading that the agent performed. It is the deepest layer of provenance evidence.
