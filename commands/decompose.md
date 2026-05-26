启动 modularize skill 的 Phase 1：任务分析 + 模块拆分。

**立即执行**：

1. 读取 `~/.claude/skills/modularize/phases/01-analyze.md` 中的所有指令
2. 严格按照其中的 Step 1 → Step 2 → Step 3 → Step 4 流程执行
3. 第一步先读当前目录结构：`ls -R src/`（如果存在）
4. 分析用户的任务描述 $ARGUMENTS，画出模块依赖图
5. 展示拆分方案给用户确认
6. 确认后写入 `.reasonix/modules/plan.json`

用户任务描述：$ARGUMENTS
