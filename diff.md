# OperitRp 实现详情 (diff.md)

基于原版 OperitRpRead，对照 AGENTS.md 需求，逐项说明改动。

---

## 1. 基础配置

### 1.1 包名与应用名
- **需求**：包名 `com.ai.assistance.operitrp`，应用名 `OperitRp`，与原版共存
- **实现**：
  - `app/build.gradle.kts` — `applicationId` 改为 `com.ai.assistance.operitrp`，`versionName` 改为 `1.11.0RP0.07`
  - `app/src/main/res/values/strings.xml` — `app_name` 改为 `OperitRp`，`software_identity_option_operit` 改为 `OperAI_RP`

### 1.2 签名
- **需求**：独立签名密钥，与原版区分
- **实现**：
  - `key/operitrp-release.jks` — RSA 2048，有效期 10000 天，别名 `operitrp`，密码 `operitrp123`
  - `local.properties` — 配置 `RELEASE_STORE_FILE` 等四项
  - `app/build.gradle.kts` — `storeFile` 改为 `rootProject.file(releaseKeystorePath)`（原版用 `file()` 相对于 app 目录）
  - `.gitignore` — 添加 `key/` 排除

### 1.3 侧栏导航
- **需求**：标题改为 `OperAI_RP`，角色扮演强化作为侧栏第五项
- **实现**：
  - `ui/common/NavItem.kt` — 新增 `object RpSettings`，图标 `Icons.Default.Casino`，路由 `rp_settings`，标题 `R.string.rp_settings`
  - `ui/main/OperitApp.kt` — `navItems` 列表顺序调整：AiChat → AssistantConfig → MemoryBase → Toolbox → **RpSettings** → Packages → ...
  - `ui/main/screens/ScreenRouteRegistry.kt` — 添加 `hostEntryDefinition(entryId = "main.rp_settings", ...)`，挂在 `MAIN_SIDEBAR_TOOLS`，order=25
  - `ui/main/screens/OperitScreens.kt` — `data object RpSettings` 使用 `NavItem.RpSettings`

---

## 2. 数据层（新增文件）

### 2.1 RpValueEntry
- **文件**：`data/model/RpValueEntry.kt`
- **内容**：`data class RpValueEntry`（id/name/initialValue/currentValue/minValue/maxValue/randomRound/enableRandomOnStart/isEnabled）+ `RpPanelSnapshot`（values Map + round）
- **关键方法**：`shouldTriggerRandom(currentRound)` — `isEnabled && randomRound > 0 && currentRound > 0 && currentRound % randomRound == 0`

### 2.2 RpValueManager
- **文件**：`data/preferences/RpValueManager.kt`
- **存储**：DataStore，key 前缀 `rp_panel_entries_`、`rp_snapshots_`、`rp_template_entries`、`rp_random_opening_prompt`
- **模板 CRUD**：`getTemplateEntries`、`saveTemplateEntries`、`addTemplateEntry`、`updateTemplateEntry`、`removeTemplateEntry`
- **面板 CRUD**：`getPanelEntries(chatId)`、`savePanelEntries`、`addPanelEntry`、`setPanelValue`、`adjustPanelValue`、`randomPanelValue`、`removePanelEntry`
- **快照**：`saveSnapshot(chatId, messageId)` — 保存当前面板 values Map；`restoreSnapshot` — 恢复 values；`deleteSnapshot` — 删除
- **随机**：`processRandomRound(chatId, currentRound)` — 遍历条目，对满足 `shouldTriggerRandom` 的执行随机；`generateRandomReport` — 生成文本报告
- **预填充**：`initializeDefaultsIfNeeded()` — 首次启动写入默认模板（尾巴 0-3，开局随机=true）和默认开局提示词

### 2.3 RpPreferences
- **文件**：`data/preferences/RpPreferences.kt`
- **内容**：`showHiddenTextFlow` / `saveShowHiddenText()` — 控制隐藏内容显示开关

### 2.4 RpRandomOpeningManager
- **文件**：`data/preferences/RpRandomOpeningManager.kt`
- **内容**：`executeRandomOpening(chatId)` — 随机所有 `enableRandomOnStart=true` 的条目，置 false，返回报告文本

---

## 3. AI 工具层

### 3.1 RpToolRegistration
- **文件**：`core/tools/RpToolRegistration.kt`
- **注册**：`registerRpTools(handler, context)` 在 `ToolRegistration.kt` 的 `registerAllTools` 末尾调用

#### 面板工具（7个）

| 工具名 | 参数 | 逻辑 |
|--------|------|------|
| `rp_set` | name, value | 调 `rpValueManager.setPanelValue(chatId, name, value)`，超出范围返回错误 |
| `rp_adjust` | name, delta | 调 `rpValueManager.adjustPanelValue(chatId, name, delta)`，超出范围返回错误 |
| `rp_random` | name | 调 `rpValueManager.randomPanelValue(chatId, name)`，在 [min,max] 内随机 |
| `rp_add` | name, initialValue, minValue, maxValue | 调 `rpValueManager.addPanelEntry(chatId, entry)`，name 重复则拒绝 |
| `rp_remove` | name | 调 `rpValueManager.removePanelEntry(chatId, name)`，不存在则静默成功 |
| `rp_rename` | old_name, new_name | 遍历面板条目，找到 old_name 后 copy(name=new_name)，name 重复则拒绝 |
| `rp_update_entry` | name, min?, max?, random_round?, enabled?, random_on_start? | 遍历找到条目，按传入参数逐项 copy 更新元数据，min>max 自动修正 |

#### 模板工具（6个）

| 工具名 | 参数 | 逻辑 |
|--------|------|------|
| `rp_template_read` | 无 | 返回模板条目 JSON |
| `rp_template_set` | name, value | 调 `rpValueManager.updateTemplateEntry(name, value)` |
| `rp_template_add` | name, initialValue, minValue, maxValue | 调 `rpValueManager.addTemplateEntry(entry)` |
| `rp_template_remove` | name | 调 `rpValueManager.removeTemplateEntry(name)` |
| `rp_template_rename` | old_name, new_name | 同面板 rp_rename 逻辑 |
| `rp_template_update_entry` | name, min?, max?, random_round?, enabled?, random_on_start? | 同面板 rp_update_entry 逻辑 |

#### 工具共用逻辑
- `getChatId(tool)` — 从 `__operit_package_chat_id` 参数获取 chatId
- 返回格式：`{ "success": true, "entry": { "name": "金币", "currentValue": 5, "minValue": 0, "maxValue": 100 } }`

### 3.2 ToolRegistration
- **文件**：`core/tools/ToolRegistration.kt`
- **改动**：在 `registerAllTools` 末尾添加 `registerRpTools(handler, context)`

---

## 4. 提示词层

### 4.1 RpIdentityPromptBuilder
- **文件**：`core/config/RpIdentityPromptBuilder.kt`
- **buildIdentityPrompt(activeCard, panelValues)**：
  - 输出 `## RP数值面板` 标题
  - 遍历 enabled 条目，生成 JSON：`enableRandomOnStart=true` 时 value 显示 `"-"`，否则显示 currentValue
  - 有 `-` 标记时输出提示："看到值为\"-\"的条目，表示该数值需要在首次使用前通过rp_random工具进行随机"
  - 输出数值操作规则（暗骰约束、拒绝策略等）
- **buildSystemRpPrefix()**：返回 `"[Narrator]"`
- **buildSystemRpInstructions()**：system_rp 身份说明
- **buildRandomOpeningInstructions()**：`-` 标记说明

### 4.2 ConversationService
- **文件**：`api/chat/enhance/ConversationService.kt`
- **改动**：在 `prepareConversationHistory` 构建 `finalSystemPrompt` 时：
  - 获取 `RpValueManager.getPanelEntries(chatId)`
  - 调用 `RpIdentityPromptBuilder.buildIdentityPrompt(activeCard, panelValues)`
  - 将结果追加到系统提示词（在 waifu 规则之后、用户偏好之前）

### 4.3 AIMessageManager
- **文件**：`core/chat/AIMessageManager.kt`
- **改动**：`getMemoryFromMessages` 中：
  - filter 条件添加 `it.sender == "system"`
  - 新增 `message.sender == "user" && message.roleName == "system_rp"` 分支：转为 `PromptTurn(kind=USER, content="[Narrator] ${message.content}")`
  - 用 `when` 替代原来的 `when(message.sender)` 以支持多条件

---

## 5. 消息处理层

### 5.1 MessageProcessingDelegate
- **文件**：`services/core/MessageProcessingDelegate.kt`
- **新增参数**：`senderIdentityOverride: String = "user"`
- **改动**：`sendUserMessage` 中：
  - 发送前：`senderIdentityOverride == "user"` 时保存快照 + 检查随机触发（轮次 = `getFullChatHistory.count { sender=="user" && roleName!="system_rp" }`）
  - 有随机报告时附加到消息内容
  - 创建 `ChatMessage` 时：sender 固定为 `"user"`，roleName 为 `"system_rp"` 或默认用户角色名
  - system_rp 消息 displayMode 设为 `HIDDEN_PLACEHOLDER`

### 5.2 ChatHistoryDelegate
- **文件**：`services/core/ChatHistoryDelegate.kt`
- **改动**：`deleteMessage` 中：
  - 删除 user 消息前调 `rpValueManager.restoreSnapshot` + `deleteSnapshot`
  - 轮次不再单独存储（由 user 消息数量决定）

### 5.3 MessageCoordinationDelegate
- **文件**：`services/core/MessageCoordinationDelegate.kt`
- **改动**：
  - `sendUserMessage` 新增 `senderIdentityOverride` 参数
  - `sendMessageInternal` 新增 `senderIdentityOverride` 参数
  - 透传到 `messageProcessingDelegate.sendUserMessage`

### 5.4 ChatViewModel
- **文件**：`ui/features/chat/viewmodel/ChatViewModel.kt`
- **新增**：
  - `_senderIdentity: MutableStateFlow<String>("user")` + `setSenderIdentity(identity)`
  - `sendUserMessage` / `sendTextMessage` 使用 `_senderIdentity.value` 作为 `senderIdentityOverride`
  - `executeRandomOpening()` — 随机标记条目 → 构建注入消息（报告+提示词+第0轮指令）→ 以 system_rp 身份发送

---

## 6. UI 层

### 6.1 P 面板（RpPanelMenu）
- **文件**：`ui/features/chat/components/style/input/common/RpPanelMenu.kt`
- **RpPanelButton** 参数：`rpValueManager, rpPreferences, chatId, currentRound`
- **布局**：
  - 第一行：添加条目按钮 | 调试开关（眼睛图标）| 轮次显示 `[轮次:X]`
  - 表头：启用 | 名称 | 值 | 下限 | 上限 | 随机轮数 | 骰子 | 删除
  - 数据行：所有可交互元素带 1dp 描边
- **交互**：
  - 点击任意值框 → `RpEditDialog` 弹窗（标题显示字段类型，输入框+确认/取消）
  - 点击名称 → `RpRenameDialog` 弹窗
  - 点击启用开关 → 切换 isEnabled
  - `enableRandomOnStart=true` 的条目当前值显示为 `-`（紫色）
- **EditField 枚举**：NAME, CURRENT, MIN, MAX, RANDOM_ROUND

### 6.2 设置页面（RpSettingsScreen）
- **文件**：`ui/features/settings/screens/RpSettingsScreen.kt`
- **布局**：完整展开所有字段（无省略模式）
  - 每个条目：Card 包含启用开关、名称、初始值、下限/上限、随机轮数、开局随机、删除、保存
  - 底部：添加条目区域 + 开局提示词输入框
- **SettingsScreen** 添加入口：提示词配置 section 下，waifu 设置之后

### 6.3 输入栏集成
- **ClassicChatInputSection**：在附件按钮前添加 u/s 切换 + P 按钮
- **AgentChatInputSection**：两个渲染路径（透明/非透明）都添加了 u/s 切换 + P 按钮
- **参数传递**：`senderIdentity, onSenderIdentityChange, currentChatId, currentRound`

### 6.4 随机开局按钮
- **ChatArea**：`showRandomOpeningButton && chatHistory.isEmpty() && onRandomOpening != null` 时显示
- **ChatScreenContent**：两个 ChatArea 调用都传递了 `showRandomOpeningButton` 和 `onRandomOpening`
- **AIChatScreen**：`showRandomOpeningButton = chatHistory.isEmpty()`，`onRandomOpening = { actualViewModel.executeRandomOpening() }`

### 6.5 轮次计算
- **规则**：直接数 user 消息数量（排除 `roleName == "system_rp"`），不存计数器
- **位置**：`AIChatScreen` 中 `actualViewModel.chatHistory.value.count { msg -> msg.sender == "user" && msg.roleName != "system_rp" }`
- **传递**：作为 `currentRound` 参数传入 ClassicChatInputSection / AgentChatInputSection → RpPanelButton

---

## 7. 随机标记系统

### 7.1 "-" 标记
- **规则**：`enableRandomOnStart=true` 的条目在系统提示词中显示 `"value": "-"`
- **AI行为**：看到 `-` → 调 `rp_random(name)` → 值变为具体数字，`enableRandomOnStart` 自动置 false
- **P面板**：当前值显示 `-`（紫色），区别于正常数字（蓝色）

### 7.2 随机开局流程
1. 用户点击随机开局按钮
2. `executeRandomOpening()` 随机所有 `enableRandomOnStart=true` 的条目
3. 构建 system_rp 消息：确认数值报告 → 开局提示词 → "现在开始第0轮开局演绎..."
4. 以 `senderIdentityOverride="system_rp"` 发送
5. AI 收到后独自演绎，等 user 发言后才回应

### 7.3 纯净模式
- 不点随机开局按钮 → 不注入开局提示词
- 面板数值和工具照常可用（AI 可用 rp_set 等做别的事）
- 系统提示词始终注入面板 JSON + 工具规则

---

## 8. 预填充示例

- **模板条目**：`尾巴`（初始值 0，范围 0-3，随机轮数 0，开局随机=true）
- **开局提示词**：`"助手每次进行重置，都会以最好的状态面对user...看到值为-的条目，需要在首次涉及前调用rp_random执行随机..."`
- **首次启动**：`OperitApp` 的 `LaunchedEffect(Unit)` 调 `RpValueManager.initializeDefaultsIfNeeded()`

---

## 9. 修复记录

| 版本 | 问题 | 原因 | 修复 |
|------|------|------|------|
| v0.02 | 按钮不显示 | 只在 Agent 透明模式分支加了按钮 | 非透明模式分支也加按钮 |
| v0.03 | P按钮不显示 | `currentChatId != null` 条件导致新对话时不渲染 | 去掉 null 检查，用 `?: ""` |
| v0.04 | 随机开局按钮无响应 | `executeRandomOpening` 返回 null 时静默跳过 | 改为始终注入提示词 |
| v0.05 | 400 错误 | system_rp 用 `sender="system"` 导致多个 SYSTEM 消息 | 改为 `sender="user"` + `roleName="system_rp"` |
| v0.05 | system_rp 不显示 | `sender="system"` 被过滤 | 改为 USER 类型 + `[Narrator]` 前缀 |
| v0.06 | 轮次计数错误 | system_rp 也触发轮次+1 | 改为直接数 user 消息数量（排除 system_rp） |
| v0.07 | 编译错误 | `chatHistory.count { it.roleName }` 的 `it` 引用歧义 | 显式类型声明 `msg: ChatMessage` |
| v0.07 | P面板R列文字 | 显示 "R" 不够清晰 | 改为显示随机轮数文字 |
