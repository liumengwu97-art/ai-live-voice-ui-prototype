# AI Live Voice Assistant V2.0 — 全新 UI 与布局设计（UI_DESIGN）

> 依据：`unified_spec.md`（统一开发规范 V2.0）§1–§7，并参照 `TEST_PLAN.md` / `AUDIT_CHECKLIST.md` 中 UI-01~07、MODE-01~06、SEC-S04~S07 等条目。
> 交付物：`prototype.html`（可点击高保真原型，单文件、无外部依赖）、`01_console.png … 06_settings.png`（1920×1080 截图）、`07_estop_state.png`（紧急停止状态示意）。
> **原型中所有商品、评论、延迟、显存等数字均为“示例数据”，不是实测结果**；界面里也处处标了“示例数据 · 非实测”。真实数值必须由性能测试回填（规范 §7 严禁伪造）。
> **范围变更（2026-11-18，项目负责人决定）**：PlatformOutputPolicy（平台/地区/账号合规门控、默认禁用直播语音输出）已从范围中移除。直播语音输出（AI TTS → 直播输出设备 / 外置声卡 → iPhone）默认开启，是常规播放路径。

---

## 1. 设计原则

1. **安全优先、始终可见**：P0 紧急停止和人工接管固定在顶栏最右侧，任何页面、任何弹窗下都能一键触发，并支持全局快捷键。
2. **直播输出状态一目了然**：顶栏中央是“直播输出”状态块（绿色）：开关、输出设备（外置声卡 Out 3/4 → iPhone）、实时电平、本地监听模式。关闭后变灰，任务只在本地监听播放。界面任何地方都**不把“电脑播放成功”表述为“手机/直播间已收到”**（这是技术事实）；iPhone/直播间接收状态单独显示，固定为“未验证 · 需实机回环测试”，直到有实机记录。
3. **三路音频分离**：中文麦克风输入、本地监听/试听输出、直播音频输出各有独立的指示灯（顶栏）、链路图（控制台）和设备行（设置页）；AI 合成音频作为数据源单独标出。“本地试听”按钮只在耳机播放，不进直播。
4. **唯一调度出口**：右侧常驻“统一调度队列”，所有可播放内容（三种模式 + 人工播报）都在这里排队，用 P0–P5 色条和 8 种状态标签显示；每条任务都标明输出去向（默认“→ 直播输出”，正在播放的显示“→ 直播输出 + 监听”）。
5. **事实不可被 AI 改写**：价格、币种、尺码、体重范围、材质、包邮门槛、促销规则带 🔒“事实锁”；话术、翻译和评论答复在播出前逐项比对，不一致就拒播并告警。
6. **长时间直播友好**：深色低眩光主题，关键信息高对比，次要信息降到灰阶；状态主要用颜色加文字双编码（照顾色弱用户），动画只用于“直播中/聆听中”的呼吸点。

---

## 2. 信息架构

```
主窗口 QMainWindow
├─ 顶部状态栏（常驻）：品牌 | 会话+计时 | 模式 A/B/C | 当前商品 | ★直播输出（开关/设备/电平/监听） | 资源 GPU/CPU/RAM/端到端延迟 | 设备 麦克风/本地监听/直播输出/iPhone | 人工接管 | ★紧急停止
├─ 左侧导航（常驻）
│   ├─ 直播：① 直播控制台  ② 商品管理  ③ 话术工作台（模式 C）  ④ 实时翻译（模式 B）  ⑤ 评论中心（模式 A 问答）
│   ├─ 系统：⑥ 模型与音频设置
│   └─ 底部：离线可用 / 云端可选 / 评论源授权状态
├─ 中央工作区（QStackedWidget，6 页）
├─ 右侧统一调度队列（常驻 Dock，可停靠到底部）
└─ 底部状态栏：UI 帧耗时 | 推理线程心跳 | SQLite 写入计数 | 最新 AuditLog | 快捷键提示
```

| 模式 | 主要入口 | 产出任务优先级 |
|---|---|---|
| A 商品讲解与评论问答 | 商品管理（多套讲解、商品切换）、评论中心 | P2 高价值评论 / P3 普通评论 / P4 自动讲解 |
| B 中文实时翻译英文 | 实时翻译 | P2 |
| C AI 话术工作台 | 话术工作台 | P1 立即播放 / P4 顺序·轮换·商品切换触发 / P5 定时；评论触发为 P3 |
| 人工 | 控制台“人工立即播报”、任意页“立即播放” | P1 |
| 系统 | 紧急停止 / 人工接管 | P0（不排队，直接抢占） |

---

## 3. 布局栅格（1920×1080 基准）

| 区域 | 尺寸 | 说明 |
|---|---|---|
| 顶部状态栏 | 高 60 px，全宽 | 各信息块固定宽度、不换行；紧急停止 150×44 px |
| 左侧导航 | 宽 208 px | 可折叠为 56 px 图标栏（≤1600 px 宽时自动折叠） |
| 中央工作区 | 1320×992 px | 内边距 16/14 px；卡片间距 12 px；内部使用 12 列网格 |
| 右侧调度队列 | 宽 392 px | 可拖到底部变成横向队列（窄屏或竖屏时） |
| 底部状态栏 | 高 28 px | |

- 间距刻度：4 / 8 / 12 / 16 px；圆角：控件 6 px、卡片 10 px、对话框 12 px。
- 最小窗口：1600×900；更小时队列 Dock 自动转为可隐藏的抽屉，**紧急停止按钮永不隐藏**。
- 多显示器：支持把“实时翻译”或“调度队列”拆成独立窗口（QDockWidget 浮动），放在提词副屏上。

---

## 4. 设计令牌（Design Tokens）

### 4.1 颜色

| Token | 值 | 用途 |
|---|---|---|
| `bg0` | `#0A0E13` | 窗口底色 / 输入框底 |
| `bg1` | `#10161E` | 顶栏、导航、Dock |
| `bg2` | `#151D27` | 卡片 |
| `bg3` / `bg4` | `#1C2633` / `#243142` | 悬停 / 进度底、次级按钮 |
| `line` / `line2` | `#253141` / `#33425A` | 分隔线 / 控件边框 |
| `t1` / `t2` / `t3` | `#E6EDF3` / `#A3B0BE` / `#6C7A8A` | 主文 / 次文 / 弱文 |
| `accent` | `#3B82F6` | 主操作、选中 |
| `ok` | `#22C55E` | 在线、通过、播放中 |
| `warn` | `#F59E0B` | 待审核、低库存、ASR 不确定、iPhone 未验证 |
| `danger` | `#EF4444` | 紧急停止、设备断开、失败、注入拦截 |
| `P0…P5` | `#EF4444` `#F97316` `#EAB308` `#38BDF8` `#A78BFA` `#94A3B8` | 优先级徽标 |

状态标签：QUEUED 灰、PROCESSING 蓝、READY 浅绿、PLAYING 实心绿、COMPLETED 弱灰、CANCELLED 红字、FAILED 实心深红、EXPIRED 琥珀。

### 4.2 字体

- 界面：`Microsoft YaHei UI`, `Segoe UI`（回退 Noto Sans CJK SC）。
- 数字/代码/ID/延迟：`Cascadia Mono`, `Consolas`，等宽，防止跳动。
- 字号：页标题 19 px/600；卡片标题 13.5 px/600；正文 13 px；辅助 11–12 px；关键数值 15–17 px 等宽。
- 实时翻译和评论正文默认 13–15 px，可在设置中整体放大到 125%（提词需要）。

---

## 5. 组件清单

| 组件 | 说明 | PySide6 实现 |
|---|---|---|
| StatusBarTop | 顶部常驻栏，内含以下子件 | `QToolBar`（不可移动）+ 自定义 `QWidget` |
| LiveOutputStatus | 直播输出开关 + 设备 + 电平 + 监听模式，点击弹出详情 | 自定义 `QWidget`（`ToggleSwitch` + `QLabel` + 电平表）+ `QDialog` |
| ResourceMini | 小型数值 + 进度条 | `QLabel` + `QProgressBar`（高 4px） |
| DeviceIndicator | 麦克风/本地监听/直播输出/iPhone 四态：在线/关闭/断开/未验证 | `QToolButton` + 图标 + tooltip |
| EmergencyStopButton | 红色大按钮，P0 | `QPushButton#estop`，接全局热键 |
| TakeoverButton | 暂停自动生成，保留人工 P1 | `QPushButton` |
| NavList | 左侧导航 + 徽标 | `QListWidget` + 自定义 delegate |
| Card | 标题栏 + 内容 | `QFrame#card` + `QVBoxLayout` |
| PriorityBadge / StateChip | P0–P5、8 种状态 | `QLabel` + QSS 动态属性 `[prio="P2"]` |
| QueueItem | 优先级、状态、来源、文本、输出去向、上移/下移/编辑/取消 | `QListView` + `QStyledItemDelegate`，Model 为 `QAbstractListModel` |
| NowPlayingCard | 当前播放、进度、跳过/中断 | 自定义 `QWidget` |
| AudioRouteDiagram | 链路图（麦克风输入 / AI 音频 → 直播输出 / 本地监听 / iPhone 未验证） | `QGraphicsView` 或自绘 `paintEvent` |
| VramBar | 分段显存条（TTS/CUDA 上下文/其他/空闲） | 自绘 `QWidget` |
| Sparkline | CPU/RAM/显存曲线 | `QtCharts.QLineSeries`（或自绘，开销更低） |
| LatencyStage | VAD/ASR/翻译/TTS 首包/播放 五段 + P50/P95 | `QFrame` ×5 + 堆叠比例条 |
| TranscriptTable | 原始识别 / 翻译 / 最终播报三列 | `QTableView` + 富文本 delegate（不确定=黄底、已校验=绿底），第 3 列可编辑 |
| CommentList | 评论流、合并计数、分类、优先级、注入拦截 | `QListView` + delegate |
| VariantGrid | 尺码 × 颜色库存、体重区间 | `QTableView` + `QSpinBox`/`QDoubleSpinBox` delegate |
| ChipInput | 颜色、材质成分 | 自定义 FlowLayout + `QToolButton` |
| FactCheckPanel | 逐项比对结果 | `QListWidget` |
| ScheduleModeSelector | 6 种调度模式 | `QButtonGroup` + `QRadioButton` |
| ProviderCard | 模型提供者、版本、量化、许可证、状态 | `QGroupBox` |
| DeviceRow | 用途 / 设备下拉 / 采样率 / 状态 / 测试 | `QComboBox` + `QLabel` + `QPushButton` |
| Toast / Banner | 非阻塞提示 / 紧急停止横幅 | 叠加 `QWidget` + `QPropertyAnimation` |

---

## 6. 各页面区域与交互

### 6.1 直播控制台（`01_console.png`）
- **页头**：暂停全部自动任务、结束会话（结束时把所有未播任务标为 CANCELLED/EXPIRED）。
- **模式卡 ×3**：A/B/C 开关、运行状态、关键指标（待审评论数、翻译段号与积压、话术库数量）。关闭模式只停止它生成新任务，已在队列中的任务按策略处理（可选择一并取消）。
- **当前讲解商品**：主图、结构化字段（价格 S$、币种 SGD、颜色、尺码 S–2XL、体重 kg、材质 %、库存）、卖点；上一个/下一个（Alt+←/→）。切换前提示“将有 N 个旧商品任务转为 EXPIRED”。
- **音频链路与输出设备**：三行链路图
  1) 麦克风 → VAD/ASR/翻译（CPU，显示重采样与电平）→ 英文文本；
  2) AI 合成音频 → Scheduler/AudioManager → 直播音频输出（播放中，外置声卡 Out 3/4 → iPhone）；
  3) 本地监听输出（可选，同步监听）；iPhone/直播间 **未验证**（琥珀虚框，需实机回环测试）。
  下方固定红条提示（技术事实）：“电脑声卡播放成功 ≠ 手机或直播间已收到声音”。
- **人工立即播报（P1）**：英文原文或“中文→AI 英文”、音色、语速、音量；本地试听 / 加入队列 / 立即播报。“立即播报”经 Scheduler 送直播输出（同时本地监听），“本地试听”只在耳机播放。
- **资源监控**：分段显存条（8 GB，TTS 独占）、CPU/内存/显存曲线、音频缓冲、欠载次数、首包、失败率，以及本场 P50/P95/峰值对照目标表。
- **待处理高价值评论**：前 5 条，点击进入评论中心。

### 6.2 商品管理（`02_products.png`）
- **左栏**：搜索/筛选、商品列表（状态：讲解中/下一个/缺卖点/库存 0）、拖拽导入 CSV/图片、商品切换设置（手动/自动轮换、每个商品讲解时长、跳过零库存）。
- **中栏编辑器**：中英文名、原价/直播价 🔒、币种（下拉，默认 SGD）、总库存（由变体汇总，只读）；材质成分 chip（合计必须等于 100%）；颜色 chip（带英文播报名）；**ProductVariant 表**（尺码 S–2XL × 颜色库存、体重范围 kg、腰围 cm、状态）；物流与售后结构化字段（发货地、时效、包邮门槛 🔒、促销规则 🔒、退换）；SellingPoint 列表（来源：人工 / AI 扩展待审核，需人工采纳）；图片（首张为主图）；AI 英文讲解多套（简洁/详细/促销，每套带校验结果）。
- **右栏**：事实锁说明（哪些字段受保护、修改价格后绑定话术标“需复核”、队列任务 EXPIRED）；绑定话术列表及校验状态（例如“材质与字段不符”标红）。
- 校验错误：字段红框 + 行内说明；保存按钮仍可点，但会列出阻止项。

### 6.3 话术工作台 · 模式 C（`03_scripts.png`）
- **左栏**：分类（开场/讲解/价格促销/尺码/互动/定时提醒/结束语）、模板 ScriptTemplate（占位符 `{price}` `{size}` 等）、按绑定商品筛选。
- **中上 · 话术列表**：多选、批量入队、批量换音色；列：内容、分类、语言、绑定商品、音色、校验、近 30 分钟播出次数（达到上限时标琥珀）。
- **中下 · 编辑器**：中文原文（可原文播报）与英文播报文本左右对照；AI 生成英文 / AI 润色 / 恢复原文；**事实校验面板**（价格/币种/促销逐项比对、违禁词、相似度）；声音角色（只列出有授权记录的音色，“导入真人音色”需提供授权书）、语速、音量；本地试听 / 加入队列（P4）/ 定时（P5）/ 立即播放（P1）/ 保存。
- **右栏 · 调度策略**：手动 / 顺序 / 自动轮换 / 定时 / 评论触发 / 商品切换触发，加上轮换间隔；防重复与容量：最短间隔、30 分钟内最多重复次数、相似度阈值、本模式队列上限、无互动时停止轮换（防无意义循环）。超出限制的任务不入队，并写入 AuditLog。

### 6.4 实时翻译 · 模式 B（`04_translate.png`）
- **页头**：状态（聆听中/已暂停/已停止）、暂停/恢复（F8）、中断当前播报（F9）、停止翻译。
- **分段延迟条**：① VAD 断句 ② 中文 ASR ③ 中→英翻译 ④ TTS 首包 ⑤ 播放输出，每段显示当前值和 P50/P95，最慢的一段高亮；下方堆叠条显示各段占比；右上角显示合计及目标（1–3 s、首包 < 1 s）。
- **右侧控制**：麦克风电平、VAD 灵敏度、英文 TTS 开关、播报去向（直播输出 + 本地监听）、积压段数（超过 3 段时合并或丢弃最旧的一段，并告警）。
- **识别与翻译记录**（核心）：每段一行，三列分开保存、分开显示：
  ① 原始识别文本（ASR raw，不确定的字词黄底虚线）；② 机器翻译原文；③ 最终播报文本（当前段可编辑，已校验的术语/数字绿底，附修正说明，例如“译文币种 yuan 与商品 SGD 不符 → 已按字段修正，原译文保留”）。
  状态：PROCESSING / COMPLETED / 已修正 / 跳过（语气词）。可导出本场 CSV 供事实核查。

### 6.5 评论中心（`05_comments.png`）
- **来源条**：模拟评论源、人工输入（降级路径，始终可用）、TikTok 官方授权接口（未接入时灰色虚线）；标签“非官方接口不作为稳定能力”；统计：收到 / 去重合并 / 待审 / 已答。
- **答复模式**：全自动 / AI 草拟 + 人工审核（默认）/ 仅人工。
- **左栏评论流**：按 尺码/价格/颜色/材质/物流/其他 筛选；每条显示合并计数（×3）、优先级、分类、处理状态；注入或广告评论划线、红色左边框；底部可人工输入评论。
- **中栏详情**：原评论（含来源授权徽标）、相似合并说明、识别意图、抽取实体、匹配到的结构化字段、是否高价值；**输入隔离提示**（评论只作为数据字段传给模型）；AI 草拟答复（可编辑）、其他候选和字段回填模板、中文提词（只显示文字，主播自己口播）；事实校验结果、优先级下拉；本地试听 / 复制文本 / 忽略 / 人工接管 / 审核通过 → 入队（Ctrl+Enter）。顶部 SLA 倒计时，超时自动 EXPIRED。
- **右栏**：分类分布、分级规则（P2：尺码/库存/价格 + 购买意向；P3：一般咨询；闲聊只计数；注入/违禁拦截）、去重窗口、过期时间、每人答复频率限制。

### 6.6 模型与音频设置（`06_settings.png`）
- **模型提供者**（TTS / ASR / LLM·翻译 / 云端 四个标签）：TTS 候选卡（Chatterbox Turbo 350M 首选、Qwen3 TTS 0.6B 对照、Kokoro 82M 轻量回退），显示版本、精度、显存常驻、首包/RTF（“待实测”）、许可证（“待核对”）、来源、采样率、离线可用；切换时**先卸载、确认显存回落后再加载**。ASR（FunASR + SenseVoiceSmall，CPU）、LLM（Qwen3 4B Q4_K_M，llama.cpp，CPU 线程数）。**ModelConfig 记录表**：用途/模型/运行时/量化/来源/许可证/SHA256。云端 API 密钥保存在 Windows 凭据管理器（DPAPI），界面只显示掩码。
- **音频设备 AudioManager**：WASAPI 枚举；中文麦克风输入、本地监听输出、直播音频输出三行，每行有设备下拉、采样率/位深/声道、状态、测试按钮；缓冲、重采样（如 TTS 24k → 48k）；要求本地试听和直播输出必须是不同的物理设备。
- **直播输出与监听**：直播输出总开关（默认开启）、输出音量、本地监听模式（同步监听 / 仅试听 / 关闭）、限幅器、设备断开处理；技术提示“电脑声卡播放成功 ≠ 手机或直播间已收到声音”；“运行 iPhone 回环测试”按钮和 iPhone 收音状态（默认“未实机验证”）。
- **离线自检与告警阈值**：断网启动、模型文件校验、SQLite/日志轮转、TikTok 未连接不影响核心测试；显存告警阈值、欠载告警、TTS 超时、设备断开处理（立即告警 + 暂停出队）。

### 6.7 全局调度队列（右侧 Dock）
- 头部：容量 `活动数 / 上限`；P0 说明（不排队，直接抢占并清空）。
- 正在播放卡：状态、优先级、来源、任务号、输出去向、进度、跳过、中断（F9）。
- 状态筛选：活动 / 全部 / QUEUED / PROCESSING / READY / COMPLETED / CANCELLED / FAILED / EXPIRED。
- 任务项：优先级色块、状态标签、来源（人工、翻译 #段、评论·分类、自动讲解·商品、定时）、文本、附加信息（例如“90s 后过期”“TTS_TIMEOUT · 已重试 1 次”“商品已切换”）、输出去向、上移/下移/编辑/取消。
- 底部：清空 P4/P5、暂停出队、任务日志（SpeechTask 明细：task_id、session_id、product_id、source、priority、original_text、final_text、voice_id、status、创建/开始/结束时间、error_code）。

---

## 7. 键盘快捷键

| 快捷键 | 作用 | 作用范围 |
|---|---|---|
| **F12** | **P0 紧急停止**：立即停止播放，清空队列（CANCELLED），暂停全部自动生成 | 全局（应用内 `QShortcut` + 可选系统级热键 `RegisterHotKey`，应用不在前台也能用） |
| **Esc 连按两次（500 ms 内）** | 紧急停止的备用快捷键 | 应用内；单击 Esc 只关闭弹窗，避免误触 |
| **F10** | 人工接管：暂停 P3–P5 自动生成，P1 人工播报仍可用 | 全局 |
| F9 | 中断当前播报（跳到下一条） | 全局 |
| F8 | 实时翻译 暂停/恢复 | 全局 |
| Ctrl+1…6 | 切换六个页面 | 应用内 |
| Alt+← / Alt+→ | 上一个 / 下一个商品 | 应用内 |
| Ctrl+P | 本地试听当前文本（只在耳机播放） | 编辑区 |
| Ctrl+Enter | 加入队列 / 评论审核通过入队 | 编辑区 |
| Ctrl+Shift+Enter | 立即播报（P1） | 编辑区 |
| Ctrl+S | 保存商品/话术 | 编辑区 |

解除紧急停止需要二次确认，不提供“单键恢复”。

---

## 8. 状态、空状态与错误状态

| 场景 | 表现 |
|---|---|
| 紧急停止已触发 | 中央区顶部红色横幅“P0 紧急停止已触发 · 人工接管中”+ 解除按钮（需确认）；紧急停止按钮变深红并加描边；队列任务全部 CANCELLED（见 `07_estop_state.png`） |
| 直播输出开启（默认） | 顶栏绿色直播输出块，显示设备与实时电平；队列任务去向“→ 直播输出” |
| 直播输出已关闭 | 顶栏块变灰、电平隐藏；任务只在本地监听播放，去向显示“→ 本地监听” |
| iPhone 未实机验证 | iPhone 指示保持“?”，直到回环测试记录写入 |
| 设备断开 | 对应指示灯变红闪烁 + Toast + AuditLog；暂停出队；链路图节点变红；设置页该行显示“已断开 · 重新枚举” |
| CUDA OOM / TTS 失败 | 任务 FAILED（显示 error_code）；自动重试 1 次后回退到 Kokoro（CPU），顶栏显存块变红 |
| 模型加载中 | Provider 卡显示进度和“后台加载中”，UI 可继续操作；相关按钮暂时禁用并给出原因 tooltip |
| 翻译积压 | 积压段数变琥珀色（>3 段变红），自动合并或丢弃最旧的一段并提示 |
| 事实校验失败 | 任务不能入队；红色标签“与字段不符”，列出差异（例如“材质 80% ≠ 字段 70%”） |
| 评论注入 | 划线 + 红边框 + “注入已拦截”，不进入模型，写入 AuditLog |
| 队列为空 | 虚线框“队列为空 · 自动任务将按调度策略生成” |
| 无商品 / 无话术 / 无评论 | 虚线空状态卡 + 主操作（导入 CSV / 从模板新建 / 开启模拟评论源） |
| 离线 | 底部导航显示“离线可用”；云端功能灰显并注明“可选” |
| 内容超长 | 列表两行截断 + tooltip；表格单行省略 |

---

## 9. PySide6 实现映射

### 9.1 窗口结构

```
MainWindow(QMainWindow)
├─ addToolBar(Qt.TopToolBarArea, TopStatusBar)        # 不可移动、不可隐藏
├─ setCentralWidget(QSplitter[NavList | QStackedWidget])
│     QStackedWidget: ConsolePage, ProductPage, ScriptPage, TranslatePage, CommentPage, SettingsPage
├─ addDockWidget(Qt.RightDockWidgetArea, SchedulerDock)  # QDockWidget，可浮动/停靠到底部
├─ setStatusBar(QStatusBar)                               # 线程心跳、SQLite、AuditLog
└─ EStopOverlay(QWidget)                                  # 叠加横幅，raise_() 置顶
```

- 页面切换：`NavList.currentRowChanged → stack.setCurrentIndex`；`QShortcut(QKeySequence("Ctrl+1"))` 等。
- 列表一律用 Model/View（`QAbstractTableModel` / `QAbstractListModel` + `QSortFilterProxyModel`），不要在 `QListWidget` 里塞大量自定义 widget（1000 条模拟评论压测时会卡）。
- 信号只在页面构造时连接一次（对应 UI-07），模式切换只改模型状态，不重新 `connect`。

### 9.2 线程与异步（推理永不阻塞 UI，对应 UI-01 / UI-04）

```
UI 主线程 (Qt event loop)
  └─ 只做：渲染、接收 Signal、发出用户意图（Command）
Core 线程 (QThread + asyncio 事件循环，qasync 或自管 loop)
  ├─ Scheduler（唯一）：PriorityQueue[P0..P5]、状态机、取消/超时、过期清理
  ├─ 输出路由：直播输出 / 本地监听（由 AudioManager 按任务 route 字段分发）
  └─ AuditLog / SQLite 写入（单写线程 + 队列）
推理工作者（独立 QThread 或子进程，各自持有模型，避免 GIL 争用）
  ├─ ASRWorker   (CPU, FunASR+SenseVoiceSmall, VAD)
  ├─ LLMWorker   (CPU, llama.cpp 线程数可配，给 UI/ASR 预留核心)
  └─ TTSWorker   (GPU 独占, 返回 PCM/采样率/时长/首包/推理耗时)
AudioManager 线程（sounddevice/WASAPI 回调；只做 PCM 拷贝，不做推理）
ResourceSampler（QTimer 1s → 后台采样 psutil / NVML → Signal）
```

- 跨线程只用 `Signal/Slot`（`Qt.QueuedConnection`），工作线程**不直接操作控件**。
- 每个任务都是 `asyncio.Task`，带 `asyncio.timeout()`；取消时抛 `CancelledError`，写入结构化异常（error_code、stage、traceback 摘要）。
- 紧急停止的执行路径：`QShortcut/按钮 → Signal estop()` → Core 线程 `scheduler.emergency_stop()`：① `AudioManager.stop_all()`（直接清空输出缓冲）② 取消全部任务 ③ 广播 `stateChanged`。目标是 UI 点击到静音 < 100 ms（待实测）。
- 示例：

```python
class SchedulerBridge(QObject):
    queueChanged = Signal(list)          # 任务快照（不可变 dataclass 列表）
    nowPlaying   = Signal(object)
    liveOutputChanged = Signal(object)   # 开关/设备/电平
    estopTriggered = Signal()

    def __init__(self, loop: asyncio.AbstractEventLoop, scheduler):
        super().__init__(); self._loop, self._s = loop, scheduler

    @Slot()
    def emergency_stop(self):            # 在 UI 线程被调用
        asyncio.run_coroutine_threadsafe(self._s.emergency_stop(), self._loop)

    @Slot(str, int)
    def enqueue(self, text: str, prio: int):
        asyncio.run_coroutine_threadsafe(self._s.submit(text=text, priority=prio, source="manual"), self._loop)
```

### 9.3 关键控件对照

| 原型区域 | 控件 |
|---|---|
| 顶栏信息块 | `QFrame` + `QHBoxLayout`，固定宽度 `setFixedWidth` |
| 直播输出块 | `QFrame#liveOutput`，动态属性 `state=on/off/disconnected`；内含 `ToggleSwitch` 和电平表 |
| 资源条 | `QProgressBar#mini`（textVisible=False, 高 4px） |
| 模式开关 | 自定义 `ToggleSwitch(QAbstractButton)` |
| 链路图 | `QGraphicsScene`（节点 `QGraphicsRectItem`，未验证节点用 `Qt.DashLine` 琥珀色边框） |
| 分段显存条 / 延迟堆叠条 | 自绘 `QWidget.paintEvent` |
| 曲线 | `QtCharts.QChartView`（或自绘 `QPainterPath`，刷新开销更低） |
| 三列翻译表 | `QTableView` + `QStyledItemDelegate` 绘制富文本；第 3 列 `QTextEdit` 编辑器 |
| 变体表 | `QTableView` + `QSpinBox`/`QDoubleSpinBox` delegate；尺码列只读 |
| 话术编辑 | `QPlainTextEdit` ×2 + `QSyntaxHighlighter`（高亮占位符和校验结果） |
| 调度模式 | `QButtonGroup` + `QRadioButton` |
| 评论流 / 队列 | `QListView` + delegate，`QSortFilterProxyModel` 按分类/状态过滤 |
| 设备下拉 | `QComboBox`，数据来自 AudioManager 枚举（`sounddevice.query_devices()` / WASAPI） |
| API 密钥 | `QLineEdit(echoMode=Password)`，读写通过 `keyring`（Windows Credential Manager） |
| 全局热键 | 应用内 `QShortcut(..., context=Qt.ApplicationShortcut)`；系统级用 `RegisterHotKey` + `QAbstractNativeEventFilter` |

### 9.4 QSS 主题片段

```css
* { font-family: "Microsoft YaHei UI", "Segoe UI"; font-size: 13px; color: #E6EDF3; }
QMainWindow, QStackedWidget { background: #0A0E13; }
QToolBar#topStatus, QListWidget#nav, QDockWidget#scheduler { background: #10161E; border: none; }
QFrame#card { background: #151D27; border: 1px solid #253141; border-radius: 10px; }
QLabel[role="muted"] { color: #6C7A8A; font-size: 12px; }
QLabel[role="mono"]  { font-family: "Cascadia Mono", Consolas; }

QPushButton { background: #1C2633; border: 1px solid #33425A; border-radius: 6px; padding: 5px 12px; }
QPushButton:hover { background: #243142; }
QPushButton[kind="primary"] { background: #3B82F6; border-color: #3B82F6; color: #fff; }
QPushButton[kind="warn"]    { background: rgba(245,158,11,0.13); border-color: #F59E0B; color: #F59E0B; }
QPushButton#estop { background: #EF4444; color: #fff; font-size: 15px; font-weight: 700;
                    border: 2px solid rgba(239,68,68,0.35); border-radius: 6px; min-width: 150px; min-height: 44px; }
QPushButton#estop:hover { background: #DC2626; }
QPushButton#estop[active="true"] { background: #7F1D1D; border-color: #EF4444; }
QFrame#liveOutput[state="on"]  { background: rgba(34,197,94,0.13); }
QFrame#liveOutput[state="off"] { background: #1C2633; }
QFrame#liveOutput[state="disconnected"] { background: rgba(239,68,68,0.14); }

QLabel#prio[prio="P0"] { background: #EF4444; } QLabel#prio[prio="P1"] { background: #F97316; }
QLabel#prio[prio="P2"] { background: #EAB308; } QLabel#prio[prio="P3"] { background: #38BDF8; }
QLabel#prio[prio="P4"] { background: #A78BFA; } QLabel#prio[prio="P5"] { background: #94A3B8; }
QLabel#prio { color: #0A0E13; font: bold 11px "Cascadia Mono"; border-radius: 4px; padding: 0 4px; }
QLabel#state[state="PLAYING"] { background: #22C55E; color: #0A0E13; }
QLabel#state[state="FAILED"]  { background: #B91C1C; color: #fff; }
QLabel#state[state="EXPIRED"] { background: rgba(245,158,11,0.14); color: #FCD34D; }

QLineEdit, QPlainTextEdit, QTextEdit, QComboBox, QSpinBox { background: #0A0E13; border: 1px solid #33425A; border-radius: 6px; padding: 4px 8px; }
QLineEdit:focus, QPlainTextEdit:focus, QComboBox:focus { border-color: #3B82F6; }
QTableView { background: #151D27; gridline-color: #253141; selection-background-color: rgba(59,130,246,0.18); }
QHeaderView::section { background: #151D27; color: #6C7A8A; border: none; border-bottom: 1px solid #253141; padding: 6px 8px; }
QProgressBar#mini { background: #243142; border: none; border-radius: 2px; max-height: 4px; }
QProgressBar#mini::chunk { background: #3B82F6; border-radius: 2px; }
QScrollBar:vertical { background: transparent; width: 8px; } QScrollBar::handle:vertical { background: #243142; border-radius: 4px; }
```

改了动态属性之后，要调用 `w.style().unpolish(w); w.style().polish(w)` 刷新样式。

---

## 10. 与审计清单的对应

| 条目 | 设计落点 |
|---|---|
| UI-01 推理不在主线程 | §9.2 线程模型；模型加载时卡片显示“后台加载中” |
| UI-02 六个界面 | 左侧导航 6 页 |
| UI-03 无空壳按钮 | 原型每个按钮都对应一个业务动作（§6）；实现时用 `clicked.connect` 清单逐项核对 |
| UI-05 错误可见 | §8 错误状态、底部 AuditLog、Toast |
| UI-06 试听/直播区分 | 顶栏直播输出块、四个设备指示、控制台链路图、设置页“直播输出与监听”卡 |
| SEC-S04 评论隔离 | 评论中心的“输入隔离”提示与注入拦截 |
| SEC-S06 不改写事实 | 事实锁、校验面板、翻译第 3 列修正说明 |
| SEC-S07 音色授权 | 声音角色只列出授权音色，真人音色导入需授权书 |
| §5 唯一 Scheduler / 状态机 | 右侧统一队列、8 种状态、P0 抢占 |
| §6 SpeechTask 字段 | 队列“任务日志” |

---

## 11. 文件清单

- `prototype.html`：可点击原型。导航可点击，可用 Ctrl+1…6、F12、Esc×2、F10、F8；可按状态筛选队列、取消任务、选择评论、选调度模式、暂停/恢复翻译、开关直播输出、打开直播输出详情。URL 加 `#products` 等锚点可直接打开对应页，加 `?estop` 可直接显示紧急停止状态。
- `01_console.png` `02_products.png` `03_scripts.png` `04_translate.png` `05_comments.png` `06_settings.png`：1920×1080 截图。
- `07_estop_state.png`：紧急停止触发后的界面状态。
- `shot.sh`：用 headless Chrome 重新生成截图的脚本；`src_parts/`：原型的分段源文件（已合并进 prototype.html，仅供后续修改）。
