# OperitRp Design Document

## 项目背景

- 源项目: https://github.com/AAswordman/Operit
- Fork: https://github.com/knGear/OperitLink
- 最初代号 Link，计划做单用户多设备功能
- 后因 Operit2 已有相关计划而搁置
- 转向角色扮演强化，现代号 OperitRp

## 迁移计划

- 最终目标：迁移至 operit2Rp
- 当前状态：主线完成度不高，仅有 CLI
- 等待 operit2 稳定后再动手迁移
- 现阶段需记录好改进内容

## 基础要求

- 稳定的 APK 签名，方便覆盖安装
- 包名和应用名：OperitRp（品牌名后带 Rp），与原版共存并做好区分
- 版本号基于主线 1.11.0
  - 大版本：1.11.0RP1、1.11.0RP2...
  - Bug 修复：1.11.0RP1.1、1.11.0RP1.2...
  - 当前输出：1.11.0RP0.x
- 大版本完成无 bug 才 push
- 输出 APK 到 ./out/，命名格式：OperitRp1.11.0RP0.1.apk，每次更新末尾数字递增

## 概览

Rp 由三部分组成：
1. 数值系统
2. 随机开局
3. system_RP 身份

## 数据模型

### RpValueEntry（数值条目）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | String | UUID，不可变 |
| name | String | 条目名称，AI 工具按此操作 |
| initialValue | Int | 初始值，用于重置 |
| currentValue | Int | 当前值 |
| minValue | Int | 下限 |
| maxValue | Int | 上限 |
| randomRound | Int | 随机轮数，0 = 禁用 |
| enableRandomOnStart | Boolean | 开局随机标记，见"开局随机"章节 |
| isEnabled | Boolean | 启用开关 |

### 两层数值体系

- **模板数值**：存在设置中，用于初始化 P 面板
- **P 面板数值**：存在当前对话中，随对话实时变动
- **同步规则**：新建对话时模板 -> 面板一次性初始化；此后只要对话中存在任何身份（user/AI/system_RP 等）的输出，两者完全断开，互不影响
- 面板工具和模板工具按需使用，两者没有特别的限制

## AI 工具接口

### 面板工具

AI 通过 system prompt 的 JSON 注入读取当前面板数值状态（无需单独读取工具）。
写操作通过以下工具完成，所有工具操作的是 P 面板数值。

| 工具名 | 用途 | 参数 |
|--------|------|------|
| rp_set | 设定当前值 | name: String, value: Int |
| rp_adjust | 增减当前值 | name: String, delta: Int |
| rp_random | 触发随机 | name: String |
| rp_add | 新建条目 | name: String, initialValue: Int, minValue: Int, maxValue: Int |
| rp_remove | 删除条目 | name: String |

#### 校验规则

- **拒绝策略**：值超出 [min, max] 范围时拒绝写入，返回错误信息，由 AI 自行演绎（如"金币不足"）
- rp_set / rp_adjust：超出范围则拒绝，返回 `{ "success": false, "error": "值超出范围 [min, max]" }`
- rp_random：在 [min, max] 范围内随机，结果写入 currentValue
- rp_add：name 重复则拒绝；initialValue 直接作为面板 currentValue
- rp_remove：name 不存在则静默成功
- 所有范围校验在服务端强制，AI 不负责校验

#### 返回值格式

成功：`{ "success": true, "entry": { "name": "金币", "currentValue": 5, "minValue": 0, "maxValue": 100 } }`
失败：`{ "success": false, "error": "条目'金币'不存在" }`

#### 校验示例

钱袋：下限 0，当前 2。AI 执行 rp_adjust(name="金币", delta=-3) ->
拒绝，返回 { "success": false, "error": "金币不足，当前 2，无法减少 3" }
AI 看到拒绝后自行演绎"金币不足"的场景。
randomRound 下限为 0，AI 尝试减到负数时同样拒绝。

### 模板工具

模板工具操作设置中的模板数值，按需使用，模板数据不默认注入 prompt。
用于调试场景，如用户说"帮我加个狐狸尾巴"时 AI 调用 rp_template_add。

| 工具名 | 用途 | 参数 |
|--------|------|------|
| rp_template_read | 读取当前模板配置 | 无 |
| rp_template_set | 设定模板条目值 | name: String, value: Int |
| rp_template_add | 新建模板条目 | name: String, initialValue: Int, minValue: Int, maxValue: Int |
| rp_template_remove | 删除模板条目 | name: String |

校验规则与面板工具一致（拒绝策略、范围校验）。写入非法值时拒绝。

## 设置

入口：设置 -> 提示词配置 -> 角色扮演强化

包含：数值、开场提示词（无二级菜单，不需收纳）

### 数值

由数值条目和添加条目组成（添加条目始终跟随在已添加条目下方）

数值条目组成（设置页完整展开所有字段）：
1. 启用开关（勾选切换）
2. 数值名（字符）
3. 初始值（整数）
4. 上限/下限（数值）
5. 随机轮数（>=0 整数）
6. 开局随机（勾选切换）
7. 删除此数值条目

### 开局提示词

普通字符输入框

### 输入规则

- 数值输入框：焦点在时允许非法内容（负数、空值）
- 失去焦点时填充 0（如删除仅有的"1"后关闭键盘，填充 0）
- 随机轮数最小为 0，-1 为非法值，有则强制为 0
- 添加条目默认值：0（或下限）
- 修改上下限导致数值非法时，自动修正非法部分

## P 菜单 (UI)

按钮字母 P，位于对话主页下方、输入框底部模型设置左边

展开/收起：
- 点击 P 按钮展开
- 展开后点击 P 按钮或旁边非 P 菜单区域收起

展开内容：
- 顶部：简化条目切换按钮、加号增加条目按钮、[当前轮数:X]（右侧，只读）、显示隐藏内容按钮（调试用）
- 每个条目支持切换完整/省略两种视图：
  - **省略模式**：数值名、当前数值、随机按钮（3项，紧凑显示）
  - **完整模式**：启用开关、数值名、当前数值、上限、下限、随机轮数、随机按钮、删除条目（8项）
- 点击条目可在完整/省略之间切换

### 显示隐藏内容按钮

P面板顶部增加一个调试按钮，点击后切换显示/隐藏所有隐性内容：
- system 消息（原有，AI 不回复）
- system_RP 消息（新增，引导叙事）
- 默认隐藏，点击按钮后显示
- 再次点击恢复隐藏

### 对话轮次

- 内置只读数值，默认 0
- user 说一句 +1，删除一句 -1
- 示例：user 说 1 句 ai 回 1 句 = 1；user 说 2 句 ai 回 1 句 = 2；说 3 轮全删 = 0
- 计数时机：user 发送消息时先 +1，再进行后续步骤（随机检查、prompt 构造等）

### 随机轮数

- 当前 user 轮次 / 设置的随机轮次 = 整除时触发，0 时禁用
- 示例：随机轮数=3，第 3/6/9... 轮触发，在设置上下限内随机
- 随机后把已随机和新数值报告给 AI
- **不会在第 0 轮触发**，第 0 轮的随机由"开局随机"独立处理
- 删除消息导致轮次回退后再次到达整除点，会再次触发（预期行为，不做去重）
- randomRound 本身是可被 AI 工具修改的数值（如设计"雨越来越频繁"场景时，AI 可在随机触发后用 rp_adjust 将 randomRound 减小）

### shouldTriggerRandom 逻辑

```
触发条件：isEnabled && randomRound > 0 && currentRound > 0 && currentRound % randomRound == 0
```

- currentRound > 0：防止第 0 轮与"开局随机"冲突
- 第 0 轮的随机由 enableRandomOnStart 标记驱动（见下文"开局随机"）

### processRandomRound 职责

- 只修改 P 面板数值，不修改模板
- 触发后返回被随机的条目列表，供系统向 AI 报告

### 模板数值 vs P 面板数值

- 设置内：模板数值（用于传递给 P 面板）
- P 面板：随对话演绎实时变动
- 新建对话时模板 -> 面板一次性初始化
- 对话中存在任何身份的输出后，两者完全断开，互相不影响
- AI 面板工具只操作 P 面板数值
- AI 模板工具操作模板数值（调试场景使用）

### 开局随机（AI 驱动）

不再由系统在初始化时自动随机，改为通过 prompt 驱动 AI 调用工具完成：

- 面板初始化时，enableRandomOnStart=true 的条目 currentValue 保持为 initialValue（不随机）
- prompt 注入时，enableRandomOnStart=true 的条目显示为 "金币": "r" 而非数字
- system prompt 中说明：看到 "r" 标记的数值，AI 在首次使用前应先调用 rp_random
- AI 调用 rp_random 后，enableRandomOnStart 自动置为 false，后续不再触发
- 用户在 P 面板手动填入数值 -> enableRandomOnStart 置为 false -> AI 不再随机

"r" 只是 prompt 里的显示约定，数据层 currentValue 仍为 Int。

### 随机开局按钮

点击后对所有 enableRandomOnStart=true 的条目执行随机（等效于 AI 调 rp_random），置 false，然后注入 system_RP 消息。

**重要限制：随机开局按钮仅在新建对话时出现，且仅当对话完全空（任何身份都没有内容）时可见。已有对话记录（即使删除所有消息变空）也不显示此按钮，以防止不可预期的故障。**

判断逻辑：
- 对话必须是全新创建的（从未有任何消息）
- 已有对话记录但删除所有消息变空 -> 不显示
- 有总结消息但无其他消息 -> 不显示

### 数值初始化

- 新建对话时：模板数值传递给 P 面板，enableRandomOnStart=true 的条目标记为 "r"
- 对话完全空（任何身份都没有输出）时修改模板：同步到面板

## system_RP

### 背景

- system 是 operit 的重要组成部分，已有广泛使用和约束规则
- 新开 Rp 以区分并不受现有规则影响
- 未来可能扩展更多身份（如旁白 narrator）

### 行为定义

- system_RP 不参与演绎，不是角色
- 引导叙事走向和数值变化（如"风突然大了"、"AI突然摔跤了"）
- AI 不回复 system_RP，而是顺着要求自然演绎

### 消息传递方式

通过消息元数据 identityType 字段传递身份信息：

- 用户点 u 发送 -> role=user, identityType=user
- 用户点 s 发送 -> role=system, identityType=system_rp

构造发送给模型的消息列表时：
- identityType=user -> 正常 user message
- identityType=system_rp -> 作为 system message 注入，RpIdentityPromptBuilder 在其前拼接身份提示

### system_RP 提示词

在 system prompt 中说明 system_rp 身份的含义和行为规则。当消息列表中遇到 system_rp 消息时，在其前面拼接身份提示，引导 AI 顺着演绎。

### 隐藏文本调试开关

模型设置中增加"显示隐藏文本"开关，默认关闭。

开启后显示所有隐性内容：
- system 消息（原有，AI 不回复）
- system_RP 消息（新增，引导叙事）

关闭时这些消息对用户不可见，保持沉浸感。

**P面板顶部也有显示隐藏内容按钮，功能与设置中的开关同步。**

### UI

长按对话 -> 编辑对话：新增三选一按钮 a/u/s（ai/user/system_Rp），针对已产生的对话
对话主页 P 按钮左边：u/s 切换按钮
  - 点击切换 u，再点切换 s
  - 发送时携带参数申明当前输出内容身份

### 随机开局 UI

对话列表顶部增加长按钮
点击：把开局提示词以 system_Rp 身份，结合随机后的数值，注入为一条 system_RP 消息
仅当对话是全新创建且完全空时出现（已有对话记录删除空也不显示）
注入的消息默认不显示，打开隐藏文本调试开关后可见

## 数值快照与回滚

每条 user 消息发送前，记录当前 P 面板的快照（所有条目的 currentValue Map + 当前轮次）。

### 存储结构

```
message_id -> RpPanelSnapshot(
    values: Map<String, Int>,   // name -> currentValue
    round: Int                   // 当时的轮次
)
```

### 回滚规则

- 发送 user 消息 -> 保存快照 -> 轮次 +1 -> 检查随机 -> AI 响应
- 撤回/删除 user 消息 -> 恢复到该消息前的快照 -> 轮次回退
- 快照只存值，不管条目增删。回滚后由 AI 工具创建的条目仍保留，但 currentValue 恢复到快照时的状态（快照中不存在的条目 currentValue 恢复为 0）
- AI 通过工具修改的数值随 user 消息撤回而一并回滚（预期行为，AI 响应依赖于该 user 消息）

### 存储开销

每个条目约 20-40 字节（name + currentValue），10 个条目约 200-400 字节/快照。轻量，不会造成存储压力。

## 开发要求

- 除工具和代码外，需做好提示词强化
- 让 AI 知道有哪些工具及限制
  - 数值读写工具（需校验上下限，拒绝越界写入）
  - 随机驱动工具
  - system_RP 介绍等

## 暗骰约束

数值是暗骰，只能在 think 中计算，正文不能直接说明数值存在

示例：
- x "因为尾巴数值的存在所以有了尾巴"
- v "助手在重置时使用了错误的工具，所以有了尾巴"
- x "随机到了0所以风停了"
- v "风停了"
- x "对餐品这个数值进行了随机"
- v "点了一下桌上随机出餐的按钮，所以有了新的餐品"

### 调试/对话内驱动

AI 可在调试情况下和用户探讨对数值进行修改：

调试示例（操作模板）：
- "帮我写个随机开局吧，大概背景是..." -> AI 调用 rp_template_add 等工具帮忙编排提示词和数值设定
- "现在的随机开局有什么" -> AI 调用 rp_template_read 描述当前设定
- "再加个狐狸尾巴吧" -> AI 调用 rp_template_add 修改模板条目

对话内驱动示例（操作面板）：
- "你来花十块钱点一下桌上的随机衣服按钮" -> AI 调用 rp_adjust 金币-10，rp_random 衣服
- "新的敌人出现了，他看起来状态不太好" -> AI 调用 rp_add 新建敌人数值条目并 rp_random

## 边界处理

| 场景 | 处理方式 |
|------|----------|
| DataStore 损坏/解析失败 | catch 后 log，返回空默认值，不 crash |
| rp_add name 为空 | 拒绝，error="名称不能为空" |
| rp_add name 超长 | 截断到 20 字符，返回截断后的结果 |
| min > max（设置页面错误） | 设置下限非法则把上限改为用户值+1；设置上限非法则把下限改为用户值-1 |
| rp_random 时 min == max | 正常返回固定值 |
| 随机开局按钮点击但无 r 标记条目 | 不响应 |
| 快照存储位置 | DataStore，跟随对话 |
| 删除消息时快照 | 一起删 |

## 示例

开局提示词："助手在重置时使用了错误的工具，当尾巴不为0时，在当前对话的外貌有尾巴，1为猫尾巴，2为狗尾巴"

数值：尾巴（下限0，上限2，启用随机）

点击随机开局 -> 随机出0/1/2 -> 以system_Rp身份注入提示词+数值 -> AI演绎（无尾巴/猫尾巴/狗尾巴的动态开局）
