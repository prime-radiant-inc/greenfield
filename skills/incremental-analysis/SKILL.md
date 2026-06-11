---
name: incremental-analysis
description: Detect existing workspace, diff current sources against high water mark metadata, classify specs as unchanged/stale/orphaned/new, re-analyze only what changed. Activates automatically when /analyze finds an existing workspace.
---

# Incremental Analysis

When a workspace already exists for a target, avoid re-analyzing unchanged sources. Diff the current state against what was previously analyzed and re-analyze only what's stale.

## When to Use

This skill activates automatically when `/analyze` is invoked and a workspace directory already exists at the target path. No `--follow` flag needed. If the workspace has a `.git` directory, this is an incremental run. Otherwise, it's a fresh analysis.

Note: `--follow` is for continuing an interrupted analysis within a single run. Incremental analysis is for subsequent runs against an evolved codebase.

## High Water Mark Metadata

Each raw specs tracks the sources it was derived from in YAML frontmatter. This metadata is the basis for incremental diffing.

```yaml
---
spec_id: SPEC-SESSION-001
derived_from:
  - path: workspace/raw/source/analysis/session-module.md
    source_ref: a1b2c3d  # git SHA of original source file, if available
    file_hash: sha256:deadbeef1234  # sha256 of source artifact at analysis time
  - path: workspace/public/docs/session-api.md
    fetched_at: 2026-04-13T10:00:00Z
    url: https://docs.example.com/sessions
  - path: workspace/raw/test-evidence/e2e-session-flow.md
    file_hash: sha256:cafebabe5678
analyzed_at: 2026-04-13T14:30:00Z
greenfield_version: "2.0"
---
```

### Derived From Entry Types

Each entry in `derived_from` uses the fields appropriate to its source type:

| Source Type | Required Fields | Optional Fields |
|-------------|----------------|-----------------|
| Source code artifact | `path`, `file_hash` | `source_ref` (git SHA) |
| Web-fetched doc | `path`, `url`, `fetched_at` | `file_hash` |
| Runtime observation | `path`, `file_hash` | timestamp in `fetched_at` |
| Git history | `path`, `source_ref` | `file_hash` |
| Test evidence | `path`, `file_hash` | -- |

### Writing High Water Marks

When writing or updating a raw spec, always include `derived_from` metadata:

- For source code artifacts: include the git SHA of the original source file (if available) and the sha256 hash of the analysis artifact
- For web-fetched docs: include the fetch timestamp and URL
- For runtime observations: include the observation timestamp
- For test evidence: include the file hash of the evidence artifact

## Incremental Detection

When `/analyze` is invoked, before workspace initialization:

```bash
WORKSPACE="${WORKSPACE_ARG:-./analysis-workspace}"

if [ -d "$WORKSPACE/.git" ]; then
  echo "Existing workspace detected at $WORKSPACE"
  echo "Running incremental analysis..."
  # Proceed with incremental flow (this skill)
else
  echo "No existing workspace. Running fresh analysis..."
  # Proceed with fresh workspace initialization (standard /analyze flow)
fi
```

To force a fresh analysis, delete the workspace directory.

## Incremental Flow

### Step 1: Fresh Discovery

Run the discovery/inventory phase to produce a new inventory manifest. This is always fresh -- the available sources may have changed since the last run. New files may exist, old files may have been deleted, documentation may have been updated.

The output is a complete inventory of what sources are available NOW, independent of what was analyzed before.

### Step 2: Diff Against Existing Specs

For each existing raw specs, check its `derived_from` metadata against current state:

```bash
# For each source with a git ref:
current_sha=$(git -C "$SOURCE_REPO" log -1 --format="%H" -- "$SOURCE_FILE")
# Compare against source_ref in derived_from

# For each source with a file hash:
current_hash=$(sha256sum "$ARTIFACT_PATH" | cut -d' ' -f1)
# Compare against file_hash in derived_from

# For web sources:
# Check if content has changed (HTTP ETag/Last-Modified, or re-fetch and diff)
```

### Step 3: Classify Specs

Each existing spec gets exactly one classification:

| Classification | Criteria | Action |
|---------------|----------|--------|
| **Unchanged** | All `derived_from` sources match current state (same hash/SHA) | Carry forward as-is |
| **Stale** | One or more `derived_from` sources have changed | Queue for re-analysis |
| **Orphaned** | A `derived_from` source no longer exists | Queue for re-analysis (may result in spec removal) |
| **New** | Discovery found sources with no corresponding spec | Queue for fresh analysis |

Write the classification results to the workspace before proceeding:

```
workspace/raw/incremental/
    classification.md        # Summary: N unchanged, N stale, N orphaned, N new
    unchanged.txt            # List of spec IDs carried forward
    stale.txt                # List of spec IDs queued for re-analysis, with changed sources
    orphaned.txt             # List of spec IDs whose sources no longer exist
    new.txt                  # List of new sources needing analysis
```

### Step 4: Re-Analyze

For **stale** specs: dispatch the appropriate Layer 1-3 agents. The analysis agents write updated specs that replace the stale versions. The `derived_from` metadata is updated with current hashes/SHAs.

For **new** sources: dispatch fresh Layer 1-3 analysis. Produces new spec files with `derived_from` metadata.

For **orphaned** specs: if the source was removed (feature deleted), remove the spec. If the source was renamed or moved, the discovery agent should detect the new location and classify it as "new" instead. When uncertain, flag for manual review rather than silently deleting.

For **unchanged** specs: no action. Carry forward as-is.

### Step 5: Re-Validate (Gates 1-2)

Run Gates 1-2 on the **full set** (carried forward + updated + new). Cross-module interactions may have changed even if a spec's direct sources didn't. A change in one module can create contradictions with an unchanged spec in another module.

Gate failures during incremental runs follow the same remediation loop as fresh runs (up to 3 attempts).

### Step 6: Re-Sanitize and Audit (Layers 5-7)

Re-run Layers 5-7 on **everything** -- not just the changed specs. This is required because:

- **Layer 5 (Sanitization)** is a rewrite pass. The sanitizer's understanding of the whole system may have changed with new/updated specs. A carried-forward spec that was properly sanitized last run may need different treatment now that adjacent specs have changed.
- **Layer 6 (Second-Pass Review)** must see the complete output. Partial audits miss cross-file leakage.
- **Layer 7 (Fidelity Check)** must validate the complete raw-to-clean mapping.

Sanitization is cheap relative to Layer 1-3 analysis. The sanitization discipline requires a fresh rewrite pass regardless.

## Workspace Metadata Update

After incremental analysis completes, append to the `run_history` array in `workspace.json`:

```json
{
  "run_history": [
    {
      "run_id": "001",
      "started_at": "2026-04-13T10:00:00Z",
      "completed_at": "2026-04-13T14:30:00Z",
      "type": "fresh",
      "specs_analyzed": 24,
      "specs_carried_forward": 0,
      "source_high_water": "a1b2c3d"
    },
    {
      "run_id": "002",
      "started_at": "2026-04-20T09:00:00Z",
      "completed_at": "2026-04-20T10:15:00Z",
      "type": "incremental",
      "specs_analyzed": 3,
      "specs_carried_forward": 21,
      "specs_orphaned": 0,
      "source_high_water": "d4e5f6a"
    }
  ]
}
```

### Run History Fields

| Field | Type | Description |
|-------|------|-------------|
| `run_id` | string | Sequential identifier (zero-padded) |
| `started_at` | ISO 8601 | When the run began |
| `completed_at` | ISO 8601 | When the run finished |
| `type` | enum | `fresh` or `incremental` |
| `specs_analyzed` | integer | Specs that went through Layer 1-3 analysis |
| `specs_carried_forward` | integer | Unchanged specs carried forward (0 for fresh runs) |
| `specs_orphaned` | integer | Specs removed because sources no longer exist |
| `source_high_water` | string | Git SHA of the source repo HEAD at analysis time |

## Relationship to Other Skills

- **analysis-pipeline**: Master methodology; incremental analysis modifies which specs enter the pipeline, not the pipeline itself
- **source-analysis**: Layer 1 source analysis; dispatched for stale and new specs during incremental runs
- **validation-methodology**: Gates 1-2 run on the full set regardless of incremental vs fresh
- **spec-sanitization**: Layers 5-7 always run on everything; incremental savings come from Layers 1-3 only
