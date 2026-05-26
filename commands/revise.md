重新执行 modularize 的一个失败模块（及所有下游模块）。

**参数**：$ARGUMENTS（必须指定模块名）

**立即执行**：

1. 读取 `.reasonix/modules/plan.json`
2. 确认 $ARGUMENTS 模块是否标记为 failed
3. 检查是否需要修正该模块的 contract.md：
   - 如果契约本身有误 → 先修改 contracts/<module>.md
   - 检查下游模块的 contract 是否受影响 → 同步更新
4. 删除该模块的旧 outputs/<module>/result.md
5. 重新执行该模块（Phase 4 流程）
6. 完成后自动 /execute-all 继续执行下游
7. 全部完成后 /integrate 验证

前置：必须先完成过 Phase 4，且存在失败模块
