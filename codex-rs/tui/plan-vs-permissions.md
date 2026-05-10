# `/plan` vs `/permissions` — 源码分析

两者的核心区别在于**操作层级**不同：`/plan` 改变模型的**协作行为模式**（软约束），`/permissions` 改变系统的**审批策略和权限配置**（硬约束）。

---

## 一、`/plan` — 切换到 Plan 模式

### 调度入口

`chatwidget/slash_dispatch.rs:223` → `apply_plan_slash_command()`

```rust
// slash_dispatch.rs:83
fn apply_plan_slash_command(&mut self) -> bool {
    if !self.collaboration_modes_enabled() {
        self.add_info_message(
            "Collaboration modes are disabled.".to_string(),
            Some("Enable collaboration modes to use /plan.".to_string()),
        );
        return false;
    }
    if let Some(mask) = collaboration_modes::plan_mask(self.model_catalog.as_ref()) {
        self.set_collaboration_mask(mask);
        true
    } else {
        self.add_info_message(
            "Plan mode unavailable right now.".to_string(),
            None,
        );
        false
    }
}
```

### 本质

将协作模式设为 `ModeKind::Plan`，并从 preset 中构建 `CollaborationModeMask` 传入 `set_collaboration_mask()`：

```rust
// collaboration_mode_presets.rs
fn plan_preset() -> CollaborationModeMask {
    CollaborationModeMask {
        name: ModeKind::Plan.display_name().to_string(),  // "Plan"
        mode: Some(ModeKind::Plan),
        model: None,                                      // 沿用当前模型
        reasoning_effort: Some(Some(ReasoningEffort::Medium)),
        developer_instructions: Some(Some(COLLABORATION_MODE_PLAN.to_string())),
    }
}
```

其中 `COLLABORATION_MODE_PLAN` 来自 `collaboration-mode-templates/templates/plan.md`，一段约 150 行的系统级 prompt，核心约束如下：

#### Plan 模式的约束（软性，prompt 层面）

| 规则 | 说明 |
|---|---|
| **三阶段工作流** | Phase 1 环境探索 → Phase 2 意图澄清 → Phase 3 方案制定 |
| **非变更操作** | 允许读文件、搜索代码、静态分析、跑构建/测试（不改 repo 文件），**禁止编辑/写文件** |
| **`request_user_input` 可用** | Plan 模式下该工具可用（`ModeKind::Plan.allows_request_user_input() -> true`），模型可向用户提问 |
| **推理力度** | 默认 `Medium`，可通过 `config.plan_mode_reasoning_effort` 覆盖 |
| **退出条件** | 用户显式切换到其他协作模式 |
| **最终输出** | `<proposed_plan>` 块，要求 "decision complete"（实施者无需再做任何决策） |

### 与 Default 模式的对比

```rust
// ModeKind
pub const fn allows_request_user_input(self) -> bool {
    matches!(self, Self::Plan)  // 只有 Plan 模式允许
}

pub const fn is_tui_visible(self) -> bool {
    matches!(self, Self::Plan | Self::Default)  // TUI 中只展示这两种
}
```

`Default` 模式的 `developer_instructions`（`templates/default.md`）很短，核心是：**偏好自行假设并直接执行，不到必要时不提问**。

### Plan Mode Nudge

TUI 中还有一个 plan mode nudge 机制：当用户在 Default 模式下输入含有 "plan" 关键词的消息时，会在输入框下方提示用户切换到 Plan 模式（支持 Tab 切换或 Esc 关闭）。`/plan` 命令和 `!plan` 前缀都不会触发 nudge。

### 支持 inline args

`/plan` 支持 inline args（如 `/plan build a todo app`），会先切换到 Plan 模式，然后以 Plan 模式提交这条消息。

---

## 二、`/permissions` — 配置权限审批策略

### 调度入口

`chatwidget/slash_dispatch.rs:256` → `open_permissions_popup()`（`chatwidget.rs:8237`）

### 本质

打开一个选择弹窗，展示**三个内置审批预设**（`approval-presets/src/lib.rs`）：

```rust
pub fn builtin_approval_presets() -> Vec<ApprovalPreset> {
    vec![
        ApprovalPreset {
            id: "read-only",
            label: "Read Only",
            approval: AskForApproval::OnRequest,
            permission_profile: PermissionProfile::read_only(),
        },
        ApprovalPreset {
            id: "auto",
            label: "Default",
            approval: AskForApproval::OnRequest,
            permission_profile: PermissionProfile::workspace_write(),
        },
        ApprovalPreset {
            id: "full-access",
            label: "Full Access",
            approval: AskForApproval::Never,
            permission_profile: PermissionProfile::Disabled,
        },
    ]
}
```

每个预设同时控制两个正交维度：

| 维度 | 类型 | 说明 |
|---|---|---|
| `AskForApproval` | `OnRequest` / `Never` | 控制是否需要用户审批才能执行操作 |
| `PermissionProfile` | `read_only` / `workspace_write` / `Disabled` | 控制文件系统和网络的权限边界 |

### 三个预设对比

| 预设 | 审批策略 | 文件系统 | 网络 | 说明 |
|---|---|---|---|---|
| **Read Only** | `OnRequest`（均需审批） | 只读工作区 | 需审批 | 最安全 |
| **Default/Auto** | `OnRequest`（网络等需审批） | 可读写工作区 | 需审批 | 默认配置 |
| **Full Access** | `Never`（从不审批） | 无限制 | 无限制 | 最宽松 |

当 `GuardianApproval` feature 开启时，还会提供 **Auto-review** 变体：将部分 `OnRequest` 审批路由给 auto-reviewer 子代理自动审核，而非要求用户手动批准。

### 弹窗还处理：

- **Windows degraded sandbox 提示**：在受限令牌模式下提示用户 `/setup-default-sandbox`
- **Full Access 二次确认**：选择 Full Access 时需要额外确认
- **world-writable 目录警告**：当工作区存在全局可写目录时发出警告

---

## 三、关键对比总结

| 维度 | `/plan` | `/permissions` |
|---|---|---|
| **操作的类型** | `CollaborationModeMask` | `AskForApproval` + `PermissionProfile` |
| **约束层级** | LLM prompt 指令（**软约束**） | 系统沙箱/审批拦截（**硬约束**） |
| **文件写入控制** | 通过 prompt 告知模型"不要写文件" | 通过 Approval Policy 拦截/审批 write 操作 |
| **网络访问控制** | 模型在 Plan 模式下倾向于避免 | 通过 PermissionProfile 和审批策略控制 |
| **是否需要用户审批** | 不涉及 | 核心就是控制此行为 |
| **与模式的关系** | 本身就是一个协作模式 | 是底层策略，适用于所有模式 |
| **inline args** | 支持（`/plan <msg>` 直接以 Plan 模式提交） | 不支持，只打开弹窗 |
| **运行时状态** | `active_collaboration_mask` | `config.permissions.approval_policy` + `config.approvals_reviewer` |

### 两者可以组合使用

- **Plan 模式 + Read Only 权限**：模型只能读 + 探索 + 做方案，且系统层面也拦截写入操作。双保险。
- **Plan 模式 + Default 权限 + Auto-review**：模型在 Plan 模式探索和提问，如有操作需要审批则由 auto-reviewer 子代理处理。
- **Default 模式 + Full Access**：模型直接执行，永不询问审批。

### 核心直觉

> `/plan` 决定「**模型怎么想**」— 是深入调研后出方案，还是直接上手写代码。
>
> `/permissions` 决定「**系统允许什么**」— 能碰哪些文件、能不能联网、要不要你点头。
