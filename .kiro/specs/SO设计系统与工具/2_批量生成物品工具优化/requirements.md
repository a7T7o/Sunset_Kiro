# 需求文档：批量生成物品工具 Sprite 选择优化

## 简介

优化 `Tool_BatchItemSOGenerator` 编辑器工具的 Sprite 选择区域交互体验。当前工具的 Sprite 选择功能存在预览截断、无法批量筛选、无法排序等问题。本次优化参考 `InventoryBootstrapEditor`（拖拽排序）和 `ItemBatchSelectWindow`（批量选择弹窗）的交互模式，全面升级 Sprite 选择和已选列表的交互能力。

## 术语表

- **批量生成工具（Generator）**：`Tool_BatchItemSOGenerator` EditorWindow，用于批量创建物品 ScriptableObject
- **已选列表（SelectedList）**：工具中 `selectedSprites`（`List<Sprite>`）对应的 UI 列表区域
- **批量选择弹窗（BatchSelectWindow）**：独立的 `EditorWindow`，用于从 Texture/文件夹中网格展示并勾选 Sprite
- **拖拽手柄（DragHandle）**：列表项左侧的 ≡ 图标，用于拖拽排序
- **预测 ID（PredictedID）**：根据列表顺序和起始 ID 自增计算的预览 ID

## 需求

### 需求 1：Sprite 获取按钮区域重构

**用户故事**：作为策划，我想要清晰的按钮区域来获取和批量选择 Sprite，以便快速构建待生成列表。

#### 验收标准

1. WHEN 用户打开批量生成工具时，THE Generator SHALL 在 Sprite 选择区域显示「获取选中项」和「批量选择」两个并排按钮
2. WHEN 用户在 Project 窗口选中 Sprite/Texture/文件夹后点击「获取选中项」按钮，THE Generator SHALL 将选中的所有 Sprite 替换到已选列表中（保留现有功能）
3. WHEN 用户点击「批量选择」按钮，THE Generator SHALL 打开批量选择弹窗

### 需求 2：批量选择弹窗

**用户故事**：作为策划，我想要从一个 Texture 或文件夹中以网格形式浏览并勾选需要的 Sprite，以便精确控制哪些 Sprite 参与生成。

#### 验收标准

1. WHEN 用户在 Project 窗口选中一个 Texture 或文件夹后点击「批量选择」按钮，THE BatchSelectWindow SHALL 遍历选中资源中的所有 Sprite 并以网格形式展示
2. WHEN 用户未在 Project 窗口选中任何有效资源就点击「批量选择」按钮，THE BatchSelectWindow SHALL 显示提示信息告知用户先选择 Texture 或文件夹
3. THE BatchSelectWindow SHALL 为每个 Sprite 显示图标预览和名称
4. THE BatchSelectWindow SHALL 提供搜索框，支持按名称筛选 Sprite
5. THE BatchSelectWindow SHALL 提供「全选」和「取消全选」按钮
6. WHEN 用户勾选若干 Sprite 后点击「添加」按钮，THE BatchSelectWindow SHALL 将选中的 Sprite 按弹窗中的默认排序追加到 Generator 的已选列表末尾，并关闭弹窗
7. WHEN 用户点击「取消」按钮，THE BatchSelectWindow SHALL 关闭弹窗且不修改已选列表

### 需求 3：已选列表完整展示

**用户故事**：作为策划，我想要看到已选列表的全部内容而非截断显示，以便确认所有待生成的 Sprite。

#### 验收标准

1. THE SelectedList SHALL 展示所有已选 Sprite，不进行截断或省略
2. WHEN 已选列表项数超过可视区域高度，THE SelectedList SHALL 提供滚动条浏览全部内容
3. THE SelectedList SHALL 为每项显示：拖拽手柄（≡）、序号、Sprite 图标、名称、预测 ID

### 需求 4：已选列表拖拽排序

**用户故事**：作为策划，我想要通过拖拽手柄调整 Sprite 的顺序，以便控制生成的 ID 分配顺序。

#### 验收标准

1. WHEN 用户按住某项的拖拽手柄并上下拖动，THE SelectedList SHALL 实时移动该项到目标位置
2. WHEN 拖拽排序完成后，THE SelectedList SHALL 根据新顺序重新计算并显示所有项的预测 ID

### 需求 5：已选列表右键菜单

**用户故事**：作为策划，我想要通过右键菜单快速调整列表项的位置或删除，以便精细管理已选列表。

#### 验收标准

1. WHEN 用户右键点击已选列表中的某一项，THE SelectedList SHALL 显示上下文菜单，包含：上移、下移、移到顶部、移到底部、删除
2. WHEN 用户选择「上移」且该项不在首位，THE SelectedList SHALL 将该项与上一项交换位置
3. WHEN 用户选择「下移」且该项不在末位，THE SelectedList SHALL 将该项与下一项交换位置
4. WHEN 用户选择「移到顶部」，THE SelectedList SHALL 将该项移动到列表首位
5. WHEN 用户选择「移到底部」，THE SelectedList SHALL 将该项移动到列表末位
6. WHEN 用户选择「删除」，THE SelectedList SHALL 从列表中移除该项
7. WHEN 列表项位置发生变化后，THE SelectedList SHALL 重新计算并显示所有项的预测 ID

### 需求 6：数据兼容性

**用户故事**：作为开发者，我想要确保优化后的 Sprite 选择功能与现有生成逻辑完全兼容，以便不影响 SO 生成流程。

#### 验收标准

1. THE Generator SHALL 继续使用 `List<Sprite>` 作为 `selectedSprites` 的数据类型
2. WHEN 用户点击生成按钮，THE Generator SHALL 按 `selectedSprites` 列表的当前顺序依次生成 SO，ID 按列表索引自增分配
3. WHEN 拖拽排序或右键菜单操作改变列表顺序后，THE Generator SHALL 直接操作 `selectedSprites` 列表，确保生成时使用最新顺序
