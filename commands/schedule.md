启动 modularize skill 的 Phase 3：依赖图计算 + 波次调度。

**立即执行**：

1. 读取 `~/.claude/skills/modularize/phases/03-schedule.md` 中的所有指令
2. 读取 `.reasonix/modules/plan.json` 获取模块依赖关系
3. 使用 Kahn 算法计算拓扑排序 → 划分执行波次
4. 展示调度计划给用户确认（Wave 0/1/2...）
5. 写入 plan.json 的 waves 字段，status → ready

前置：必须先完成 Phase 2（/contracts）
