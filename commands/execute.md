启动 modularize skill 的 Phase 4：逐模块执行。

**参数**：$ARGUMENTS（可选，指定模块名；为空则执行全部）

**立即执行**：

1. 读取 `~/.claude/skills/modularize/phases/04-execute.md` 中的所有指令
2. 读取 `.reasonix/modules/plan.json` 和 `prefix.md`
3. 如果指定了模块名 → 执行该模块（检查上游依赖是否完成）
4. 如果未指定 → 按波次执行全部（execute-all 模式）
5. 每个模块：
   - 生成 subagent skill 文件到 `.reasonix/modules/skills/modularize-<module>.md`
   - 使用 `/skill modularize-<module>` 启动 subagent 执行
   - 完成后验证 output/result.md
6. 每完成一个模块更新 plan.json 状态
7. Wave 间提示用户进度

前置：必须先完成 Phase 3（/schedule）
