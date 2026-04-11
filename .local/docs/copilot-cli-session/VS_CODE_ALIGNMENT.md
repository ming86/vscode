# VS Code Alignment Mapping

> **Status:** Placeholder — to be populated during Phase 1 implementation.
> Last updated: 2026-04-21

## Purpose

This document maps webapp components to their VS Code source counterparts,
enabling upstream change tracking and hybrid alignment per constraint **ARC-10**.

See [08-constraints-and-requirements.md](./08-constraints-and-requirements.md) for the full ARC-10 specification.

## Mapping Table

| Webapp File | VS Code Source | Alignment | Notes |
|---|---|---|---|
| *To be populated during implementation* | | | |

## Drift Detection

CI checks will compare this mapping against the webapp source tree.
Any new VS Code file not reflected here will trigger a warning.

## How to Update

1. When creating a new webapp component, add a row mapping it to the VS Code source
2. When VS Code updates a mapped file, review and update the webapp counterpart
3. Run `npm run alignment:check` to verify all mappings are current
