# 单 Agent 编排 Skill 模板

- 当前 Agent 按依赖顺序调用能力 Skills。
- 写清阶段输入、输出、交接产物、放行门槛和失败停止。
- 不 dispatch 下级 Agent，不复制能力实现。
- 仅在调用方提供实际定时任务时创建 `cron/<task-name>.sh`；没有定时任务时不创建 `cron/`。
