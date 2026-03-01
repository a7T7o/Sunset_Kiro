# 999_全面重构 / 3_装备与工具 - 子代办

## 前情提要

来源：`.kiro/specs/999_全面重构_26.01.27/3_装备与工具/`
装备系统重构，EquipmentData 创建和 EquipmentService 重构。第一波攻击（EquipmentData.cs）已完成，第二波（EquipmentService 重构）和后续任务未完成。

## 待办详情

| # | 描述 | 优先级 | 来源会话 | 状态 | 分析/见解 |
|---|------|--------|---------|------|----------|
| 1 | 更新 EquipmentSlotUI.Refresh() 兼容新数据结构 | P1 | tasks.md 2.5.1 | 未开始 | UI 层需要适配 EquipmentData 新字段 |
| 2 | 确保 InventoryInteractionManager 兼容新数据结构 | P1 | tasks.md 2.5.2 | 未开始 | 拖拽装备路径需要适配 |
| 3 | 创建 Equipment 输出目录结构 | P2 | tasks.md 3.4 | 未开始 | Assets/111_Data/Items/Equipment/ 及子目录 |
| 4 | 重新生成装备 SO + 验证拖拽装备功能 | P1 | 会话 7-8 遗留 | 待处理 | 旧 SO 数据可能已损坏，需要用批量工具重新生成 |
