# greenfield

> A Claude Code plugin that reverse-engineers clean behavioral specs, test vectors, and acceptance criteria from any codebase, with a full provenance trail.

**Family:** superpowers · **Type:** tool · **Lifecycle:** production · **Owner:** obra

## What it does
Greenfield reads source code, docs, SDKs, runtime behavior, and binaries, then produces behavioral specifications, test vectors, acceptance criteria, and provenance citations describing what software does rather than how a codebase does it, so a fresh team can reimplement without inheriting internal structure. /analyze runs a seven-layer pipeline dispatching analyzer/sanitizer agents under many roles; /sanitize re-runs the sanitization pass. It stops at the specs.

## How it fits
- Depends on: —
- Used by: —
- External: Claude Code (runs inside the claude CLI; dispatches Claude agents as workers); distributed via prime-radiant-marketplace.

## Runtime & data
- Runs: Installed as a Claude Code plugin; not a long-running service.
- Data in: Target codebase, docs, SDKs, runtime/binary artifacts.
- Data out: Behavioral specs, test vectors, acceptance criteria, provenance trail.

<!-- Maintained by the maintaining-project-map skill. Do not hand-edit; regenerated. -->
