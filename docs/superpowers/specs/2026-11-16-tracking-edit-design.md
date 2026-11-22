# 追踪编辑模式 + 快捷键设置 + 侧栏边缘箭头 设计文档

日期：2026-11-16
状态：已确认（方案 A）
涉及仓库：md-reader-rs（主）、md-reader（Python 版资产同步）

## 背景与目标

阅读模式下点击页面某内容区域，该区域以悬浮层形式在其原位置附近弹出并进入编辑状态；页面其余部分以遮罩突出当前编辑区。保存写回原区域，取消还原。另含两项配套：通用快捷键自定义对话框、侧栏边缘收起箭头。

## 一、追踪编辑模式（核心）

### 状态与入口
- `state.tracking`：模式开关；`state.trackSession`：当前编辑会话（同一时刻最多一个）。
- 工具栏 `btnTrack`（准星图标，激活态样式同 `btnEdit`）；菜单「编辑 → 追踪编辑」；默认快捷键 `Ctrl+Shift+E`（注册表 id `edit.track`，可自定义）。
- 开启后 `#layout` 加 `tracking` 类：`[data-src-line]` 块 hover 显示虚线轮廓 + pointer 光标。

### 点击 → 会话
- `#reader` 事件委托：`e.target.closest('[data-src-line]')` 定位顶层块；追踪模式下链接/图片点击拦截转为编辑。
- 源码区间：收集 DOM 全部 `data-src-line`（升序），本块区间 = `[本块起始行, 下一块起始行-1]`，再去尾部空行；锚点即原始行号（参考链接剔除由 keptIdx 跟踪），区间直接对应 `tab.doc.content`。
- 会话快照 `original` 供取消还原。前置内容（front-matter 等）无锚点、不可编辑——已知限制。

### 悬浮层
- `#trackMask`：fixed 全屏半透明遮罩，拦截滚轮/触摸防滚动，点击 = 保存并关闭。
- `#trackLayer`：fixed，按块 `getBoundingClientRect()` 定位并钳制在视口内（16px 边距）；宽 `max(块宽, 480px)`；高不超视口，textarea 自动伸缩、超限内滚。
- 层头：`编辑块 · L{s+1}–L{e+1}` 标签 + 保存/取消按钮；样式走现有 CSS 变量（主题自适应）、accent 边框 + 阴影；原块加 `.track-target` 轮廓。

### 保存 / 取消 / 退出
- 保存：新文本按行 splice 回 `tab.doc.content` 的 `[s,e]`（块间空行分隔符不在区间内，天然保留）→ 捕获并恢复 `reader.scrollTop` → 整体重渲染（锚点自动重算）→ 关闭会话，追踪模式保持开启；清空文本 = 删除该块。
- 取消：丢弃修改，直接关闭。
- 遮罩点击 / Esc = 保存并关闭（Notion 风格）；`Ctrl+S` 在会话打开时先提交会话再 `saveActive()` 落盘。
- 互斥：进入 Ctrl+E 编辑模式 → 先提交会话并关闭追踪模式；追踪按钮在编辑模式中点击 → 先退编辑模式再开追踪；打印前先提交。

## 二、快捷键设置（通用对话框）

- 命令注册表 `COMMANDS = [{id, label, def, run}]` 覆盖现有全部硬编码键 + `edit.track`；`setupKeyboard()` 重构为「keydown → 组合键归一化（`Ctrl+Shift+E` 格式）→ 查表执行」；Escape 链（查找/灯箱/追踪会话）独立处理。
- 持久化 `cfg.shortcuts = {命令id: 组合键}`（仅存覆盖项），走现有 `saveConfig`；Rust 配置为无模式 JSON，零 Rust 改动。
- 设置对话框复用 quit-modal 样式：每行 = 命令名 + 当前键位 + 「修改」（录入态：按下新组合即录入，Esc 取消）+「恢复默认」；底部「全部恢复默认」；冲突检测：与已有绑定冲突则拒绝并提示。
- 启动及改键后，菜单栏 `<kbd>` 提示与工具栏按钮 title 按注册表刷新。

## 三、侧栏边缘收起箭头

- `#sidebarEdge` 挂在 `#layout`（不放 sidebar 内：隐藏时 sidebar 为负 margin + 透明，会被带走）。
- 贴侧栏右缘垂直居中小竖条按钮（18×36，chevron 图标），点击 `toggleSidebar()`；隐藏态停在屏幕左缘、图标旋转 180°；`left` 过渡 0.22s 与侧栏动画同步。

## 四、测试与验收

- selftest 新增：`uiTrackToggle`、`uiTrackOpenEditSave`、`uiTrackCancel`、`uiTrackOutsideEsc`、`uiTrackSingleSession`、`uiShortcutDialog`、`uiShortcutEffective`、`uiSidebarEdge`；既有断言保持绿。
- CDP 可视化：hover 轮廓、悬浮层+遮罩、保存后原位更新。
- 两版 JS 资产字节一致同步 Python 仓库；cargo test 12/12；Python unittest 全过；INVENTORY §14。

## 边界与已知限制

- 无锚点内容（front-matter、被剔除的参考链接定义行）不可经追踪编辑修改。
- 编辑导致行数变化由整篇重渲染消化，锚点自动重算；不做局部增量渲染。
- 悬浮层编辑期间页面滚动被遮罩拦截；退出后恢复。
