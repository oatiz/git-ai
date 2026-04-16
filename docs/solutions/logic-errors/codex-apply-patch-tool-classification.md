---
title: Codex apply_patch classified as Skip causing silent attribution loss
date: "2026-04-16"
category: logic-errors
module: checkpoint_agent
problem_type: logic_error
component: tooling
severity: high
symptoms:
  - "Codex apply_patch PreToolUse/PostToolUse hook events silently discarded"
  - "No file-edit attribution for Codex apply_patch — fell back to coarser Stop hook"
  - "Model name from Codex hook payload ignored, always parsed from rollout JSONL"
root_cause: logic_error
resolution_type: code_fix
related_components:
  - development_workflow
tags:
  - codex
  - apply-patch
  - tool-classification
  - file-edit
  - checkpoint-agent
  - attribution
---

# Codex apply_patch classified as Skip causing silent attribution loss

## Problem

Codex's `apply_patch` tool (its primary file-editing tool) was classified as `ToolClass::Skip` in `classify_tool()`, causing all PreToolUse/PostToolUse hook events to be silently discarded. File-edit attribution for Codex degraded to the much coarser "Stop" hook granularity, losing per-tool-call file tracking.

## Symptoms

- Codex `apply_patch` events return `Err(PresetError("Skipping Codex PreToolUse for unsupported tool apply_patch"))` — but this error is swallowed, producing no visible warning
- `will_edit_filepaths` never populated for Codex file edits (no Human checkpoint before edits)
- `edited_filepaths` never populated for Codex file edits (no AiAgent checkpoint after edits)
- Model name from `hook_data["model"]` ignored; always fell back to rollout JSONL parsing or `"unknown"`

## What Didn't Work

- The codebase had a TODO comment acknowledging the gap: `// TODO: classify Codex file-edit tools here once Codex ships file-edit tool hooks.` — but the TODO was never resolved
- The existing bash_tool snapshot/diff pipeline is not appropriate for apply_patch: it adds latency for deterministic file-edit tools whose payloads already contain the file paths

## Solution

Four changes across two files:

**1. Tool classification** (`src/commands/checkpoint_agent/bash_tool.rs`):

```rust
// Before
Agent::Codex => match tool_name {
    // TODO: classify Codex file-edit tools here ...
    "Bash" => ToolClass::Bash,
    _ => ToolClass::Skip,  // apply_patch fell here
},

// After
Agent::Codex => match tool_name {
    "apply_patch" => ToolClass::FileEdit,
    "Bash" => ToolClass::Bash,
    _ => ToolClass::Skip,
},
```

**2. PreToolUse file-edit fast path** (`src/commands/checkpoint_agent/agent_presets.rs`):

Early return before transcript parsing — file-edit tools don't need rollout parsing for the pre-edit checkpoint:

```rust
if hook_event_name == Some("PreToolUse") && is_file_edit_tool {
    let will_edit_filepaths =
        CodexPreset::extract_filepaths_from_tool_input(&hook_data);

    return Ok(AgentRunResult {
        checkpoint_kind: CheckpointKind::Human,
        will_edit_filepaths,
        transcript: None,           // no transcript needed
        captured_checkpoint_id: None, // no bash snapshot needed
        ..
    });
}
```

**3. PostToolUse file-edit branch**: Extracts `edited_filepaths` with two-tier fallback (`tool_input.files` preferred, `tool_response` output parsing as fallback).

**4. Model priority**: `stdin_model.or(rollout_model).unwrap_or("unknown")`. Rollout parse failure now returns `None` instead of `Some("unknown")` so stdin model takes effect.

## Why This Works

- **Root cause**: `classify_tool(Agent::Codex, "apply_patch")` returned `Skip` — a simple mapping omission. Adding the `FileEdit` classification routes hook events into the correct handler
- **File-edit tools bypass bash snapshot/diff**: Bash commands have unpredictable side effects (need snapshot diffing). `apply_patch` is deterministic — the hook payload already says which files will be modified. This matches the `ClaudePreset` pattern where file-edit tools extract paths from the payload while bash tools go through the snapshot pipeline
- **Model priority chain**: If rollout parsing returns `Some("unknown")` on failure, it blocks a valid stdin model. Using `None` for parse failures lets the priority chain work correctly

## Prevention

**When adding a new tool to any agent preset:**

1. **Always classify in `classify_tool()`** — unclassified tools silently fall to `ToolClass::Skip`. There is no runtime warning. Check `bash_tool.rs` for the agent's match arm
2. **Choose the right ToolClass** — `FileEdit` for deterministic file-editing tools (paths in payload), `Bash` for shell commands (need snapshot diffing)
3. **Add test for classification** — at minimum: `assert_eq!(classify_tool(Agent::X, "tool_name"), ToolClass::FileEdit)`
4. **Handle file-edit in PreToolUse/PostToolUse** — file-edit tools should extract paths from the hook payload, not go through bash snapshot/diff. Follow the ClaudePreset pattern (`agent_presets.rs:278-332`)
5. **Test both PreToolUse and PostToolUse paths** — PreToolUse should produce `Human` checkpoint, PostToolUse should produce `AiAgent` checkpoint

**File path extraction helpers** (`extract_filepaths_from_tool_input`, `extract_filepaths_from_tool_response`) are reusable for other agents that send similar payload shapes.

## Related Issues

- Plan: `docs/plans/2026-04-16-001-feat-codex-apply-patch-hook-plan.md`
- Fork commit: `e74fc2ce` on `feat/codex-pretooluse-posttooluse` branch (original implementation by oatiz)
- Existing pattern reference: `ClaudePreset` file-edit handling (`agent_presets.rs:278-332`)
- Similar classification: `Agent::Droid` has `"ApplyPatch" => ToolClass::FileEdit` (`bash_tool.rs:441`)
