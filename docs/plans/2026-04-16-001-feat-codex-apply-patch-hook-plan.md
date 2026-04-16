---
title: "feat: Add Codex apply_patch hook support"
type: feat
status: active
date: 2026-04-16
---

# feat: Add Codex apply_patch hook support

## Overview

Codex 的 PreToolUse/PostToolUse hook 目前仅处理 bash 工具。`apply_patch` 是 Codex 中用于文件编辑的核心工具，但被 `classify_tool` 归为 `ToolClass::Skip`，导致 hook 事件被丢弃。本计划将 `apply_patch` 作为 `FileEdit` 类型整合到现有架构中，同时添加 stdin model 优先级和文件路径提取逻辑。

## Problem Frame

用户在 `v1.2.6-codex` fork（commit `e74fc2ce`）中实现了 Codex apply_patch hook 支持。现在需要将该功能迁移到 `v1.3.0-codex`，后者引入了 `bash_tool` 基础设施（snapshot/diff 机制）。fork 的实现是在没有 `bash_tool` 框架的环境下写的，需要适配 v1.3.0 的架构。

## Requirements Trace

- R1. Codex `apply_patch` 工具在 PreToolUse 时生成 Human checkpoint，携带 `will_edit_filepaths`
- R2. Codex `apply_patch` 工具在 PostToolUse 时生成 AiAgent checkpoint，携带 `edited_filepaths`
- R3. 文件路径提取：优先从 `tool_input.files`，降级到解析 `tool_response` 输出
- R4. Model 优先级：stdin `model` 字段 > rollout JSONL 解析 > `"unknown"`
- R5. 保持现有 bash tool hook 行为不变
- R6. 单元测试覆盖 PreToolUse、PostToolUse、legacy 兼容、model 优先级等场景

## Scope Boundaries

- 不修改 bash_tool 的 snapshot/diff 机制
- 不修改其他 agent preset（Claude、Gemini 等）
- 不改变 `AgentRunResult` struct 定义
- 不处理 Codex 尚未支持的其他 file-edit 工具（仅 `apply_patch`）

## Context & Research

### Relevant Code and Patterns

- `src/commands/checkpoint_agent/bash_tool.rs:460-466` — `classify_tool(Agent::Codex, ...)` 目前仅识别 `"Bash"`，TODO 注释明确提到需要添加 file-edit 支持
- `src/commands/checkpoint_agent/agent_presets.rs:1359-1543` — `CodexPreset::run()`，PreToolUse/PostToolUse 对非 bash 工具返回 error
- `src/commands/checkpoint_agent/agent_presets.rs:162-352` — `ClaudePreset::run()`，参考模式：file-edit 工具在 PreToolUse 返回 Human checkpoint + `will_edit_filepaths`，PostToolUse 返回 `edited_filepaths`
- `src/commands/checkpoint_agent/agent_presets.rs:3024` — `collect_apply_patch_paths_from_text()` 已存在于其他 preset 中（Windsurf），可参考解析逻辑

### Fork Reference

- Commit `e74fc2ce` on `feat/codex-pretooluse-posttooluse` branch
- 添加了 `extract_filepaths_from_tool_input()` 和 `extract_filepaths_from_tool_response()` 两个 helper
- 包含 7 个单元测试

## Key Technical Decisions

- **整合而非替换**：在现有 bash_tool 框架旁添加 file-edit 路径，而非替换整套 PreToolUse/PostToolUse 逻辑。原因：v1.3.0 的 bash snapshot/diff 机制对 bash 工具仍有价值，apply_patch 不需要 snapshot
- **遵循 Claude preset 模式**：file-edit 工具的 PreToolUse/PostToolUse 处理逻辑参照 `ClaudePreset` 的模式（`agent_presets.rs:278-332`），保持一致性
- **apply_patch 分类为 FileEdit**：而非引入新的 ToolClass。与其他 agent 对 file-edit 工具的分类方式一致（如 Droid 的 `"ApplyPatch"` → `ToolClass::FileEdit`）
- **tool_response 解析作为降级**：`tool_input.files` 是更可靠的文件路径来源；`tool_response` 解析为降级方案，处理旧版 payload

## Open Questions

### Resolved During Planning

- **是否需要 bash pre-snapshot for apply_patch？** 不需要。apply_patch 是文件编辑工具，不像 bash 命令有不可预测的副作用。直接从 hook payload 提取路径即可
- **FileEdit 的 PreToolUse 是否需要 captured_checkpoint_id？** 不需要。bash snapshot 机制不适用于 file-edit 工具，设为 None

### Deferred to Implementation

- tool_response 解析的具体边界情况（非标准输出格式）可在实现时根据实际 payload 调整

## Implementation Units

- [ ] **Unit 1: Classify apply_patch as FileEdit**

**Goal:** 让 `classify_tool(Agent::Codex, "apply_patch")` 返回 `ToolClass::FileEdit`

**Requirements:** R1, R2 (前置条件)

**Dependencies:** None

**Files:**
- Modify: `src/commands/checkpoint_agent/bash_tool.rs`

**Approach:**
- 在 `classify_tool` 的 `Agent::Codex` 分支中添加 `"apply_patch" => ToolClass::FileEdit`
- 移除或更新 TODO 注释

**Patterns to follow:**
- `Agent::Droid` 的分类方式 (`bash_tool.rs:441`)：`"ApplyPatch" => ToolClass::FileEdit`

**Test scenarios:**
- Happy path: `classify_tool(Agent::Codex, "apply_patch")` returns `ToolClass::FileEdit`
- Happy path: `classify_tool(Agent::Codex, "Bash")` still returns `ToolClass::Bash` (regression)
- Edge case: `classify_tool(Agent::Codex, "unknown_tool")` still returns `ToolClass::Skip`

**Verification:**
- `cargo test` 中 bash_tool 相关测试通过
- 现有 `classify_tool` 测试扩展后覆盖 Codex apply_patch

---

- [ ] **Unit 2: Add file-edit PreToolUse/PostToolUse handling in CodexPreset**

**Goal:** CodexPreset 的 PreToolUse 和 PostToolUse 能正确处理 file-edit 工具（apply_patch），生成对应类型的 checkpoint

**Requirements:** R1, R2, R3, R5

**Dependencies:** Unit 1

**Files:**
- Modify: `src/commands/checkpoint_agent/agent_presets.rs`

**Approach:**
- 在 `CodexPreset::run()` 中添加 `is_file_edit_tool` 判断（类似 `is_bash_tool`）
- PreToolUse 分支：当 `is_file_edit_tool` 时，提取 `will_edit_filepaths`，返回 Human checkpoint（不经过 bash snapshot）。保持 bash tool 原有行为不变
- PostToolUse 分支：当 `is_file_edit_tool` 时，提取 `edited_filepaths`（优先 `tool_input.files`，降级 `tool_response` 解析），返回 AiAgent checkpoint。保持 bash tool 原有行为不变
- 未知工具仍返回 error（现有行为）

**Execution note:** 参照 ClaudePreset 的 PreToolUse/PostToolUse 模式 (`agent_presets.rs:278-332`)

**Patterns to follow:**
- `ClaudePreset::run()` 中 file-edit 和 bash 工具的分流逻辑

**Test scenarios:**
- Happy path: PreToolUse + apply_patch → Human checkpoint + will_edit_filepaths from tool_input.files
- Happy path: PostToolUse + apply_patch → AiAgent checkpoint + edited_filepaths from tool_input.files
- Integration: PostToolUse 无 tool_input.files 时，降级到 tool_response 解析提取路径
- Integration: PostToolUse + tool_response 为 JSON 对象（非字符串）时正确解析
- Happy path: PreToolUse + Bash → 原有 snapshot 行为不变（regression）
- Happy path: PostToolUse + Bash → 原有 bash_tool::handle_bash_tool 行为不变（regression）
- Edge case: PreToolUse + unknown tool → 仍返回 error

**Verification:**
- `cargo test` 通过
- 新增测试验证 PreToolUse/PostToolUse 的 file-edit 路径

---

- [ ] **Unit 3: Add stdin model priority**

**Goal:** Codex hook payload 中的 `model` 字段优先于 rollout JSONL 解析的 model

**Requirements:** R4

**Dependencies:** Unit 2

**Files:**
- Modify: `src/commands/checkpoint_agent/agent_presets.rs`

**Approach:**
- 在 `CodexPreset::run()` 中，从 `hook_data` 提取 `stdin_model`
- PreToolUse 路径（Unit 2 已处理）：直接使用 `stdin_model`
- PostToolUse/Stop/legacy 路径：将 rollout 解析的 model 改名为 `rollout_model`，最终 model = `stdin_model.or(rollout_model).unwrap_or("unknown")`
- rollout 解析失败时返回 `None` 而非 `Some("unknown")`，让 stdin_model 有机会生效

**Patterns to follow:**
- Fork commit `e74fc2ce` 中的 model 优先级逻辑

**Test scenarios:**
- Happy path: stdin model 存在时使用 stdin model
- Happy path: stdin model 不存在时使用 rollout model
- Edge case: 两者都不存在时使用 "unknown"
- Happy path: legacy payload（无 hook_event_name）仍能正确获取 model

**Verification:**
- 新增测试验证 model 优先级链

---

- [ ] **Unit 4: Add helper methods and comprehensive tests**

**Goal:** 添加文件路径提取 helper 方法，补齐完整的单元测试覆盖

**Requirements:** R3, R6

**Dependencies:** Unit 2, Unit 3

**Files:**
- Modify: `src/commands/checkpoint_agent/agent_presets.rs`

**Approach:**
- 在 `impl CodexPreset` 中添加 `extract_filepaths_from_tool_input()` 和 `extract_filepaths_from_tool_response()` 私有方法
- `extract_filepaths_from_tool_response` 解析 Codex apply_patch 输出格式：`"A path\nM path\n"`（状态字母 + 空格 + 路径）
- 在文件末尾的 `#[cfg(test)] mod tests` 中添加测试（注意：当前文件已有 `mod tests` 在 line 118-157，可能需要在 CodexPreset 附近新建独立 test module 或追加到现有 module）

**Patterns to follow:**
- Fork commit 中的 `extract_filepaths_from_tool_input` 和 `extract_filepaths_from_tool_response` 实现
- 现有 `collect_apply_patch_paths_from_text()` (line 3024) 的解析逻辑可参考

**Test scenarios:**
- Happy path: extract_filepaths_from_tool_input 从 files 数组提取路径
- Edge case: extract_filepaths_from_tool_input 无 files 字段返回 None
- Edge case: extract_filepaths_from_tool_input files 为空数组返回 None
- Happy path: extract_filepaths_from_tool_response 解析 "A path\nM path\n" 格式
- Happy path: extract_filepaths_from_tool_response 处理 JSON 对象形式的 tool_response
- Edge case: extract_filepaths_from_tool_response 输出无匹配行返回 None
- Integration: 完整 PreToolUse payload → Human checkpoint（端到端）
- Integration: 完整 PostToolUse payload → AiAgent checkpoint + edited_filepaths（端到端）
- Integration: Legacy payload（无 hook_event_name）→ AiAgent checkpoint（向后兼容）

**Verification:**
- `cargo test` 全部通过
- `cargo clippy` 无新增 warning

## System-Wide Impact

- **Interaction graph:** 仅影响 Codex agent 的 checkpoint 生成路径。不影响 daemon、其他 agent preset、或 checkpoint 存储层
- **Error propagation:** file-edit 路径不涉及 bash_tool snapshot，不会产生新的 snapshot 失败模式
- **API surface parity:** 不涉及公共 API 变更
- **Unchanged invariants:** bash tool 的 PreToolUse snapshot + PostToolUse diff 行为完全不变；Stop/legacy hook 行为完全不变；`AgentRunResult` struct 不变

## Risks & Dependencies

| Risk | Mitigation |
|------|------------|
| tool_response 格式变化导致解析失败 | tool_response 解析为降级方案，extract_filepaths_from_tool_input 是主路径；解析失败返回 None 而非 panic |
| Codex 未来添加新的 file-edit 工具 | classify_tool 中添加新工具名即可，架构已支持 |

## Sources & References

- Fork commit: `e74fc2ce` on `feat/codex-pretooluse-posttooluse` branch
- Claude preset pattern: `src/commands/checkpoint_agent/agent_presets.rs:162-352`
- bash_tool TODO: `src/commands/checkpoint_agent/bash_tool.rs:462-463`
