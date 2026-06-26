# AGENTS.md

## 执行准则
- 默认不要执行编译、构建或测试命令。
- 只有在用户明确要求时，才执行编译/构建/测试（例如 `./gradlew :app:compileDebugKotlin`、`npm run build`、`pnpm run build`）。

当方案更换时，一定要询问用户是否该版本为已发布版本。如果是，请做向前兼容。如果不是，请彻彻底底把老的方案的一切代码全部清理，除非是还能用到的一些部分就继续留着。

如果是方案迭代，则只要在原来的基础上进行正常增删即可。

除非用户要求，禁止写一切的回退代码。优先查找真正的发生原因。这是一条严格执行的规则，回退是正常被禁止的。

严令禁止各种回退逻辑，包括“xxx才会退回”、“降级处理”、“优先 再”、“如果没有 就”这种字眼，绝对禁止！！！绝对禁止！！！这种就是兜底！出现一次严肃惩罚！

禁止写任何的兜底代码，除非用户要求。

用户开始骂的时候，需要道歉以及反思，安抚用户。

用户表达愤怒的时候，需要先停下一切工作，仔细确认用户需求再去实现

如果运行python，项目用的是venv。

严禁使用powershell编辑代码文件，否则会出现严重的编码错误和损坏。

禁用Search files工具，请使用rg

编写Typescript时，对于hook确定的类型，严禁回退兜底成任何unknown/any/带空类型/联合类型。更禁止使用String(??)形式兜底，返回什么就是什么。

如果类型就是string|undef，那么不要直接as string！！那么请使用?? ""或者写个if！！

ts的报错catch后需要log出来

## RP实现状态

### 版本
- 当前版本: 1.11.0RP0.07
- 包名: com.ai.assistance.operitrp
- 软件名: OperitRp

### 已完成
- 数据层: RpValueEntry, RpValueManager, RpPreferences, RpRandomOpeningManager
- 工具层: 13个AI工具（面板7个+模板6个）
- 提示词层: RpIdentityPromptBuilder, ConversationService注入, AIMessageManager处理
- UI层: P面板弹窗编辑, 设置页完整展开, u/s切换, 随机开局按钮
- 导航: 侧栏第五个"角色扮演强化", 标题OperAI_RP
- 签名: key/operitrp-release.jks

### 关键设计
- 随机标记: "-" 表示需要随机（enableRandomOnStart=true时prompt显示"-"）
- 轮次: 直接数user消息数量（排除system_rp），不存计数器
- system_rp: sender="user" + roleName="system_rp"，作为USER类型+[Narrator]前缀注入AI
- 快照: 每条user消息前保存面板快照，删除时恢复
- 随机开局: 点击按钮→随机标记条目→注入system_rp消息（确认数值+提示词+第0轮指令）

### AI工具（13个）
**面板7个:** rp_set, rp_adjust, rp_random, rp_add, rp_remove, rp_rename, rp_update_entry
**模板6个:** rp_template_read, rp_template_set, rp_template_add, rp_template_remove, rp_template_rename, rp_template_update_entry

### 修复记录
- v0.07: 修复400错误（system_rp改为USER类型注入）, 修复轮次计数（改为直接数user消息）, 修复随机开局按钮无响应, 修复P面板R列显示