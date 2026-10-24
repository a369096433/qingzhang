# 开发与发布说明

> 这份文件给维护者看，普通用户请读 [README](README.md)。

## 文件结构

| 文件 | 说明 |
|---|---|
| `index.html` | **仓库跟踪的入口文件**，GitHub Pages 与备份站点都从这里打开 |
| `qingzhang.html` | 本地开发副本（已在 `.gitignore` 中忽略），改完代码必须 `cp qingzhang.html index.html` |
| `guide.html` | 在线图文说明书，配置为同一站点的子页面（`/guide.html`） |
| `小青账使用说明书.html` / `.pdf` | 说明书的源文件与打印版，PDF 用 `make-manual.js` 重新生成 |
| `manual/` | 说明书配图与实测报告 |
| `docs/` | README 用的截图 |
| `manual-test.js` | Playwright 全功能回归脚本（165 项，期望值运行时从 `DATA` 现算，不要硬编码） |

## 两个发布入口

| 入口 | 地址 | 更新方式 |
|---|---|---|
| GitHub Pages（主） | https://a369096433.github.io/qingzhang/ | 推送到 `main` 后由 Actions 自动构建，约 1~2 分钟生效；CDN 另有 10 分钟缓存 |
| WorkBuddy 托管（备） | https://qingzhang-ledger-63298.app.workbuddy.host/ | 重新发布后即时生效 |

> **两个平台必须同时更新**，只更一个会造成「同一版本两种内容」。

## 发版流程

1. 改 `qingzhang.html`，同步把关于页版本号改成新版本（`HTML 复刻版 · vX.Y · 日期`）。
2. `cp qingzhang.html index.html`。
3. 推送 `main`，**等 Actions 的 `pages build and deployment` 变成 `completed`**，再用
   `curl -s https://a369096433.github.io/qingzhang/ | sha256sum` 与本地 `sha256sum index.html` 比对，一致才算生效。
4. 重新发布 WorkBuddy 托管，同样比对 `sha256` 与字节数。
5. 核对三处版本号（关于页 / 本文件版本历史 / `guide.html` 封面 chip）与 README 顶部链接。

## 版本号规则

`v主.次` 递增，同一个「主.次」下的小改可以补第三段：

- **主版本 +1**：数据结构变更、需要迁移或可能影响已有账单的功能重做
- **次版本 +1**：新增功能、界面调整、缺陷修复等常规更新
- **补丁位 +1**（如 `v1.4.1`）：不改变版本主体的同日微调，用户要求时使用

版本号必须同时出现在三处，缺一处就无法判断线上是不是最新：

| 位置 | 形式 |
|---|---|
| App「我的 → 关于小青账」 | `HTML 复刻版 · v1.4.1 · 2026-10-24` |
| `DEVELOPING.md` 本节版本历史 | 表格一行 |
| `guide.html` 封面 chip | `适用 App v1.4.1` |

## 版本历史

| 版本 | 日期 | 主要变化 |
|---|---|---|
| v1.0 | 2026-10-17 | 首个可发布版本：记账、报表、资产、预算、日历、搜索、导入导出、深色模式、PIN 锁 |
| v1.4.1 | 2026-10-24 | **AI 只保留免费模型 + 解析提速**。① 模型清单按服务商价格页重做，只留官方标注「免费」的两个：文本 `XingChenAGI/Xing4.0-29B`（实测非流式 0.85~1.34s、流式首字 189~272ms、多笔拆分与中文数字都对）、视觉 `PaddlePaddle/PaddleOCR-VL-1.5`；收费模型（Qwen3-8B / Qwen3-VL 等）全部移出清单，老配置里的旧模型名在 `aiCfg` 校验时自动回落到清单首项，用户不用手动改。② 文本请求改流式：新增 `readSseText()` 逐块解析 SSE，`aiParse` 透传 `onChunk`，首字到达即点亮提示条「正在理解这句话…（已收到 N 字）」，解决点了按钮几秒没反应的问题；视觉请求不流式。③ `AI_REG` 支持 `noJson` 标记：PaddleOCR 带 `response_format` 时只回 `{"text":"支付成功"}` 会把金额丢掉，改为不传该参数走纯文本。④ 新增 `aiOcrTextFallback()`：从 OCR 纯文本保守提取金额（优先带负号/币符的小数，避免取到余额）、日期、渠道、收款方，认不出就失败不猜；视觉超时 90s→120s（实测热态单张 19~38s）。⑤ 修两处提示词把 `\n` 写成 `/n`（问一问、月度洞察）。验证：`manual-test` 165/165、0 控制台错误；真机实测单笔 860ms、两笔 1.9s、月度洞察 3.7s、截图 19.5s（金额 38.50 与商户均正确回填）。 |
| v1.1 | 2026-10-20 | 日历新增月/年账单统计与逐条明细（按日/按月分段、可切平铺、首屏 30 条 + 加载更多）；App 内新增「使用说明」入口；修复月份切换后选中日期不跟随等缺陷 |
| v1.4 | 2026-10-24 | **（同日二次发布：报表页三卡一屏紧凑化——趋势图 240→152、环形图改 190×190 正方形并与榜单左右并排、榜单最多 5 类+其余合并、AI 卡行高收紧；`.chart-wrap` 高度与 `drawTrendChart`/`drawDonutChart` 内部画布同步改，否则 preserveAspectRatio 会把图压小；`pad.t` 从 18 调回 20、图例组下移到 `pad.t - 11`，否则右上角「支出/收入」的字顶会跑出 viewBox 被裁掉半截；矮屏（`max-height:720px`）再压一档到 120px，两个图表改为读容器实际尺寸自适应；`switchTab` 切换页面时把 `.pages` 的 `scrollTop` 归零，否则会继承上一页的滚动位置，小屏上报表页被顶到中间、首张卡片顶部被容器切掉）** **修 AI 真实调用不可用**。① 超时：默认 `Qwen/Qwen3-8B` 是思考型模型，实测回答 "OK" 都需 17.3s（先输出 `reasoning_content`），真实解析提示词必然超过 60s；改为请求体固定带 `enable_thinking:false`（同请求 0.56s）。② 截图识别失败：原默认视觉模型 `THUDM/GLM-4.1V-9B-Thinking` 已被服务商下架（403 `Model disabled`），改为实测可用的 `Qwen/Qwen3-VL-32B-Instruct`（同一张支付截图 3.4s、七个字段全对），并保留 8B / GLM-4.5V 兜底。③ 通道容错：新增 `aiRequest()`，遇 400 且报文含 `enable_thinking` 或 `json mode/response_format` 时摘掉该参数重试（最多 3 次）；遇 403/404 或 model disabled 则返回 `nextModel`，由 `aiCall` 按清单换下一个（最多 3 个模型）；`aiJson()` 增加 `<|begin_of_box|>` 包裹清洗。④ 错误文案：401/402/429/5xx 分别给人话，402 指向实名认证与充值。⑤ 模型清单按 `/v1/models` 实查结果更新。 |
| v1.3 | 2026-10-24 | **AI 智能助手（方案 A：用户自带 Key）**。`#keypad` 删掉 `date/remark/img/ok` 四个冗余键（date 原本点了没反应，remark 只是聚焦输入框，ok 等同头部保存），改 `repeat(3,1fr)` 三列；新增 `.ai-shot` 整行「截图识别」按钮。`#m-add` 顶部加 `#aiSayRow` 一句话输入，**仅当 `billAmount === "0"` 时显示**。新增 `#m-ai`（Key 设置）、`#m-ai-batch`（多张批量确认）两个弹窗、报表页 `#aiCard`（洞察 + 问一问）、我的页 `ai` 行。JS 新增约 520 行 AI 模块：`aiCfg/aiStoreCfg/aiCall`（OpenAI 兼容通道，60s 超时，401/402/429 分别给可读文案）、六个消毒函数 `aiJson/aiNum/aiStr/aiDate/aiCatId/aiAccId`（分类与账户均闭集匹配）、`aiParse/aiOcr/aiGenReport`、`aiCompress`（canvas 最长边 1400 / JPEG .85，只在内存流转）、批量 `aiDrafts` + `aiRenderBatch`、报告按 `ledgerId|ym` 缓存于 `qingzhang.ai.report.v1`。**重构**：抽出 `createBill(p)` 作为唯一写库入口，`saveBill` 新建分支改为调用它。**未配置 Key 时所有 AI 入口 `hidden`，App 行为与 v1.2 完全一致。** |
| v1.2 | 2026-10-20 | **正式发布的 v1.2（覆盖同号早期误发版）**。① 本月预算对所有账本开放（含旅行账本与自定义账本）：首页预算卡片不再依赖是否已设预算，未设置时给醒目「新增」胶囊 + 「＋ 新增本月预算」大按钮，已设置时可点「修改」，整卡可点；预算弹窗显示「当前账本：xxx」，并兼容缺失预算结构的历史数据。② 分类预算改为可增删改：`#bgCatList` 由"列出全部一级分类"改为"只列已设预算项"，新增 `STATE.bgDraft` 草稿态（增/删/改名先改草稿，点保存才落库，取消即丢弃）；「＋ 新增分类预算」展开 picker，可从已有分类挑选或「新建自定义分类」（写入 `DATA.categories`，记账同步可用）；每行铅笔改名、垃圾桶移出。③ 账本列表每行加铅笔/垃圾桶入口（原来只能点当前账本或右键编辑），新增 `deleteLedger()` 连带清理该账本账单/账户/分类/预算，强制至少保留一个账本；编辑弹窗底部加「删除这个账本」。④ 新建自定义分类改为专用弹窗 `#m-newcat`（不再用 `promptDialog`）：名称输入 + 图标选择按钮，点「选择 ▾」展开 `#ncIconPanel`，图标来自 `ICON_GROUPS`（12 类共 108 个彩色 Emoji，与预设分类同源），点选即写入 `STATE.ncIcon` 并高亮，保存时随分类一起落库 |

## 技术说明

| 项 | 说明 |
|---|---|
| 形态 | 单个 HTML 文件，约 3800 行，无构建、无 npm 依赖 |
| 图标 | 26 个内联 SVG 线性图标（`ICON_PATHS` + `svgOf()`），不依赖图标字体 |
| 图表 | 手写 SVG：三次贝塞尔平滑曲线、渐变面积、环形图、条形柱状图 |
| 设计 | 纸感手账风：米白纸底 + 暖描边 + 衬线数字；深色为暖黑 |
| 存储 | `localStorage`，key `qingzhang.v1` |
| 兼容 | 现代浏览器（Chrome / Edge / Safari / Firefox），移动端优先 |

## 容易踩的坑

- 给自绘按钮写 `display:block` 时，务必备上 `[hidden]{display:none}`，否则浏览器的 `[hidden]` 会被盖掉。
- 新增状态类名先全局 grep，避免撞上已有的组件类（踩过 `.empty` 的坑，把柱状图撑成 80px）。
- 12 列栅格用 `repeat(12, minmax(0, 1fr))`，`1fr` 的自动最小值会被内容撑破。
- 一个工作区只能有一个已发布应用，附加文档只能做成同站子页面，另建目录部署会覆盖原应用。
