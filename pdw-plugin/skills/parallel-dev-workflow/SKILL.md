---
name: parallel-dev-workflow
description: 长程并发开发工作流。涉及多个模块/文件的复杂编程任务时自动激活，或用户说「长程任务」「并发开发」「保持 context 轻量」时触发。主 agent 负责探索、规划、分发与决策，代码写作和审查全部通过 Task 工具交给子 agent 并发执行。
---

# Parallel Dev Workflow

主 agent 主导的并发开发工作流，共 6 个阶段。

**核心原则：主 agent 保持轻量 context，代码写作和审查通过 `Task` 工具交由子 agent 并发执行。**

> ⚠️ `Task` 工具是 fork-join 模型：子 agent 启动后独立运行直到完成才返回结果，无法中途交互。因此 Phase 1 的 plan 必须把背景写到「子 agent 不需要额外提问就能动手」的程度。

---

## 工作流总览

```
Phase 1  探索 & Plan      ← 主 agent 独立完成，输出充分的 .plan.md
Phase 2  Worktree 隔离    ← 主 agent 建立独立分支环境
Phase 3  并发代码写作      ← Task 调用 code-writer 子 agent 并发执行
Phase 4  主 agent 初审    ← git diff 快速一致性检查
Phase 5  并发代码审查      ← Task 并发：logic-reviewer + style-reviewer + codex
Phase 6  优化 → 单测 → 合并
```

---

## Phase 1：探索 & Plan

主 agent 完整探索后输出 `.plan.md`，**不在此阶段写任何代码**。

### 探索步骤
1. 理解需求边界（功能、约束、验收标准）
2. 阅读相关代码文件，理解现有架构和接口
3. 识别需要新增/修改的模块，拆分为独立 task
4. 分析任务依赖关系，规划并发批次

### Plan 充分性标准（关键）

由于 Task 工具是 fork-join 模型，子 agent 启动后无法中途补充信息，每个 task 的描述必须满足：

- **接口明确**：依赖的类型/函数签名直接写在 plan 里，不要只写文件路径
- **行为具体**：描述「做什么」而不只是「实现 xxx 功能」，包括边界处理
- **背景内联**：把子 agent 需要了解的现有逻辑摘录进来
- **约束显式**：不允许修改的文件、必须兼容的接口，明确列出

> **自检**：写完每个 task 后问自己：「如果我是子 agent，只看这段描述，能不能不查任何其他文件就开始写代码？」答案是否则继续补充。

### Plan 格式（保存为 `.plan.md`）

```markdown
# Task Plan: <task-name>
> Branch: feature/<task-name>

## 需求摘要

## 全局约束
- 语言/框架版本:
- 命名约定:
- 测试框架:
- lint 命令:
- 不允许改动的文件:

## 任务分解

### Task-1: <模块名>
- **目标文件**: `src/xxx/yyy.ts`（新增 / 修改）
- **工作内容**: <具体描述，含函数名、参数、返回值、边界处理>
- **依赖的接口**（直接粘贴签名）:
  ```typescript
  // from src/types/user.ts
  interface IUser { id: string; name: string }
  ```
- **需要暴露的接口**:
  ```typescript
  export function doSomething(input: InputType): OutputType
  ```
- **背景信息**: <内联相关逻辑摘要>
- **注意事项**: <并发安全、错误处理方式等>

## 任务依赖图
Task-1 ──► Task-3
Task-2 ──► Task-3

## 并发批次
- 第一批: Task-1, Task-2（同时启动）
- 第二批: Task-3（等第一批全部完成后启动）
```

---

## Phase 2：Worktree 隔离

```bash
git worktree add ../$(basename $PWD)-feature -b feature/<task-name>
cd ../$(basename $PWD)-feature
```

---

## Phase 3：并发代码写作

对第一批所有 task **同时调用 Task 工具**，每次调用传入完整任务包：

```
Task("code-writer", """
【当前分支】feature/<task-name>
【目标文件】<path>（新增 / 修改）
【工作内容】<从 plan 完整复制>
【依赖的接口】<从 plan 完整复制，含类型签名>
【需要暴露的接口】<从 plan 完整复制>
【背景信息】<从 plan 完整复制>
【全局约束】<从 plan 完整复制>
【注意事项】<从 plan 完整复制>

完成后输出：
1. 已修改/新增的文件路径列表
2. 每个文件的变更简述
3. 如有偏离规格的取舍，说明原因

不要安装新依赖，不要修改范围外的文件，不要提交 commit。
""")
```

第一批全部返回后，检查结果，再同时启动第二批。

---

## Phase 4：主 agent 初审

```bash
git diff main...HEAD --stat   # 变更文件概览
git diff main...HEAD          # 详细 diff
```

检查：
- [ ] 每个 task 的目标文件都已创建/修改
- [ ] 各模块暴露的接口与 plan 规格一致
- [ ] 有依赖关系的模块之间接口对齐
- [ ] 没有遗留 TODO 或未实现的占位符

有明显问题直接修复或重跑对应 Task，不要留到审查阶段。

---

## Phase 5：并发代码审查

**同时**启动三个 Task：

```
Task("logic-reviewer", "请审查 git diff main...HEAD 的变更，重点检查逻辑正确性、接口一致性和潜在 bug，不评价风格。")

Task("style-reviewer", """
变更文件：<从 git diff --name-only 列出>
请对每个变更文件，先读取其所在目录的现有文件建立风格基线，再检查新代码的风格一致性。不评价逻辑正确性。
""")

# codex 根据你的配置调用
codex review --diff "$(git diff main...HEAD)"
```

三路全部返回后，主 agent 合并反馈：

| 优先级 | 处理方式 |
|--------|---------|
| 必须修复（逻辑错误、安全问题） | 全部处理 |
| 必须统一（破坏风格一致性） | 全部处理 |
| 建议修复 | 酌情处理 |
| 纯主观意见 | 可忽略 |

---

## Phase 6：优化 → 单测 → 合并

```bash
# 单测：覆盖正常路径、边界、错误路径
<your-test-command>

# 全部通过后合并
git checkout main
git merge feature/<task-name> --no-ff -m "feat: <task-name>"
git worktree remove ../$(basename $PWD)-feature
```
