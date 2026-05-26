启动 modularize skill 的 Phase 2：接口契约生成 + 共享前缀。

**立即执行**：

1. 读取 `~/.claude/skills/modularize/phases/02-contracts.md` 中的所有指令
2. 读取 `~/.claude/skills/modularize/templates/contract.template.md` 和 `prefix.template.md`
3. 读取 `.reasonix/modules/plan.json` 获取模块列表
4. 按拓扑顺序为每个模块生成契约（contracts/<module>.md）
5. 逐模块展示给用户审核确认
6. 全部确认后生成 `prefix.md`（仅含项目概述+规范，不含模块专属内容）
7. 更新 plan.json status → contracted

前置：必须先完成 Phase 1（/decompose）
