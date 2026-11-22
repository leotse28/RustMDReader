# 修订/批注 v1.2.0 设计：数据嵌入 md + Word 式侧边批注栏

日期：2026-11-19。前置：v1.1.0 修订/批注（侧车 .rev.json + 角标/浮层/审阅页签）、v1.1.1 伪锚点修复。

## 背景与目标

用户反馈两点：
1. **标记显示不直观**——ins/del 不醒目、批注只是小角标 + 点击浮层，不如 Word 的页边批注框一眼可见。
2. **侧车 .json 文件烦扰**——每次修订/批注都在 md 旁生成 `<doc>.md.rev.json`，希望数据嵌入 md 本身。

目标：数据嵌入 md 文件末尾的 ```mdr-rev 围栏块（用户选定），目录不再出现 .json；审阅 UI 重构为 Word 式右侧批注栏（用户选定）。

## 一、数据嵌入（存储层）

### 格式

文件末尾追加（含数据时）：

````
```mdr-rev
{"version":1,"revisions":[…],"comments":[…]}
```
````

- JSON 结构与 v1.1.0 侧车完全一致（revisions/comments 字段不变）。
- **识别规则**：lang 恰为 `mdr-rev` 的围栏块且位于文件末尾（其后仅允许空白行）才视为数据块；JSON 解析失败 → 按普通代码块渲染、数据忽略（不报错）。
- **无数据不写块**：revisions 与 comments 均为空时不追加（全部接受/拒绝后文件自动回归干净）。

### 生命周期（JS 层剥离/回写，引擎零改动）

- **剥离**：`normalizeDoc(doc)`（renderDoc/activateTab 入口）检测末尾数据块 → 从 content 移除（含块前多余空行归一）→ 解析 JSON 暂存 `doc._rev`；渲染、Ctrl+E 编辑器、追踪会话、修订 diff、data-src-line 锚点全部只见剥离后内容。
- **装载数据**：`loadTabRev(tab)` 优先级 = `doc._rev`（嵌入）> `.rev.json`（旧侧车兜底）；`doc._rev` 存在时跳过侧车读取。
- **回写**：新增 `serializeDoc(tab)` 作为所有保存路径（Ctrl+S saveActive、另存为、退出保存）的统一出口：`tab.rev` 有数据 → content + "\n```mdr-rev\n" + JSON + "\n```"；无数据 → content 原样。
- **迁移清理**：嵌入保存成功后调用新桥接方法 `remove_sidecar(path)` 删除旧 `<path>.rev.json`（不存在则忽略）。此后不再写任何侧车文件。
- **会话恢复**：标签页会话配置同时持久化 `tab.rev`（数据量小），恢复的标签页不丢修订/批注。
- **边界**：用户把数据块移到文件中部 → 不再识别（当普通代码块），下次保存会在末尾追加新块（旧块残留为可见代码块，可手动删）；用户手删数据块 → 修订/批注丢失（与删侧车文件同义）。

## 二、桥接新增 remove_sidecar

- Rust `bridge.rs`：`remove_sidecar(path)` → 删除 `<path>.rev.json`，不存在/失败返回 false，成功 true。
- Python `main.py`：同名方法，`os.remove` + try/except。
- `pywebview-shim.js` METHODS 登记 `"remove_sidecar"`。

## 三、Word 式侧边批注栏（UI 层）

### 布局与显隐

- `#reader`（滚动容器）内、`#content` 之后新增 `<div id="revMargin">`，绝对定位于内容列右缘外 24px、宽 280px，随内容滚动。
- 显隐：当前 tab 的 `tab.rev` 有 revisions 或 comments 且 `cfg.showMarkup !== false` 时显示；工具栏新增「显示标记」切换按钮（btnMarkup，状态持久化）。

### 卡片

- **头部**：`N 修订 · M 批注` 统计 + 「全部接受」「全部拒绝」按钮。
- **修订卡片**：± 图标 + newSrc 首行摘要（title 悬停显示 old→new 完整内容）+ 行内「接受」「拒绝」按钮；点击卡片主体滚动到锚点块。
- **批注卡片**：作者/时间 + quote 引文 + 回复列表 + 回复输入框 + 解决/重开/删除按钮；点击卡片主体滚动到锚点块。
- **定位**：renderDocIntoPage 后（及 resize、字号/缩放变化）按锚点块 offsetTop 逐卡设置 top，碰撞下推（top = max(期望值, 上一卡底边 + 8)）。
- **双向联动**：悬停卡片 ↔ 对应锚点块高亮；点击卡片 → 块 scrollIntoView。

### 正文侧标记

- 锚点块淡色背景 + 左侧强调边（.rev-host 修订 / .cmt-host 批注，可叠加），**取代 cmt-badge 小角标**。
- 行内 ins/del 标记样式保留。

## 四、移除的旧 UI

- 侧栏「审阅」页签（stab + panel + renderReviewPane）——职责并入批注栏头部与卡片。
- `#cmtPop` 线程浮层及其点外关闭/Esc 分支——回复/解决/重开/删除全部在批注卡片内完成。
- `applyCommentBadges` 角标逻辑。

## 五、测试

- 自检改写：uiRevPane → uiMarginCards（卡片数量、统计文本、修订卡片含接受/拒绝按钮）；uiCmtAdd/uiCmtReply/uiCmtResolve/uiCmtDelete 改走卡片断言。
- 自检新增：uiEmbedRoundtrip（带嵌入块的 renderDoc → content 剥离 + tab.rev 解析；serializeDoc 回写含块；无数据不写块）、uiSidecarMigrate（有侧车无嵌入 → 兜底读取）。
- markdown.js 零改动（Node 渲染/数学/mermaid 测试不受影响）。
- Python：remove_sidecar 契约测试。
- 门禁：build + selftest 8 次全绿 + cargo test 12/12 + Node 三套 + Python ALL TESTS PASSED + 资产字节一致同步 + CDP 实机验证（嵌入回写落盘、批注栏渲染、旧 json 自动删除）。

## 六、版本

1.1.1 → **1.2.0**（存储格式变更 + UI 重构，次版本位），tag v1.2.0。
