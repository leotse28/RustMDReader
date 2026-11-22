# 修订/批注模式 设计文档（v1.1.0）

日期：2026-11-18
状态：已确认（方案 A：块级源码对 + 渲染期标记注入）
前置调研：Word 修订（Track Changes：插入下划线/删除删除线/作者色、接受/拒绝逐条+全部、审阅窗格、显示三档）与批注（锚定气泡、回复线程、解决/重开、删除、导航）。

## 需求映射

| Word 概念 | 本期实现 |
| - | - |
| 修订开关 | 修订模式（edit.revise，默认 Ctrl+Shift+R），开启自动启用追踪模式 |
| 修改被记录 | 复用追踪编辑浮层：保存按钮变「保存修订」，保存 = 对 (oldSrc,newSrc) 做 token 级 LCS diff 生成待审修订，正文不动 |
| 插入/删除标记 | 预览渲染命中块用标记源码：`<ins>`/`<del>` 内联 HTML；代码块被删行以 `<del>` 伪块显示在新块上方 |
| 接受/拒绝 | 审阅窗格逐条 + 全部（按行号从大到小应用）；接受 = splice 回正文并平移锚点 |
| 批注 | 追踪编辑浮层「＋ 批注」锚定当前块；块右上角序号气泡；线程浮层（回复/解决/重开/删除） |
| 审阅窗格 | 侧栏第四页签「审阅」：统计 + 修订列表（逐条接受/拒绝）+ 批注列表（跳转+开浮层）+ 已解决折叠 |
| 显示三档 / 逐条导航 / @提及 | 本期不做（显示模式与导航按钮留待后续） |

## 数据模型（侧车 JSON）

`<文档名>.md.rev.json`（与 md 同目录）：

```json
{ "version": 1,
  "revisions": [ { "id": "r1", "line": 4, "oldSrc": "…", "newSrc": "…",
                   "author": "…", "time": "ISO8601", "status": "pending" } ],
  "comments":  [ { "id": "c1", "line": 4, "quote": "≤40字", "resolved": false,
                   "replies": [ { "author": "…", "time": "…", "text": "…" } ] } ] }
```

- 按 tab 挂载（tab.rev），打开文档异步加载，变更 500ms 防抖写回；写盘失败 toast 一次、纯内存继续
- 同块再改：oldSrc 恒定、只更新 newSrc（每块至多一条待审修订）
- 作者默认取系统用户名（数据结构含 author/time，本期不做设置 UI）

## 关键机制

- diff：token 级 LCS（CJK 单字 / ASCII 词 / 空白序列为 token）；代码块退化为行级
- 锚点：修订/批注锚定块起始行（data-src-line）；接受修订产生行数差时，所有更大行号锚点同步平移；Ctrl+E 整篇编辑可能使锚点漂移——跳转失配 toast「原文已变化」优雅降级（已知限制）
- 侧车桥接：Rust/Python 各加 `read_sidecar(path)`（不存在返回 Null/None）与 `write_sidecar(path, json)`（原子写）——唯二的桥接改动
- 渲染钩子：markdown.js parse(content, revMap)，命中行用标记源码替代普通源码；无修订零开销
- 批注角标：renderDocIntoPage 后处理附加（不动 markdown.js 渲染）；已解决不显示角标
- 模式互斥：修订模式开启自动开追踪；进 Ctrl+E 编辑模式先提交会话（修订模式下提交 = 创建/更新修订）；Ctrl+S 落盘写正文（原文+已接受）+ 侧车（待审数据）

## 版本规范

APP_VERSION 随交付必升：功能 → 次版本位（本次 1.1.0），修复 → 补丁位；tag 同步；assetVersion 缓存清理自动跟随。

## 测试与验收

- selftest：uiRevToggle、uiRevSaveCreatesRevision、uiRevMarkedRender、uiRevUpdateSameBlock、uiRevAccept、uiRevReject、uiCmtAddReply、uiCmtResolveDelete、uiRevPane
- CDP：真实文件侧车写盘验证 + 截图（修订标记、批注浮层、审阅窗格）
- cargo test（含 parity）；Python unittest；两版 JS 资产字节一致
