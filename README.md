
# parallel-dev-workflow

Claude Code 插件 — 主 agent 主导的并发开发工作流。

## 安装

```
/plugin marketplace add morty-lou/parallel-dev-workflow-plugin
/plugin install parallel-dev-workflow@morty-lou-plugins
```

## 使用

安装后在 Claude Code 中直接说：

```
用 parallel-dev-workflow 开始长程任务：<需求描述>
```

或触发 skill：

```
/parallel-dev-workflow:parallel-dev-workflow
```

---

## 使用

安装后在 Claude Code 中直接说：

```
用 parallel-dev-workflow 开始长程任务：<需求描述>
```

或触发 skill：

```
/parallel-dev-workflow:parallel-dev-workflow
```

---

## 包含内容

```
├── .claude-plugin/
│   ├── plugin.json              ← 插件 manifest
│   └── marketplace.json         ← 市场注册文件
├── skills/
│   └── parallel-dev-workflow/
│       └── SKILL.md             ← 工作流主文档（6 个阶段）
└── agents/
    ├── code-writer.md           ← 专职写代码
    ├── logic-reviewer.md        ← 专职逻辑审查
    └── style-reviewer.md        ← 专职风格审查
```

插件安装后，三个子 agent 自动注册，主 agent 可直接通过 Task 工具调用。

---

## 工作流概览

```
Phase 1  探索 & Plan      主 agent 输出充分的 .plan.md（含内联接口签名和背景）
Phase 2  Worktree 隔离    建立独立分支，不污染主分支
Phase 3  并发写作         Task 工具同时调用多个 code-writer
Phase 4  主 agent 初审    git diff 快速检查一致性
Phase 5  并发审查         Task 同时调用 logic-reviewer + style-reviewer + codex
Phase 6  优化→单测→合并
```

## 卸载

```
/plugin uninstall parallel-dev-workflow
```
