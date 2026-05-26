启动 modularize skill 的 Phase 5：集成验证 + 失败处理。

**立即执行**：

1. 读取 `~/.claude/skills/modularize/phases/05-integrate.md` 中的所有指令
2. 读取 `.reasonix/modules/plan.json` 和所有 `contracts/*.md`
3. 读取所有 `outputs/*/result.md`
4. 逐模块比对：契约 vs 实际产出
5. 检查上下游接口调用链是否匹配
6. 输出集成验证报告
7. 如有不匹配 → 标记失败模块，提示 `/revise <module>`
8. 全部通过 → plan.json status → done

前置：全部模块完成 Phase 4
