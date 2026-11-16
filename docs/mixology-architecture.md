# 独家特调（Mixology）模块架构文档

> 本文档描述「独家特调」功能的完整代码结构、模块职责与关键数据流。适合小卷（吉祥物工具）和开发者快速定位代码位置、理解设计意图。

---

## 心智模型

**材料** → 玩家收进**酒柜**，在**吧台**每个槽位挑一件调成**特调（方案）**，特调可命名保存/分享。**对局** = 角色卡 + 特调的一次运行。

---

## 文件总览

### lib/mixology/（领域逻辑层）

| 文件 | 职责 |
|------|------|
| `types.ts` | 全部领域类型定义：材料种类、各类材料结构、特调方案、对局、槽位、条件 |
| `storage.ts` | 本地持久化：酒柜（材料）/ 方案 / 对局，全部走 kv-db；出厂件工厂直读 |
| `assembler.ts` | 装配器：把角色卡 + 各槽材料拼成提示词（system + postHistory + opening） |
| `engine.ts` | 对局运行时：开局、发言生成、重说、继续、回溯、编辑、机括钩子调度 |
| `state.ts` | 记住的值：初始化、按小票原文推进、条件判定、回溯还原 |
| `prose.ts` | 正文处理：从模型原文剥离 [状态栏]/[小剧场] 块、滤网清洗、流式文本处理 |
| `builtin.ts` | 官方出厂件工厂：序言/基底/杯型/核对/朗读，不落酒柜、天然随版本更新 |
| `card-freeform.ts` | 角色卡一框式 ↔ 分框表单的互转逻辑 |
| `css-scope.ts` | 把装饰 CSS 收口到 `.mix-garnish-scope` 下，防止污染页面其他区域 |
| `mechanism-runtime.ts` | 沙盒机括运行时：在 iframe 里跑钩子逻辑，含超时熔断 |
| `trusted-runtime.ts` | 信任模式机括运行时：代码直接在页面里执行，不进沙盒 |
| `mechanism-protocol.ts` | 机括钩子协议：数据包类型、钩子名称枚举、state 合并工具 |
| `connectors.ts` | 连接器调用：宿主代替沙盒发外部请求，参数模板替换、频率配额 |
| `audio-player.ts` | 机括音频播放（mix.play / mix.stop） |
| `hall-client.ts` | 酒材页/配方页云端 API 客户端 |
| `hall-parts.ts` | 配方上架时打包各件材料的辅助逻辑 |
| `mat-thumb.ts` | 材料缩略图生成（小票/尾调的海报截图） |
| `transfer.ts` | 材料导入/导出（本地 JSON 文件） |
| `mascot-tools.ts` | 小卷（吉祥物）工具：供 AI 助手调用，增删改材料/方案 |
| `crafting-guides.ts` | 编辑器里「发给 AI 的制作说明」：每类材料的写法指南文本 |
| `ticket-doc.ts` | 小票渲染代码的 API 文档文本（编辑器速查用） |
| `frame-height.ts` | iframe 内高度上报协议（沙盒向宿主报告实际渲染高度） |

### components/mixology/（UI 层）

| 文件 | 职责 |
|------|------|
| `mixology-app.tsx` | 顶层容器：路由（酒柜/吧台/对局/酒材页）、Toast、全局事件 |
| `mixology-editor.tsx` | 材料编辑表单（底部弹层）：13 类材料各自的字段，含预览 |
| `mixology-game.tsx` | 对局画面：封面背景、消息流、流式输出、机括面板、修改方案弹层 |
| `mixology-hall.tsx` | 酒材页（在线材料）+ 配方页（在线配方）：瀑布/列表 + 详情弹层 |
| `mixology-preview.tsx` | 提示词结构速查弹层（MixStructureSheet）、AI制作说明弹层（MixCraftSheet）、内联预览（MixPreviewInline） |
| `mixology-shared.tsx` | 共用 UI 组件：KindGlyph、MatCard、MixConfirm、MixTagList 等 |
| `prose-view.tsx` | AI 正文渲染：解析 `「」` 对白、`* *` 心声、`【】` 场景、`~ ~` 强调、HTML 片段 |
| `rich-text.tsx` | 开场画布 / 静态小品的 iframe 沙盒渲染（MixRichText） |
| `ticket-frame.tsx` | 小票/尾调的沙盒 iframe 渲染（MixTicketFrame） |
| `mechanism-panel.tsx` | 沙盒机括常驻面板（MixMechanismPanel / MixMechanismInline） |
| `trusted-slot.tsx` | 信任模式机括坑位（MixTrustedSlot）：prose/turn/bottom/float |
| `slot-editor.tsx` | 叠层编辑器（MixSlotEditor）：槽位内材料排序 / 生效条件 / 移除 |

---

## 材料种类（MixMaterialKind）

| 种类 | 标识 | 叠放语义 | 进提示词 | 说明 |
|------|------|----------|----------|------|
| 角色卡 | `character` | first | ✓ | 角色资料+开场白，必填槽 |
| 面具 | `persona` | first | ✓ | 用户人设（{{user}}） |
| 序言 | `preface` | first | ✓ | 提示词最顶端，可覆写各段标题 |
| 基底 | `base` | concat | ✓ | 扮演总纲 |
| 风味 | `flavor` | concat | ✓ | 文风 |
| 杯型 | `glass` | concat | ✓ | 正文输出格式 |
| 苦精 | `strength` | concat | ✓ | 最高优先级要求（排在历史之后） |
| 小票 | `ticket` | concat | ✓ | 状态栏契约进提示词，渲染代码在 iframe |
| 外观 | `garnish` | concat | ✗ | CSS，注入对局画面 |
| 尾调 | `encore` | concat | 契约有则✓ | 小剧场，渲染代码在 iframe |
| 核对 | `checklist` | concat | ✓ | 输出格式检查，提示词末段 |
| 滤网 | `filter` | concat | ✗ | 正则清洗正文 |
| 机括 | `mechanism` | concat | ✗ | 钩子逻辑 + 常驻界面 |

---

## 提示词装配顺序（assembler.ts）

```
序言 → 基底 → 角色资料 → 世界与剧情 → 用户资料 → 风味 → 杯型（含内置正文协议）
→ 状态栏契约 → 小剧场契约 → 示例对话 → 输出格式检查
→ [对话历史]
→ 苦精（postHistory，最靠近生成，权重最高）
```

开场白单独作为首条 assistant 消息，不进系统提示词。

---

## 对局运行时（engine.ts）核心流程

### 玩家发言 → 生成回复（generateMixReply）

```
1. runBeforeSendHooks（机括落杯前钩子：可改写玩家发言、追加临时提示、改写最近一条 assistant）
2. 玩家发言落库
3. runMixGeneration：
   a. assembleFromSession → 装配提示词
   b. buildMixMessages → 拼消息列表（含历史回传裁决）
   c. sendLLMStreamRequest / sendLLMRequest → 拿到模型原文
   d. runMechanismHooks("rawReply") → 机括可在剥块前剪走自己注入的伪装块
   e. stripMixReply → 剥离 [状态栏]/[小剧场] 块 + 进上下文滤网
   f. advanceMixStateWithBlocks → 推进记住的值
   g. runMechanismHooks("afterReply") → 机括可改正文、写记住的值
4. 存档（含 rawText 真原文、机括存储快照）
```

### 其他操作

| 操作 | 函数 | 说明 |
|------|------|------|
| 重说 | `rerollMixReply` | 丢弃最后一条 assistant，回滚状态重生成 |
| 继续 | `continueMix` | 不发言，推进指令不落库 |
| 回溯 | `truncateMixAfterTurn` | 截断后回滚 state + 机括存储 |
| 编辑 | `editMixTurn` | 新原文重跑剥离管线；后续轮若都有 rawText 则 `runMixEditSync` 逐轮重画 |
| 滤网重跑 | `rerunMixFilters` | 对历史全轮按当前规则重洗「进上下文」模式的规则 |

---

## 机括（mechanism）运行时

### 两种模式

**沙盒模式（默认）**
- 钩子代码在 `mechanism-runtime.ts` 管理的 iframe 沙盒里执行
- 断网、超时熔断、碰不到宿主 DOM
- 界面代码（panelHtml）在独立 iframe 里，通过 `window.mix` API 与宿主通信

**信任模式（trusted: true）**
- 代码直接在对局页面执行（`trusted-runtime.ts`）
- 可访问裸 DOM，可自行 fetch，可读写小手机数据
- 坑位：`prose`（正文容器本身）/ `turn`（每轮正文下方）/ `bottom`（所有轮之后）/ `float`（悬浮层）
- 装入/入柜时向玩家明示

### 钩子时机

| 钩子 | 触发时机 | 可返回 |
|------|----------|--------|
| `sessionStart` | 开局后异步补跑一次 | store, state |
| `beforeSend` | 玩家发言落库前 | text（改写发言）, note（追加提示）, sections（挂段）, lastReply（改写最近 assistant）, store, state |
| `rawReply` | 收到模型原文、剥块前 | raw（剪走伪装块）, store, state |
| `afterReply` | 剥块后、finalText 确定前 | text（改写正文）, store, state |
| `sessionEnd` | 退出对局时 | store, state |

---

## 记住的值（state）

- 来源：小票材料里勾了「记住」的项（`MixTicketVar`），每轮从小票原文按键名抽值
- 抽不到时保留上一轮的值（宁停滞不跳变）
- 存在 `MixSession.state`，每轮也快照在 `MixTurn.state`
- 回溯/重说/编辑后从剩下最后一轮的 `MixTurn.state` 还原
- 可用于生效条件判断（`type: "var"`）和宏替换（`{{状态.X}}`）

---

## 生效条件（MixCondition）

四种，纯数据，不执行代码，可随材料分享：

| 类型 | 语义 |
|------|------|
| `turn` | 聊到第 N 轮之后 |
| `var` | 记住的某个值满足比较（> >= < <= = !=） |
| `keyword` | 最近 within 轮里提到过其中任一个词 |
| `chance` | 随机 percent% 的轮次生效 |

角色卡、面具、序言不支持生效条件。

---

## 存储结构（storage.ts）

所有数据存在浏览器 IndexedDB（通过 kv-db 封装）：

| 键 | 内容 |
|----|------|
| `mixology_cabinet_v1` | 酒柜（材料数组） |
| `mixology_recipes_v1` | 特调方案数组 |
| `mixology_sessions_v1` | 对局数组 |
| `mixology_profile_v1` | 创作者资料（署名/头像） |
| `mixology_connectors_v1` | 连接器配置 |

官方出厂件不落库，工厂函数直读（`builtin.ts`）。

---

## 历史回传裁决

默认只回传最近一条 assistant 的状态栏/小剧场块（格式示范 + 状态接续足够）。

每件小票/尾调可自定 `historyFeed`：
- `latest`（默认）：只最近一轮
- `all`：全部回传（token 随轮数增长）
- `none`：完全不回传

---

## 云端功能（需 Supabase）

酒材页（材料分享）和配方页需要自建 Supabase 并建表。自部署时这两个页面显示「还没开张」，本地吧台/酒柜完全不受影响。

相关文件：`hall-client.ts`（API 调用）、`hall-parts.ts`（上架打包）、`mat-thumb.ts`（封面截图）。
