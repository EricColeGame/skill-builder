---
name: skill-builder
description: 根据业务目标、输入、过程、工作流规则和可选定时任务，创建或更新规范化的 OpenClaw 能力 Skill、单 Agent 编排 Skill 或多 Agent 编排 Skill。
read_when:
  - 需要把已经跑通的脚本封装成能力 Skill
  - 需要创建单 Agent 或多 Agent 编排 Skill
  - 需要给 Skill 增加系统 crontab 定时入口
metadata: {"openclaw":{"emoji":"🧱"}}
allowed-tools: Bash(skill-builder:*)
---

# Skill Builder

## 固定流程（必须）

1. 先回复：`收到，开始创建 Skill，已委派给 Claude Code 执行。`
2. 确定 `BUILDER_CAPABILITY`、`SKILL_NAME`、`BUSINESS_GOAL`、`INPUTS`、`PROCESS`、`WORKFLOW_RULES`；目标目录固定由 `SKILL_NAME` 推导。
3. 如果需要定时执行，再确定 `CRON_TASKS`；每项必须包含任务名、上海时间、入口动作和独立 tmux 会话名。
4. 调用 `dispatch-claude-code.sh`，把下面的完整指令交给 Claude Code。
5. dispatch 自动发送 tmux 连接信息；立即返回，不等待任务完成。

`BUILDER_CAPABILITY` 只允许：

- `create-capability-skill`
- `create-single-agent-orchestrator-skill`
- `create-multi-agent-orchestrator-skill`

## 执行命令

```bash
#!/bin/bash

BUILDER_CAPABILITY="${BUILDER_CAPABILITY:?需要指定 skill-builder 能力}"
SKILL_NAME="${SKILL_NAME:?需要指定 Skill 名称}"
[[ "$SKILL_NAME" =~ ^[a-z0-9]+(-[a-z0-9]+)*$ ]] || {
  echo "SKILL_NAME 只能使用小写字母、数字和连字符" >&2
  exit 1
}
TARGET_DIR="/root/.openclaw/skills/${SKILL_NAME}"
BUSINESS_GOAL="${BUSINESS_GOAL:?需要提供业务目标}"
INPUTS="${INPUTS:?需要提供输入契约}"
PROCESS="${PROCESS:?需要提供具体过程}"
WORKFLOW_RULES="${WORKFLOW_RULES:?需要提供工作流规则}"
CRON_TASKS="${CRON_TASKS:-无}"
FEISHU_TARGET="${FEISHU_TARGET:?需要提供飞书通知目标}"

TMUX_SESSION="skill-builder-$(echo "$SKILL_NAME" | tr '_' '-')"
PROMPT="<使用下方 Claude Code 执行指令，并注入以上参数>"

env -u CLAUDECODE /root/.openclaw/skills/coding-agent/scripts/dispatch-claude-code.sh \
  -p "$PROMPT" \
  -n "skill-builder-${SKILL_NAME}" \
  -g "$FEISHU_TARGET" \
  --permission-mode 'acceptEdits' \
  --workdir '/root' \
  --tmux-session "$TMUX_SESSION"
```

## Claude Code 执行指令

```text
创建或更新一个 OpenClaw Skill。

参数：
- 能力：${BUILDER_CAPABILITY}
- Skill 名称：${SKILL_NAME}
- 目标目录：/root/.openclaw/skills/${SKILL_NAME}（固定由 Skill 名称推导）
- 业务目标：${BUSINESS_GOAL}
- 输入：${INPUTS}
- 具体过程：${PROCESS}
- 工作流规则：${WORKFLOW_RULES}
- 定时任务：${CRON_TASKS}
- 飞书目标：${FEISHU_TARGET}

阶段 1：读取依据
1. 完整读取 PROCESS 中列出的现有脚本和依赖 Skill，核对真实参数、输入、输出、退出码和失败分支。
2. 根据 BUILDER_CAPABILITY 读取对应模板：
   - create-capability-skill：/root/.openclaw/skills/skill-builder/templates/capability.md
   - create-single-agent-orchestrator-skill：/root/.openclaw/skills/skill-builder/templates/orchestrator-single.md
   - create-multi-agent-orchestrator-skill：/root/.openclaw/skills/skill-builder/templates/orchestrator-multi.md

阶段 2：创建目录
1. 校验 SKILL_NAME 只包含小写字母、数字和连字符，再创建 /root/.openclaw/skills/${SKILL_NAME}。
2. 只创建实际需要的 scripts/、prompts/、references/、configs/、data/、tmp/。
3. 只有 CRON_TASKS 包含实际定时任务时才创建 /root/.openclaw/skills/${SKILL_NAME}/cron/；没有定时任务时不创建该目录。

阶段 3：写 SKILL.md
1. 最外层声明 dispatch 流程；OpenClaw 只负责任务发放，Claude Code 负责执行和错误修复。
2. 固定流程为：先回复确认 → 确定参数 → 调用 dispatch → 发送飞书通知 → 立即返回。
3. 写出完整 dispatch-claude-code.sh 命令和传给 -p 的完整分阶段 prompt。
4. 写清目标、输入、依赖、阶段产物、放行门槛、失败停止、最终输出和关键文件路径。
5. 不复制已有脚本源码，不编造参数、产物或成功结果。
6. 使用普通人能直接照做的动作句：写“打开什么、运行什么、检查什么、失败后怎么办”，不写“结构化输入、调用契约、能力收口、状态流转、阶段交接”等空泛术语。

阶段 4：创建定时入口
1. CRON_TASKS 为“无”时不创建 cron/ 目录和任务脚本。
2. 有任务时，每个任务写入 /root/.openclaw/skills/${SKILL_NAME}/cron/<task-name>.sh；文件名使用任务语义，不使用 dispatch.sh。
3. 入口脚本设置 export TZ='Asia/Shanghai'，内部调用 dispatch-claude-code.sh，并为每个任务使用独立 tmux 会话名。
4. 服务器时区是 UTC；把上海时间换算成 UTC 后写入系统 crontab，注释标明上海时间。
5. 日志重定向到 /root/.openclaw/logs/<task-name>.log，不使用 openclaw cron。
6. 中间数据写入当前 Skill 的 tmp/，禁止写入 /tmp/ 根目录。

阶段 5：验证和通知
1. 检查 SKILL.md frontmatter、固定流程、执行命令、完整 prompt 和关键路径。
2. 检查 cron/ 存在；有定时任务时检查脚本可执行、crontab 条目、UTC 换算、日志路径和 tmux 会话名。
3. 任何验证失败都保留真实错误并停止，不得报告完成。
4. 向 FEISHU_TARGET 发送成功或失败摘要。
```

## 关键文件路径

```text
/root/.openclaw/skills/skill-builder/
├── SKILL.md
└── templates/
│   ├── capability.md
│   ├── orchestrator-single.md
│   └── orchestrator-multi.md
```
