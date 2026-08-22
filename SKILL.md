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

1. 先回复：`收到，开始创建 Skill。`
2. 确定 `BUILDER_CAPABILITY`、`SKILL_NAME`、`BUSINESS_GOAL`、`INPUTS`、`PROCESS`、`WORKFLOW_RULES`；目标目录固定由 `SKILL_NAME` 推导。
3. 如果需要定时执行，再确定 `CRON_TASKS`；每项必须包含任务名、上海时间和入口动作。
4. 当前会话按下面的执行流程直接读取依据、创建或更新文件并验证结果。
5. 全部验证通过后再向用户汇报；执行失败时保留真实错误并说明停止位置。

`BUILDER_CAPABILITY` 只允许：

- `create-capability-skill`
- `create-single-agent-orchestrator-skill`
- `create-multi-agent-orchestrator-skill`

## 执行流程

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
1. 根据目标的实际执行方式写固定流程，不强制创建独立会话或下级 Agent。
2. 写清目标、输入、依赖、执行步骤、阶段产物、放行门槛、失败停止、最终输出和关键文件路径。
3. 只有业务本身确实需要多 Agent 并行时，才在多 Agent 编排 Skill 中写清 Agent 职责、输入、输出和汇合门槛。
4. 不复制已有脚本源码，不编造参数、产物或成功结果。
5. 使用普通人能直接照做的动作句：写“打开什么、运行什么、检查什么、失败后怎么办”，不写“结构化输入、调用契约、能力收口、状态流转、阶段交接”等空泛术语。

阶段 4：创建定时入口
1. CRON_TASKS 为“无”时不创建 cron/ 目录和任务脚本。
2. 有任务时，每个任务写入 /root/.openclaw/skills/${SKILL_NAME}/cron/<task-name>.sh；文件名使用任务语义。
3. 入口脚本设置 export TZ='Asia/Shanghai'，只设置必要参数并调用一次已确定的任务入口。
4. 服务器时区是 UTC；把上海时间换算成 UTC 后写入系统 crontab，注释标明上海时间。
5. 日志重定向到 /root/.openclaw/logs/<task-name>.log，不使用 openclaw cron。
6. 中间数据写入当前 Skill 的 tmp/，禁止写入 /tmp/ 根目录。

阶段 5：验证和汇报
1. 检查 SKILL.md frontmatter、固定流程、执行步骤和关键路径。
2. 有定时任务时检查 cron/ 目录、脚本可执行权限、crontab 条目、UTC 换算和日志路径。
3. 任何验证失败都保留真实错误并停止，不得报告完成。
4. 向用户汇报创建或更新的文件、验证结果和失败项。
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
