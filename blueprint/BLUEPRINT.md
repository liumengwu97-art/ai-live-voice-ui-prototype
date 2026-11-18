# AI Live Voice Assistant V2.0 — UI 设计图纸（BLUEPRINT）

> **致 Claude：** 本文档是 PySide6 界面的施工图，请严格按这里的尺寸、配色、组件和交互实现 UI 与排版。
> 视觉基准是 `prototype.html`（可在浏览器 1920×1080 下打开对照），所有标注图都在本目录。
> **范围说明（2026-11-18 项目负责人决定）：** PlatformOutputPolicy（平台/地区/账号/场景合规门控、默认禁用直播语音输出）**已从范围中移除，请勿实现**。直播语音输出（AI TTS → 直播输出设备 / 外置声卡 → iPhone）**默认开启**，是常规播放路径；用户可在顶栏或设置页手动关闭。
> 仍须保留：P0 紧急停止、人工接管；麦克风输入 / 本地监听 / 直播输出三路设备分开显示；iPhone“需实机回环验证”状态；“电脑声卡播放成功 ≠ 手机已收到声音”技术提示；价格等事实字段锁定；界面上的“示例数据”标签（真实数值须由测试回填，严禁伪造）。

## 目录
1. 全局布局与缩放规则 · 2. 设计令牌 · 3. 顶栏 / 导航 / 队列 / 状态栏 · 4. 六个页面逐区规格 · 5. 紧急停止状态 · 6. 组件规格 · 7. QSS 主题 · 8. 控件树与对象名 · 9. 线程规则 · 10. 快捷键 · 11. 状态 / 空状态 / 错误状态 · 12. 验收清单

**图纸文件**

| 文件 | 内容 |
|---|---|
| `00_global_layout.png` | 五区外壳、尺寸、12 列栅格、Qt 对象树、缩放规则 |
| `01_console.png` … `06_settings.png` | 六个页面：原型截图（缩放 0.66）+ 橙色编号 + 青色尺寸线 + 图例（组件 / 尺寸@坐标 / Qt 控件 / 数据 / 交互） |
| `07_estop_state.png` | 紧急停止触发后的状态 |
| `08_design_tokens.png` | 颜色、P0–P5、8 种任务状态、字体、间距、圆角、指示器 |
| `09_components.png` | 组件表 |
| `theme.qss` | 第 7 节的 QSS 主题文件（可直接加载） |
| `prototype.html` | 可点击原型（视觉与交互基准） |

> 标注图里的“尺寸@坐标”是原型在 1920×1080 窗口中实测的逻辑像素，坐标原点为窗口左上角。实现时用布局管理器复现**固定尺寸 + 伸缩比例**，不要用绝对坐标。

---

## 1. 全局布局

![全局布局](00_global_layout.png)

| 区域 | 尺寸（1920×1080 基准） | Qt 实现 | 说明 |
|---|---|---|---|
| 顶部状态栏 | 全宽 × **60** | `QToolBar#topStatus`（不可移动、不可隐藏） | 信息块固定宽度、不换行，见 §3.1 |
| 左侧导航 | **208** × 992 | `QListWidget#nav` | 导航项 191×40，左右外边距 8 |
| 中央工作区 | **1320**（伸缩）× 992 | `QStackedWidget#pages`（6 页） | 页内边距 上下 14 / 左右 16 → 内容区 1288×964；区块间距 12 |
| 右侧调度队列 | **392** × 992 | `QDockWidget#schedulerDock` | 可浮动、可停靠底部、可收起 |
| 底部状态栏 | 全宽 × **28** | `QStatusBar#statusBar` | 线程心跳、SQLite、AuditLog、快捷键提示 |

- 分区之间用 1px `#253141` 分隔线；顶栏、导航、队列、状态栏背景 `#10161E`，中央背景 `#0A0E13`。
- 页面统一结构：`pageHeader`（高 34：标题 19px/600 + 副标题 12px + “示例数据”标签 + 右侧操作按钮）→ 区块（`QFrame#card`，圆角 10，标题栏高约 38，内容 padding 10/12）。
- 页内栅格参考：12 列、槽宽 12（列宽约 96.3）；各页实际列宽见 §4。

**缩放与最小窗口**
- 建议最小窗口 **1600×900**（`setMinimumSize(1600, 900)`）。
- 宽度 < 1760：导航折叠为 56px 图标栏（隐藏文字和快捷键，tooltip 显示名称）。
- 宽度 < 1600 或用户手动收起：队列 Dock 收为抽屉（顶栏出现“队列 (n)”按钮可呼出）。
- **紧急停止按钮在任何尺寸、任何页面、任何弹窗下都不隐藏、不被遮挡**。顶栏空间不足时，按 会话 → 模式 → 当前商品 的顺序先省略文字。
- 中央区随窗口伸缩；卡片内部用 `QScrollArea` 滚动，不压缩固定高度的行（如控制台中行 392）。
- 高 DPI：使用 Qt6 默认高 DPI 缩放；所有尺寸以 100% 缩放下的逻辑像素为准。

---

## 2. 设计令牌

![设计令牌](08_design_tokens.png)

| Token | 值 | 用途 |
|---|---|---|
| bg0 | `#0A0E13` | 窗口底色、输入框底 |
| bg1 | `#10161E` | 顶栏 / 导航 / 队列 / 状态栏 |
| bg2 | `#151D27` | 卡片 |
| bg3 | `#1C2633` | 悬停、次级按钮 |
| bg4 | `#243142` | 进度底、强悬停 |
| line / line2 | `#253141` / `#33425A` | 分隔线 / 控件边框 |
| t1 / t2 / t3 | `#E6EDF3` / `#A3B0BE` / `#6C7A8A` | 主 / 次 / 弱文字 |
| accent / accent2 | `#3B82F6` / `#60A5FA` | 主操作、选中、焦点 |
| ok | `#22C55E` | 在线、通过、播放中、直播输出开启 |
| warn | `#F59E0B` | 待审、低库存、ASR 不确定、iPhone 未验证、🔒 |
| danger | `#EF4444` | 紧急停止、断开、失败、注入拦截 |
| accentBg / okBg / warnBg / dangerBg | 对应色 α .14 / .13 / .13 / .14 | 标签与块背景 |
| P0–P5 | `#EF4444` `#F97316` `#EAB308` `#38BDF8` `#A78BFA` `#94A3B8` | 优先级徽标 26×18，文字 `#0A0E13` |

任务状态：QUEUED 灰底、PROCESSING 蓝、READY 浅绿、**PLAYING 实心绿**、COMPLETED 弱灰、CANCELLED 红字、**FAILED 实心深红**、EXPIRED 琥珀（精确色值见 theme.qss）。

**字体：** 界面用 `Microsoft YaHei UI`（回退 Segoe UI）；数字、ID、延迟、价格用 `Cascadia Mono` / `Consolas`。字号：H1 19/600 · H3 13.5/600 · 正文 13 · 辅助 11–12 · 关键数值 15–17/600 · kbd 10.5。
**间距：** 4 / 8 / 12 / 16 / 24。**圆角：** 标签 4 · 控件 6 · 卡片 10 · 对话框 12。
状态一律用**颜色 + 文字**双编码，不只靠颜色区分。

---

## 3. 外壳组件

### 3.1 顶部状态栏（高 60，从左到右）

| 顺序 | 块 | 宽 | 对象名 | 内容 / 交互 |
|---|---|---|---|---|
| 1 | 品牌 | 208 | `QLabel#brand` | 30×30 渐变 Logo + “AI 直播语音助手 / Live Voice Assistant V2.0” |
| 2 | 会话 | 196 | `QFrame#session` | “会话 [示例]” + 呼吸点 + 会话号 + 计时（等宽，每秒刷新） |
| 3 | 模式 | 164 | `QFrame#modes` | 三个 Chip：A 讲解 / B 翻译 / C 话术，开启为 accent |
| 4 | 当前商品 | 206 | `QFrame#currentProduct` | 商品名（省略号）+ 直播价（琥珀等宽） |
| 5 | **直播输出** | 300 | `QFrame#liveOutput` | 第一行：📡 “直播输出：开启” + 5 条电平 + ToggleSwitch；第二行：设备“外置声卡 Out 3/4 → iPhone · 48k”；第三行：“本地监听：同步 · iPhone 收音未实机验证”（琥珀）。`state=on/off/disconnected`。点击整块 → `LiveOutputDialog`；开关点击不冒泡 |
| 6 | 资源 ×4 | 315 | `ResourceMini ×4` | GPU 显存 x/8.0G · CPU% · 内存 x/32G · 端到端 P95；每项 60 宽（端到端 74），带 4px 进度条 |
| 7 | 设备 ×4 | 225 | `DeviceIndicator ×4` | 麦克风 / 本地监听 / 直播输出 / iPhone(?)，每个 48 宽，tooltip 显示设备名 |
| 8 | 弹性 | stretch | — | |
| 9 | 人工接管 | 97×40 | `QPushButton#takeover` | 琥珀描边，“人工接管 / F10 · 暂停自动” |
| 10 | **紧急停止** | 150×44 | `QPushButton#estop` | 红底白字“■ 紧急停止 / P0 · F12 / Esc×2”，右边距 12 |

### 3.2 左侧导航（208）
- 分组标题“直播”“系统”（11px t3，高 37）。导航项 191×40：图标 18 + 间距 8 + 文字 + 右侧快捷键 kbd 或徽标（LIVE 绿 / 待审数红）；**不换行**。
- 选中：accentBg 背景 + 左侧 3px accent 竖条。顺序：直播控制台 Ctrl+1 · 商品管理 Ctrl+2 · 话术工作台 Ctrl+3 · 实时翻译 Ctrl+4 · 评论中心 Ctrl+5 · 模型与音频设置 Ctrl+6。
- 底部 `navFoot`（高 140）：离线可用 / 云端未启用（可选）/ TikTok 评论未接入授权源 / “数值为示例”。

### 3.3 右侧统一调度队列（392，图 07 有标注）
| 区 | 高 | 内容 |
|---|---|---|
| 头部 `queueHeader` | 82 | “统一调度队列” + Scheduler 标签 + 容量 `活动/30`；说明行；P0 说明“紧急停止/人工接管：不排队，直接抢占并清空” |
| 正在播放 `NowPlayingCard` | 106（左右边距 12） | 状态 PLAYING + 优先级 + 来源 + **去向“→ 直播输出 + 监听”**；文本；绿色进度条；时间；跳过 / 中断 F9 |
| 状态筛选 `stateFilter` | 61 | Chip：活动 n / 全部 / QUEUED / PROCESSING / READY / COMPLETED / CANCELLED / FAILED / EXPIRED |
| 列表 `QListView#queueList` | stretch | QueueItem（见 §6）；活动任务按 P1→P5 排序，同级先进先出；下方“最近结束”分组 |
| 底部 `queueFooter` | 40 | 清空 P4/P5 · 暂停出队 · 任务日志（SpeechTask 明细） |

队列项去向：直播输出开启时显示“→ 直播输出”，关闭时显示“→ 本地监听”。

### 3.4 底部状态栏（28）
UI 主线程帧耗时 · 推理线程心跳（ASR/LLM/TTS 圆点）· SQLite 写入计数 · 最新一条 AuditLog · 快捷键提示 · “原型 · 示例数据”标签。

---

## 4. 页面逐区规格
图中橙色编号对应下表 #。尺寸为 1920×1080 窗口下的实测值；“stretch”列在更大窗口里按比例伸缩。

### 01 直播控制台 Console（`ConsolePage`）

![01 直播控制台 Console](01_console.png)
**布局：**
- QVBoxLayout(页) spacing 12：页头 34 → 模式卡行 87（QGridLayout 3 等分）→ 中行 392（5fr : 7fr）→ 底行 stretch（4fr : 5fr : 4fr）
- 中行高度固定 392；底行占剩余空间（最小 300）。窗口变矮时底行三卡内部滚动，中行不压缩
- 链路图节点网格：170 | 34 | 1fr | 34 | 230（节点最小高 52）；iPhone 节点琥珀虚线 = 未验证
- “电脑播放成功 ≠ 手机已收到”提示条常驻，不可关闭
- 主网格实测列宽：`388.922px 486.141px 388.922px`
| # | 组件 | 尺寸 @坐标 (px) | Qt 控件 / 对象 | 数据绑定 | 交互 / 状态 |
|---|---|---|---|---|---|
| 1 | 页头 PageHeader | `1288×34 @(224,74)` | QWidget#pageHeader + QHBoxLayout | 页标题、示例数据标签 | 暂停全部自动任务；结束会话（未播任务转 CANCELLED/EXPIRED，需确认） |
| 2 | 模式卡 ×3 ModeCard | `1288×87 @(224,120)` | QFrame#card ×3 + ToggleSwitch | ModeState{A,B,C}: 运行状态、待审数、段号/积压、话术数 | 开关模式：只停止产生新任务 |
| 3 | 当前商品卡 CurrentProductCard | `532×392 @(224,219)` | QFrame#card；QLabel 主图 118×150；QFormLayout | Product + ProductVariant（价格/SGD/颜色/尺码/kg/材质/库存/卖点） | 只读展示；🔒字段不可在此编辑 |
| 4 | 商品切换 ProductSwitcher | `207×23 @(536,229)` | QPushButton ×2 + QLabel(kbd) | 当前商品索引 | Alt+←/→；切换前提示 N 个旧任务将 EXPIRED |
| 5 | 切换影响提示 InfoNote | `506×49 @(237,551)` | QLabel#note[kind=info] | 待过期任务数 | — |
| 6 | 音频链路卡 AudioRouteCard | `744×392 @(768,219)` | QFrame#card + QGraphicsView(AudioRouteDiagram) | AudioManager 设备状态、电平、直播输出开关 | 设备设置→跳转设置页；输出详情→弹出 LiveOutputDialog |
| 7 | AI 音频→直播输出链路 | `718×67 @(781,334)` | RouteNode(QFrame) ×3 + 箭头(自绘) | TTS 引擎、Scheduler、直播输出设备与播放状态 | 节点状态：在线绿/断开红/关闭灰 |
| 8 | iPhone 节点（未验证） | `230×52 @(1269,411)` | RouteNode[state=unverified] 琥珀虚线边框 | iPhone 回环测试结果（默认无） | 只有回环测试记录写入后才转为“已验证” |
| 9 | 技术提示条 TechHint | `718×49 @(781,471)` | QLabel#note[kind=error] 常驻 | 固定文案 | 不可关闭：电脑播放成功 ≠ 手机已收到 |
| 10 | 人工立即播报 P1 ManualBroadcast | `389×415 @(224,623)` | QFrame#card；SegmentedButton；QPlainTextEdit；QComboBox | 文本、音色 VoiceProfile、语速、音量 | Ctrl+P 本地试听；加入队列；Ctrl+Shift+Enter 立即播报(P1→直播输出+监听) |
| 11 | 资源监控 ResourceMonitor | `486×415 @(625,623)` | QFrame#card；Sparkline(自绘) ×3 | ResourceSampler 1s：CPU/RAM/显存/缓冲/欠载/首包/失败率 | 只读；阈值超限变色 |
| 12 | 分段显存条 VramBar | `460×22 @(638,695)` | VramBar(QWidget.paintEvent) 高 22 | NVML：TTS/CUDA 上下文/其他/空闲，总 8GB | tooltip 显示各段数值 |
| 13 | 本场延迟表 LatencyTable | `460×124 @(638,814)` | QTableView(只读) | P50/P95/峰值/目标（示例值，须实测回填） | — |
| 14 | 高价值评论 TopComments | `389×415 @(1123,623)` | QListView + CommentDelegate(紧凑) | CommentEvent 前 5 条（P2 优先） | 点击→评论中心并选中该条 |

### 02 商品管理 Products（`ProductPage`）

![02 商品管理 Products](02_products.png)
**布局：**
- 三栏 QHBoxLayout：列表 270（固定）| 编辑器 stretch（实测 704）| 右栏 290（固定）；高度填满 918
- 编辑器内容在 QScrollArea 中纵向滚动；字段行 6 列：1.4 : 1.4 : 1 : 1 : 0.8 : 0.9
- 🔒 字段（价格/币种/尺码/体重/材质/包邮门槛/促销规则）用琥珀锁图标；AI 输出不可写入这些字段
- 右栏上下两卡：事实锁（内容高）+ 绑定话术（stretch）
- 主网格实测列宽：`270px 704px 290px`
| # | 组件 | 尺寸 @坐标 (px) | Qt 控件 / 对象 | 数据绑定 | 交互 / 状态 |
|---|---|---|---|---|---|
| 1 | 页头 | `1288×34 @(224,74)` | QWidget#pageHeader | — | 导入 CSV/Excel（后台解析+逐字段校验）；新建商品 |
| 2 | 商品列表栏 ProductList | `270×918 @(224,120)` | QFrame#card 270 宽；QLineEdit 搜索；QListView | ProductRepository 列表 + 状态标签 | 选择→编辑器加载；拖拽导入 |
| 3 | 商品切换设置 | `244×172 @(237,590)` | SegmentedButton + QSpinBox + ToggleSwitch | 切换模式、讲解时长、跳过零库存 | 切换时旧商品任务 EXPIRED |
| 4 | 编辑器标题栏 | `702×42 @(507,121)` | QWidget + QPushButton×3 | 商品名、SKU、保存状态 | AI 扩展卖点(需审核)；设为当前讲解；Ctrl+S 保存 |
| 5 | 基础字段（6 列） | `678×53 @(519,173)` | QLineEdit/QDoubleSpinBox/QComboBox；🔒价格 | name_zh/name_en/price_original/price_live/currency=SGD/stock(汇总只读) | 校验失败红框+行内说明 |
| 6 | 材质成分 / 颜色 ChipInput | `678×71 @(519,238)` | ChipInput(FlowLayout+QToolButton) | 材质%(合计=100) 🔒；颜色(中/英播报名) | 添加/删除 chip |
| 7 | 尺码×颜色变体表 VariantGrid | `678×216 @(519,350)` | QTableView + QSpinBox delegate | ProductVariant：尺码 S–2XL🔒、体重 kg🔒、腰围、各色库存 | 编辑库存；低库存高亮 |
| 8 | 物流与售后字段 | `678×78 @(519,578)` | QComboBox/QLineEdit ×5 | 发货地、时效、包邮门槛🔒、促销规则🔒、退换 | 评论问答只引用这些字段 |
| 9 | 卖点 / 图片 | `678×181 @(519,668)` | QTableView(拖拽排序) + 图片网格 74×74 | SellingPoint(来源：人工/AI 待审核)；图片(首张主图) | 采纳 AI 卖点；拖拽排序；添加图片 |
| 10 | AI 多套讲解 | `678×127 @(519,861)` | QFrame ×3 | 生成的讲解(简洁/详细/促销) + 校验状态 | 重新生成（后台线程） |
| 11 | 事实锁说明 FactGuard | `290×271 @(1222,120)` | QFrame#card 290 宽 | 受保护字段清单 | 改价后绑定话术标“需复核” |
| 12 | 绑定话术 BoundScripts | `290×635 @(1222,403)` | QListView | Script(product_id) + 校验结果 | 去工作台 |

### 03 话术工作台 Scripts（模式 C）（`ScriptPage`）

![03 话术工作台 Scripts（模式 C）](03_scripts.png)
**布局：**
- 三栏：分类 200 | 中栏 stretch（764）| 调度 300；中栏上下：话术列表 280（固定）+ 编辑器 stretch
- 编辑器 2×2：中文原文 / 英文播报（各高 96）；事实校验 / 音色语速音量；底部操作行右对齐，立即播放为主按钮
- 调度卡：6 个单选卡片 + 轮换间隔；下方“防重复与容量限制”固定在卡片底部（border-top 分隔）
- 主网格实测列宽：`200px 764px 300px`
| # | 组件 | 尺寸 @坐标 (px) | Qt 控件 / 对象 | 数据绑定 | 交互 / 状态 |
|---|---|---|---|---|---|
| 1 | 页头 | `1288×34 @(224,74)` | QWidget#pageHeader | — | 从模板新建；AI 批量生成 3 套；新建话术 |
| 2 | 分类/模板栏 | `200×918 @(224,120)` | QFrame#card 200 宽；QListWidget | Script 分类计数、ScriptTemplate、商品筛选 | 点击过滤列表 |
| 3 | 话术列表 ScriptTable | `764×280 @(436,120)` | QTableView + 复选框列 | 内容/分类/语言/商品/音色/校验/近30分钟播出 | 多选批量入队、批量换音色、搜索 |
| 4 | 话术编辑器 ScriptEditor | `764×626 @(436,412)` | QFrame#card | 当前 Script | AI 生成英文/AI 润色/恢复原文（后台） |
| 5 | 中文原文 | `363×116 @(449,465)` | QPlainTextEdit + 占位符高亮 | original_text（含 {price_live} 等占位符） | 可原文播报 |
| 6 | 英文播报文本 | `363×116 @(824,465)` | QPlainTextEdit + QSyntaxHighlighter | final_text（占位符由结构化字段回填） | 编辑后重新校验 |
| 7 | 事实校验面板 FactCheckPanel | `363×392 @(449,593)` | QFrame + QLabel 富文本 | 价格/币种/促销比对、违禁词、相似度 | 不通过则禁用入队/播放 |
| 8 | 音色/语速/音量 | `363×392 @(824,593)` | QComboBox + QSlider ×2 | VoiceProfile(仅授权音色)、speed、volume | 真人音色导入需授权书 |
| 9 | 操作行 | `738×30 @(449,998)` | QPushButton ×5 | — | 本地试听；加入队列 P4；定时 P5；立即播放 P1；保存 |
| 10 | 调度策略卡 | `300×918 @(1212,120)` | QFrame#card 300 宽 + ToggleSwitch | SchedulerConfig(模式 C) | 开关自动调度 |
| 11 | 6 种调度模式 | `298×627 @(1213,160)` | QButtonGroup + QRadioButton(卡片样式) | 手动/顺序/自动轮换/定时/评论触发/商品切换触发 + 间隔 | 单选 |
| 12 | 防重复与容量限制 | `298×250 @(1213,787)` | QSpinBox/QDoubleSpinBox ×4 + ToggleSwitch | 最短间隔、30 分钟上限、相似度阈值、队列上限 | 超限不入队并写 AuditLog |

### 04 实时翻译 Translate（模式 B）（`TranslatePage`）

![04 实时翻译 Translate（模式 B）](04_translate.png)
**布局：**
- 页纵向：页头 34 → 控制卡（内容高，内部 1fr | 330 两栏）→ 记录卡 stretch
- 延迟五段等宽 QFrame（最慢段琥珀边框）+ 高 10 占比条；右栏 4 行：麦克风电平 / VAD / TTS 去向 / 积压
- 记录表行网格：54 | 1fr | 1fr | 1fr；最新段在顶部，当前段浅蓝底且第 3 列可编辑
- 三列文本（原始识别 / 机器翻译 / 最终播报）分别保存到数据库，不得合并
| # | 组件 | 尺寸 @坐标 (px) | Qt 控件 / 对象 | 数据绑定 | 交互 / 状态 |
|---|---|---|---|---|---|
| 1 | 页头 + 状态 | `1288×34 @(224,74)` | QWidget#pageHeader + StatusChip | 聆听中/已暂停/已停止 | — |
| 2 | 翻译控制按钮 | `349×30 @(1163,76)` | QPushButton ×3 | — | F8 暂停/恢复；F9 中断当前播报；停止翻译 |
| 3 | 五段延迟 LatencyStage ×5 | `916×73 @(237,156)` | QFrame ×5（等宽） | VAD/ASR/翻译/TTS 首包/播放：当前值 + P50/P95 | 最慢段琥珀高亮 |
| 4 | 延迟占比条 | `916×10 @(237,237)` | 自绘 QWidget 高 10 | 各段占比 | — |
| 5 | 麦克风/VAD/TTS/积压 | `330×116 @(1169,131)` | LevelMeter + QSlider + ToggleSwitch + QLabel | 麦克风电平、VAD 灵敏度、播报去向(直播输出+本地监听)、积压段数 | 积压>3 段合并/丢弃最旧并告警 |
| 6 | 识别与翻译记录 TranscriptTable | `1288×768 @(224,270)` | QTableView + RichTextDelegate | Segment{raw_asr, mt_text, final_text, status, note} | 导出本场 CSV |
| 7 | 三列表头 | `1286×33 @(225,313)` | QHeaderView | ①原始识别 ②机器翻译 ③最终播报 | 三段文本分别保存 |
| 8 | 当前段（可编辑） | `1286×65 @(225,346)` | 第 3 列 QTextEdit 编辑器 | PROCESSING 段 | 编辑后送 TTS |
| 9 | 已修正段 | `1286×59 @(225,608)` | 状态标签 + 说明行 | 币种 yuan→S$ 按字段修正，原译文保留 | — |
| 10 | 图例 | `308×23 @(1191,280)` | QLabel ×2 | 黄=ASR 不确定；绿=术语/数字已校验 | — |

### 05 评论中心 Comments（模式 A 问答）（`CommentPage`）

![05 评论中心 Comments（模式 A 问答）](05_comments.png)
**布局：**
- 页纵向：页头 34 → 来源条（内容高 40）→ 三栏 stretch：评论流 400 | 详情 stretch（594）| 统计 270
- 评论流底部固定人工输入行；详情卡内：原评论 → 意图/实体行 → 隔离提示 → 草拟答复(84) → 候选(stretch) → 校验行 → 操作行
- 注入评论：划线 + 红色左边框 3px；合并计数标签“×N 合并”
- 主网格实测列宽：`400px 594px 270px`
| # | 组件 | 尺寸 @坐标 (px) | Qt 控件 / 对象 | 数据绑定 | 交互 / 状态 |
|---|---|---|---|---|---|
| 1 | 页头 | `1288×34 @(224,74)` | QWidget#pageHeader | — | — |
| 2 | 答复模式 | `234×28 @(1278,77)` | SegmentedButton(3) | 全自动 / AI 草拟+人工审核(默认) / 仅人工 | 单选 |
| 3 | 评论来源条 | `1288×40 @(224,120)` | QFrame#card + Chip ×3 + 统计 QLabel | 模拟源、人工输入、TikTok 官方接口(未接入)；收到/合并/待审/已答 | — |
| 4 | 分类筛选 | `316×28 @(237,182)` | SegmentedButton(7) | 全部/尺码/价格/颜色/材质/物流/其他 | 过滤列表 |
| 5 | 评论流 CommentList | `398×769 @(225,220)` | QListView + CommentDelegate | CommentEvent：用户、合并数、优先级、分类、状态 | 选中→详情；注入划线红边 |
| 6 | 人工输入（降级路径） | `398×49 @(225,989)` | QLineEdit + QPushButton | — | 注入为人工评论 |
| 7 | 评论详情卡 | `594×866 @(636,172)` | QFrame#card | 选中评论 + SLA 倒计时 | 超时自动 EXPIRED |
| 8 | 原评论（来源授权徽标） | `568×102 @(649,222)` | QFrame | 原文、来源、相似合并说明 | — |
| 9 | 输入隔离提示 | `568×49 @(649,396)` | QLabel#note[kind=error] | 注入检测结果 | 评论只作数据字段传给模型 |
| 10 | AI 草拟答复 | `568×104 @(649,455)` | QPlainTextEdit 高 84 | AIResponse.draft | 编辑；播出前再校验 |
| 11 | 候选与模板 | `568×378 @(649,569)` | QListView | 候选答复、字段回填模板、中文提词 | 点击替换上方文本 |
| 12 | 审核操作行 | `568×30 @(649,998)` | QPushButton ×5 | — | 本地试听；复制；忽略；人工接管；Ctrl+Enter 审核通过→入队 |
| 13 | 分类分布 | `270×278 @(1242,172)` | QFrame#card + QProgressBar ×7 | 近 10 分钟分类计数 | — |
| 14 | 分级规则 | `270×576 @(1242,462)` | QFrame#card + QLabel | P2/P3 规则、去重窗口、过期、频率 | 只读（可在设置中改） |

### 06 模型与音频设置 Settings（`SettingsPage`）

![06 模型与音频设置 Settings](06_settings.png)
**布局：**
- 两等分栏（各 638）；左：模型提供者卡（内部滚动）；右：音频设备（内容高）/ 直播输出与监听（内容高）/ 离线自检（stretch）
- 设备行网格：112 | 1fr | 136 | 100 | 76；直播输出默认开启，可选设备
- 切换 TTS：先卸载 → 确认显存回落 → 再加载（后台线程，卡片显示进度）
- 主网格实测列宽：`638px 638px`
| # | 组件 | 尺寸 @坐标 (px) | Qt 控件 / 对象 | 数据绑定 | 交互 / 状态 |
|---|---|---|---|---|---|
| 1 | 离线状态 / 应用 | `477×30 @(1035,76)` | StatusChip + QPushButton ×2 | 离线可启动状态 | 重新枚举设备；应用（后台重载） |
| 2 | 模型提供者卡 | `638×918 @(224,120)` | QFrame#card | ModelConfig | — |
| 3 | 提供者标签 | `257×28 @(321,130)` | SegmentedButton → QStackedWidget | TTS/ASR/LLM·翻译/云端(可选) | 切换 |
| 4 | TTS 首选卡 ProviderCard | `612×119 @(237,178)` | QGroupBox + QRadioButton | 版本/精度/显存/许可证/来源/采样率/首包RTF(待实测)/离线 | 选中=加载；切换前先卸载并确认显存回落 |
| 5 | 轻量回退卡 | `612×80 @(237,396)` | QGroupBox | Kokoro 82M CPU | OOM/失败自动回退 |
| 6 | ASR / LLM 摘要 | `612×78 @(237,493)` | QGroupBox ×2 | FunASR+SenseVoiceSmall；Qwen3 4B Q4_K_M 线程数 | — |
| 7 | 模型记录 ModelConfig 表 | `612×186 @(237,666)` | QTableView | 用途/模型/运行时/量化/来源/许可证/SHA256 | 导出许可证说明；缺项标红 |
| 8 | 云端 API 密钥 | `612×30 @(237,862)` | QLineEdit(Password) + keyring | Windows 凭据管理器(DPAPI) | 只显示掩码 |
| 9 | 音频设备卡 AudioManager | `638×275 @(874,120)` | QFrame#card | WASAPI 枚举结果 | — |
| 10 | 设备行（直播输出） | `612×50 @(887,297)` | QComboBox + QLabel + StatusChip + QPushButton | 设备、48k·16bit·立体声、在线·输出中 | 测试音；断开→红 |
| 11 | 直播输出与监听卡 | `638×198 @(874,407)` | QFrame#card + ToggleSwitch(默认开) | 音量、监听模式、限幅器、断开策略 | 关闭→任务只在本地监听播放 |
| 12 | iPhone 回环测试 | `612×30 @(887,564)` | QPushButton + StatusChip | 最近测试结果（默认未实机验证） | 运行回环测试（需外置声卡+iPhone） |
| 13 | 离线自检/阈值 | `638×422 @(874,617)` | QFrame#card + QDoubleSpinBox | 自检项、显存/欠载/超时阈值 | 重新自检 |

---

## 5. 紧急停止状态（P0）

![紧急停止](07_estop_state.png)

**布局：**
- 触发后：横幅（高 44）叠加在中央区顶部，页面内容整体下移 44；紧急停止按钮变深红 + 3px 红描边
- 正在播放卡改为 CANCELLED 并降透明度 0.45；所有活动任务转 CANCELLED；直播输出缓冲立即清空
- 解除需二次确认（QMessageBox）；不提供单键恢复

| # | 组件 | 尺寸 @坐标 (px) | Qt 控件 / 对象 | 数据绑定 | 交互 / 状态 |
|---|---|---|---|---|---|
| 1 | 紧急停止横幅 EStopBanner | `1320×48 @(208,60)` | QFrame 叠加于中央区顶部 高 44 | 触发时间、被取消任务数 | “解除并恢复自动”需二次确认 |
| 2 | 紧急停止按钮 #estop | `150×43 @(1758,8)` | QPushButton#estop 150×44，[active=true] 深红+描边 | EStopState | F12 / Esc 连按两次(500ms) / 点击；永不隐藏 |
| 3 | 人工接管 | `97×39 @(1653,10)` | QPushButton#takeover | TakeoverState | F10：暂停 P3–P5 自动生成，P1 仍可用 |
| 4 | 正在播放卡（已取消） | `367×106 @(1541,152)` | NowPlayingCard，opacity 0.45 | 状态 CANCELLED | — |
| 5 | 状态筛选 | `391×61 @(1529,258)` | Chip 按钮组 | 活动/全部/8 种状态 | 过滤 |
| 6 | 队列（全部 CANCELLED） | `391×693 @(1529,319)` | QListView + QueueItemDelegate | SpeechTask 快照 | — |
| 7 | 直播输出状态块 | `300×59 @(774,0)` | QFrame#liveOutput | 开关/设备/电平/监听 | 触发后输出缓冲立即清空 |
| 8 | 设备指示 ×4 | `225×59 @(1389,0)` | DeviceIndicator ×4 | 麦克风/本地监听/直播输出/iPhone | — |

**行为要求：**
- 触发方式：`QPushButton#estop` 点击、**F12**、500ms 内连按两次 **Esc**（应用级 `QShortcut`，`Qt.ApplicationShortcut`）；可选系统级热键（`RegisterHotKey` + `QAbstractNativeEventFilter`），应用不在前台时也能触发。
- 执行顺序（Core 线程）：① `AudioManager.stop_all()` 立即清空直播输出和本地监听的缓冲；② 取消所有活动任务（→ CANCELLED，error_code=`ESTOP`）；③ 暂停自动讲解、定时、评论触发；④ 广播状态 → UI 显示横幅。目标：点击到静音 < 100ms（待实测）。
- 解除：横幅按钮“解除并恢复自动”→ `QMessageBox` 二次确认。没有单键恢复。

---

## 6. 组件规格

![组件表](09_components.png)

| 组件 | 规格 | 实现 |
|---|---|---|
| Button | padding 5/12，圆角 6，12.5px；primary / secondary / warn / danger / ghost / sm（2/8，12px）/ disabled（.45 + tooltip 说明原因） | `QPushButton` + 动态属性 `kind`、`size` |
| EStopButton | 150×44，15px/700，`#EF4444`；active 时 `#7F1D1D` + 3px 红描边 | `QPushButton#estop[active]` |
| TakeoverButton | 琥珀描边，两行文字 | `QPushButton#takeover[active]` |
| ToggleSwitch | 30×16，滑块 12，开 accent / 关 bg4 | 自定义 `QAbstractButton`（`paintEvent` + `QPropertyAnimation`） |
| SegmentedButton | 外框 line2 圆角 6，选中 accentBg + accent2 文字 | `QButtonGroup` + checkable `QToolButton` |
| RadioCard | 单选卡片：圆点 14 + 标题 12.5 + 说明 11.5；选中 accentBg + 边框 | `QRadioButton` + 自定义样式（或 `QFrame` + `QButtonGroup`） |
| Chip / FilterChip | 高 18–20，圆角 10–12 | `QToolButton`（checkable） |
| LiveOutputStatus | 300×60；on / off / disconnected 三态（见 09 图） | `QFrame#liveOutput[state]`，内含 `ToggleSwitch` + `LevelMeter` + 2 个 `QLabel` |
| DeviceIndicator | 26×22 圆角 5 图标底 + 10.5px 文字，宽 48；在线 / 关闭 / 断开 / 未验证 | `QToolButton` + tooltip |
| LevelMeter | 竖条 4px × 5–8，间距 2，高 14–22；绿 → 琥珀（> −6 dBFS）→ 红（削波）；30Hz 刷新 | 自绘 `QWidget` |
| PriorityBadge | 26×18，粗体等宽 11 | `QLabel#prio[prio]` |
| StateChip | 高 17，等宽 10.5/600 | `QLabel#state[state]` |
| Tag | 高 18，11px，圆角 4；tone = ok/warn/err/info | `QLabel[role=tag][tone]` |
| Note | 左侧 3px 色条（info 蓝 / warn 琥珀 / error 红），11.5px | `QLabel#note[kind]`（wordWrap） |
| QueueItem | 圆角 6，padding 7/9；行1 优先级+状态+来源+ID；行2 文本（最多两行）；行3 附加信息+去向+操作（↑ ↓ ✎ ✕，22×20）。结束态 opacity .6，无操作 | `QListView` + `QueueItemDelegate`（自绘，不要每行放 widget） |
| NowPlayingCard | 绿边渐变底，进度条高 4 | 自定义 `QWidget` |
| LatencyStage | 5 个等宽块：名称 11 / 数值 17/600 等宽 / P50·P95 10.5；最慢段琥珀边框；下方占比条高 10 | `QFrame ×5` + 自绘占比条 |
| VramBar | 高 22，分段 TTS / CUDA 上下文 / 其他 / 空闲 | 自绘 `QWidget` |
| Sparkline | 120×38 折线 + 15% 填充 | 自绘 `QPainterPath`（比 QtCharts 开销低） |
| AudioRouteDiagram | 节点网格 170 \| 34 \| 1fr \| 34 \| 230；节点最小高 52；在线绿框 / 断开红框 / 未验证琥珀虚框 | `QGraphicsView` 或自绘 |
| LockedField | 标签后加 🔒（琥珀 11px）；边框 warn；校验失败边框 danger + 下方 11px 红字 | `QLineEdit[locked][invalid]` |
| TranscriptTable | 行网格 54 \| 1fr \| 1fr \| 1fr；富文本：不确定=黄底虚线，已校验=绿底 | `QTableView` + `RichTextDelegate`；第 3 列 `QTextEdit` 编辑器 |
| EmptyState | 虚线框 + 说明 + 主操作 | `QFrame#emptyState` |
| Toast | 底部居中，2.2s 自动消失 | 叠加 `QWidget` + `QPropertyAnimation` |

---

## 7. QSS 主题（`theme.qss`，完整内容）

```css
{qss}```

加载方式：`app.setStyleSheet(Path("theme.qss").read_text(encoding="utf-8"))`。动态属性改变后要调用 `w.style().unpolish(w); w.style().polish(w)`。

---

## 8. 控件树与对象名

```
QMainWindow#mainWindow  (minimumSize 1600×900)
├─ QToolBar#topStatus (Top, movable=False, floatable=False, 高 60)
│   ├─ QLabel#brand(208) ─ QFrame#session(196) ─ QFrame#modes(164) ─ QFrame#currentProduct(206)
│   ├─ QFrame#liveOutput(300) [ToggleSwitch#liveSwitch, LevelMeter#liveMeter, QLabel#liveTitle/#liveDevice/#liveMonitor]
│   ├─ QFrame#resources [ResourceMini#gpu/#cpu/#ram/#e2e]  ─ QFrame#devices [DeviceIndicator#devMic/#devMonitor/#devLive/#deviceIphone]
│   ├─ (stretch) ─ QPushButton#takeover ─ QPushButton#estop
├─ centralWidget: QWidget#central → QHBoxLayout(margins 0, spacing 0)
│   ├─ QListWidget#nav (208；折叠时 56)
│   └─ QStackedWidget#pages
│       ├─ ConsolePage   (#pageConsole)    ├─ ProductPage (#pageProducts)
│       ├─ ScriptPage    (#pageScripts)    ├─ TranslatePage (#pageTranslate)
│       ├─ CommentPage   (#pageComments)   └─ SettingsPage (#pageSettings)
│       每页：QWidget#pageRoot → QVBoxLayout(16,14,16,14; spacing 12) → PageHeader(34) + 内容
├─ QDockWidget#schedulerDock (Right, 392) → QueuePanel
│   ├─ QueueHeader(82) ─ NowPlayingCard(106) ─ StateFilter(61)
│   ├─ QListView#queueList (model: QueueModel(QAbstractListModel) + QSortFilterProxyModel)
│   └─ QueueFooter(40)
├─ QStatusBar#statusBar (28)
├─ QFrame#estopBanner（叠加在 #pages 顶部，高 44，默认隐藏，raise_()）
├─ LiveOutputDialog(QDialog)、Toast（叠加）
```

**模型 / 视图**：队列、评论、话术、变体、翻译记录、模型记录一律用 `QAbstractTableModel` / `QAbstractListModel` + delegate；不要在 `QListWidget` 里塞大量自定义 widget（1000 条模拟评论压测时会卡）。信号只在页面构造时 `connect` 一次，切换模式只改模型状态。

---

## 9. 线程规则（硬性要求）

1. **UI 主线程只负责渲染和发出用户意图**。ASR、翻译、LLM、TTS 推理，模型加载，文件/CSV 导入，数据库批量写入，设备枚举和测试音，一律不在主线程执行。
2. 线程划分：
   - `Core` 线程：`QThread` + asyncio 事件循环（qasync 或自管 loop），运行唯一 Scheduler（优先级队列 + 状态机 + 取消/超时 + 过期清理）、输出路由（直播输出 / 本地监听）、AuditLog/SQLite 单写队列。
   - 推理工作者：ASRWorker（CPU）、LLMWorker（CPU，llama.cpp 线程数可配，给 UI/ASR 预留核心）、TTSWorker（GPU 独占），各自是独立 `QThread` 或子进程。
   - AudioManager：sounddevice / WASAPI 回调线程，只拷贝 PCM。
   - ResourceSampler：`QTimer` 1s 触发 → 后台采样 psutil / NVML → Signal。
3. 跨线程只用 `Signal/Slot`（`Qt.QueuedConnection`），并传不可变快照（dataclass）；工作线程**禁止**直接操作控件。
4. 每个任务都是 `asyncio.Task` + `asyncio.timeout()`，可取消；异常写入结构化 error_code，并在 UI 可见（Toast / 状态栏 / 队列 FAILED）。
5. 示例桥接：

```python
class SchedulerBridge(QObject):
    queueChanged = Signal(list); nowPlaying = Signal(object)
    liveOutputChanged = Signal(object); estopTriggered = Signal()
    def __init__(self, loop, scheduler): super().__init__(); self._loop, self._s = loop, scheduler
    @Slot()
    def emergency_stop(self):   # UI 线程调用，立即返回
        asyncio.run_coroutine_threadsafe(self._s.emergency_stop(), self._loop)
    @Slot(str, int)
    def enqueue(self, text, prio):
        asyncio.run_coroutine_threadsafe(self._s.submit(text=text, priority=prio, source="manual"), self._loop)
```

---

## 10. 快捷键

| 快捷键 | 作用 | 范围 |
|---|---|---|
| **F12** | P0 紧急停止 | 全局（应用级；可选系统级热键） |
| **Esc ×2（500ms 内）** | 紧急停止（备用）；单击 Esc 只关闭弹窗 | 应用内 |
| **F10** | 人工接管：暂停 P3–P5 自动生成，P1 仍可用 | 全局 |
| F9 | 中断当前播报 | 全局 |
| F8 | 实时翻译 暂停 / 恢复 | 全局 |
| Ctrl+1…6 | 切换页面 | 应用内 |
| Alt+← / → | 上一个 / 下一个商品 | 应用内 |
| Ctrl+P | 本地试听（只在耳机播放，不进直播） | 编辑区 |
| Ctrl+Enter | 加入队列 / 评论审核通过入队 | 编辑区 |
| Ctrl+Shift+Enter | 立即播报 P1（→ 直播输出 + 监听） | 编辑区 |
| Ctrl+S | 保存 | 编辑区 |

---

## 11. 状态 / 空状态 / 错误状态

| 场景 | 表现 |
|---|---|
| 直播输出开启（默认） | 顶栏绿色块 + 电平；直播输出指示绿；队列去向“→ 直播输出” |
| 直播输出关闭 | 顶栏块变灰、电平隐藏；任务只在本地监听播放，去向“→ 本地监听”；Toast 提示 |
| 直播输出设备断开 | 顶栏块红色“已断开”；指示灯红色闪烁；**暂停出队**；Toast + AuditLog；设置页该行“已断开 · 重新枚举”；**不回落**到系统默认设备 |
| iPhone 未实机验证 | iPhone 指示“?”琥珀；链路图琥珀虚线节点；有回环测试记录后才转绿 |
| 紧急停止 | 见 §5 |
| CUDA OOM / TTS 失败 | 任务 FAILED（显示 error_code）；重试 1 次后回退 Kokoro（CPU）；顶栏显存变红 |
| 模型加载中 | Provider 卡显示进度和“后台加载中”；相关按钮禁用，tooltip 说明原因；UI 不冻结 |
| 翻译积压 | 积压数 > 0 琥珀、> 3 红，自动合并或丢弃最旧一段并提示 |
| 事实校验失败 | 禁止入队 / 播放；红标签“与字段不符”，列出差异 |
| 评论注入 | 划线 + 3px 红左边框 + “注入已拦截”；不进入模型；写 AuditLog |
| 空队列 / 无商品 / 无话术 / 无评论 | `EmptyState` 虚线框 + 主操作（导入 CSV / 从模板新建 / 开启模拟评论源） |
| 文本超长 | 列表两行截断 + tooltip；表格单行省略 |

---

## 12. 验收清单（请 Claude 实现后逐项自检）

**布局**
- [ ] 1920×1080 下五区尺寸：顶栏 60、导航 208、队列 392、状态栏 28、中央 1320×992；与 `00_global_layout.png` 一致（±2px）。
- [ ] 六个页面的列宽和行高与 01–06 图的尺寸线一致（固定列固定，stretch 列伸缩）。
- [ ] 最小窗口 1600×900 可用：内容不重叠、不截断按钮；导航可折叠；队列 Dock 可收起和呼出。
- [ ] 任何尺寸、任何页面、任何弹窗下紧急停止按钮都可见、可点击。
- [ ] 导航文字不换行；顶栏各块不换行。

**视觉**
- [ ] 加载 `theme.qss`；颜色、字号、圆角、间距与 08 一致；数字一律等宽字体。
- [ ] P0–P5 和 8 种状态的颜色与 08/09 一致，并带文字。
- [ ] “示例数据”标签保留；未实测的数值显示“待实测”，不伪造。

**功能与交互**
- [ ] **不存在任何 PlatformOutputPolicy / 平台合规门控 UI 或逻辑**；直播输出默认开启，可在顶栏和设置页开关。
- [ ] 本地监听、直播输出、麦克风三路设备分开选择、分开显示状态；“本地试听”不进直播。
- [ ] iPhone 状态默认“未实机验证”；“电脑声卡播放成功 ≠ 手机已收到声音”提示在控制台和设置页常驻。
- [ ] 所有可播放内容只经过唯一 Scheduler；队列显示优先级、状态、去向，可取消、上移、下移、编辑。
- [ ] F12 / Esc×2 / 按钮都能触发紧急停止：播放中断，活动任务全部变为 CANCELLED，横幅出现；解除需二次确认。
- [ ] F10 人工接管、F9 中断、F8 翻译暂停、Ctrl+1…6、Alt+←/→ 都有效。
- [ ] 🔒 字段只能人工编辑；AI 输出与字段不一致时拒绝入队并提示。
- [ ] 实时翻译三列文本（原始识别 / 机器翻译 / 最终播报）分别显示、分别保存。
- [ ] 评论中心：分类筛选、合并计数、注入拦截样式、人工输入降级路径、审核后入队。
- [ ] 每个按钮都接入真实业务逻辑（没有空槽函数、没有只弹窗的按钮）。

**线程与稳定性**
- [ ] 推理、模型加载、导入、设备测试时 UI 保持流畅（主线程单帧 < 50ms，可用状态栏帧耗时验证）。
- [ ] 工作线程不直接操作控件；跨线程只走 Signal/Slot。
- [ ] 设备断开有可见告警并暂停出队，不回落到错误设备。
- [ ] 1000 条模拟评论 / 100 条队列任务下，列表滚动流畅（Model/View + delegate）。
