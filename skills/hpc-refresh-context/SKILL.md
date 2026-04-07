---
name: hpc-refresh-context
description: Use when Stale Signals exist in hpc-context.md and need to be reviewed, confirmed, and applied as updates to the domain knowledge
---

# HPC Refresh Context

**Core principle:** Confirmed signals get applied; unconfirmed signals get discarded. Main content sections are only modified after web-search confirmation.

## Process

```dot
digraph refresh_context {
    "Read ## Stale Signals in skills/shared/hpc-context.md" [shape=box];
    "Signals present?" [shape=diamond];
    "Done — no action needed" [shape=doublecircle];
    "For each signal: web-search official release notes / changelog" [shape=box];
    "Confirmed?" [shape=diamond];
    "Update relevant section in hpc-context.md" [shape=box];
    "Discard signal" [shape=box];
    "More signals?" [shape=diamond];
    "Clear ## Stale Signals section" [shape=box];
    "Update Last updated: date to today" [shape=box];
    "git commit" [shape=box];

    "Read ## Stale Signals in skills/hpc/shared/hpc-context.md" -> "Signals present?";
    "Signals present?" -> "Done — no action needed" [label="no"];
    "Signals present?" -> "For each signal: web-search official release notes / changelog" [label="yes"];
    "For each signal: web-search official release notes / changelog" -> "Confirmed?";
    "Confirmed?" -> "Update relevant section in hpc-context.md" [label="yes"];
    "Confirmed?" -> "Discard signal" [label="no"];
    "Update relevant section in hpc-context.md" -> "More signals?";
    "Discard signal" -> "More signals?";
    "More signals?" -> "For each signal: web-search official release notes / changelog" [label="yes"];
    "More signals?" -> "Clear ## Stale Signals section" [label="no"];
    "Clear ## Stale Signals section" -> "Update Last updated: date to today";
    "Update Last updated: date to today" -> "git commit";
}
```

## What "Confirm" Means

For each signal (e.g., "oneTBB 2022.1 removes `parallel_pipeline`"):
- Web-search: official Intel oneTBB changelog, NVIDIA CUDA release notes, or Taskflow GitHub releases
- If the source is official and the change is real → confirmed
- If the source is a blog post, StackOverflow, or unverifiable → discard

## File to Modify

`skills/shared/hpc-context.md`

Apply confirmed updates to the relevant section:
- Deprecations → add row to Deprecations table
- New API → add to appropriate Quick Reference or API section
- Changed behavior → update the relevant paragraph inline

After all signals processed:
1. Clear the `## Stale Signals` section body (keep heading, set body to: `*No signals. Add entries here manually or via web search when stack updates are found.*`)
2. Update the `Last updated:` date at the top to today's date (`YYYY-MM-DD`)

## Commit

```bash
git add skills/hpc/shared/hpc-context.md
git commit -m "chore: refresh hpc-context — applied N confirmed signals, discarded M"
```

## Red Flags

- Applying an unconfirmed signal → always web-search first
- Modifying main content sections without a corresponding signal → do not make unprompted edits
- Leaving the Stale Signals section populated after refresh → always clear it at the end
- Forgetting to update `Last updated:` date → the launch-time stale check depends on this
