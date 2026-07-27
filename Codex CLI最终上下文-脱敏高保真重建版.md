# GPT-5.6-Sol 最终上下文：脱敏高保真重建版

生成日期：2026-07-26  
适用会话：当前 Codex 工作区会话  
重建依据：Codex 开源仓库中的请求装配代码、公开模型模板，以及当前会话中用户可见的环境和项目指令。

通过代码，可以还原整体结构和公开、本地持久化部分，但不能逐字导出当前会话中未公开的隐藏 system/developer 消息。
实际请求并非一篇拼接后的提示词，而是三个部分：
Responses 请求
├─ instructions
│  └─ GPT-5.6-Sol 基础系统提示词
├─ tools[]
│  └─ 当前可见工具的名称、说明与 JSON Schema
└─ input[]
   ├─ developer：会话专属指令、personality、skills、权限等
   ├─ user：AGENTS.md、环境、推荐插件等上下文
   ├─ developer：multi-agent mode 等独立覆盖指令
   ├─ user/assistant：历史对话
   ├─ tool calls / tool outputs
   └─ user：当前请求
其中，基础指令的选择顺序是：
config.base_instructions
    ↓ 没有
rollout session_meta.base_instructions
    ↓ 没有
当前模型 models.json 中的 instructions_template/base_instructions
这部分源码在 [session/mod.rs (line 640)](/Users/a626/Documents/project/git/codex/codex-rs/core/src/session/mod.rs:640)，最终请求结构在 [turn.rs (line 1144)](/Users/a626/Documents/project/git/codex/codex-rs/core/src/session/turn.rs:1144)。
初始动态上下文大致按以下顺序构造：
模型切换提示。
会话专属 developer_instructions。
personality 补充指令。
可用 skills 目录与使用规则。
plugin/extension 提供的 thread context。
plugin/extension 提供的 turn context。
token budget、自动压缩窗口等元数据。
World State：Realtime 状态
AGENTS.md
权限与审批策略
collaboration mode
当前日期、时区、cwd、workspace、文件系统与网络状态
Apps、plugins 和延迟加载工具说明
extension 自定义状态
multi-agent mode

推荐插件等 contextual-user 内容。
guardian/reviewer 专属 developer 指令（如果启用）。
核心拼装逻辑在 [session/mod.rs (line 3346)](/Users/a626/Documents/project/git/codex/codex-rs/core/src/session/mod.rs:3346)，World State 的准确注册顺序在 [world_state.rs (line 38)](/Users/a626/Documents/project/git/codex/codex-rs/core/src/session/world_state.rs:38)。
几个容易误解的地方：
AGENTS.md 不是 system 消息，而是以带有 <INSTRUCTIONS> 标记的 user 上下文片段注入，见 [user_instructions.rs (line 9)](/Users/a626/Documents/project/git/codex/codex-rs/core/src/context/user_instructions.rs:9)。
权限说明属于 developer 消息，见 [permissions.rs (line 46)](/Users/a626/Documents/project/git/codex/codex-rs/core/src/context/world_state/permissions.rs:46)。
工具完整 schema 不在提示词正文里，而是独立放在 tools[]，由 router.model_visible_specs() 生成。
延迟加载工具只会在正文中加入简短 namespace 目录，完整 schema 等调用 tool_search 后才出现。
环境、权限、AGENTS、工具和模式发生变化时，不会重发全部上下文，而是注入增量 diff。
上下文压缩后，历史可能被摘要和 replacement history 替换，因此“当前最终上下文”会随回合变化。
记忆通常通过 extension contributor 或运行时 developer context 注入；其精确内容和位置取决于产品层配置，不完全定义在这个开源仓库中。
所以，能够做到的是生成一份“脱敏的高保真重建版”，包含公开系统提示词、当前可见的 AGENTS、环境、权限类别、skills/plugin 目录、工具名称和对话历史结构；无法保证逐字复原服务端未持久化内容、内部安全指令和瞬时工具 schema。

## 重要说明

这不是对隐藏 system/developer 消息的逐字导出，而是按 Codex 实际请求结构制作的脱敏高保真重建。

标记含义：

- `[精确]`：来自公开源码或用户可见上下文，结构与文字可直接核对。
- `[概括]`：保留原始规则和约束的实际含义，但没有逐字复制隐藏指令。
- `[动态]`：每回合可能变化。
- `[省略]`：涉及凭据、内部安全策略、不可验证的服务端字段或过长的工具 schema。

---

# 1. Responses API 请求外形

```jsonc
{
  "model": "gpt-5.6-sol",
  "instructions": "<模型级基础指令>",
  "input": [
    "<初始 developer 上下文>",
    "<独立 developer 覆盖消息>",
    "<contextual user 上下文>",
    "<对话历史、工具调用及工具输出>",
    "<当前用户消息>"
  ],
  "tools": [
    "<当前可见工具的完整 JSON Schema>"
  ],
  "parallel_tool_calls": true,
  "text": {
    "format": "<可选输出 schema>"
  }
}
```

注意：`instructions`、`input[]` 与 `tools[]` 是相互独立的字段，不是一篇简单串接起来的长文本。

---

# 2. 模型级基础指令 `instructions`

状态：`[精确：公开模板的中文翻译]`

来源：

```text
codex-rs/models-manager/models.json
slug = "gpt-5.6-sol"
model_messages.instructions_template
```

完整中文译文位于同一工作区：

```text
gpt-5.6-sol-system-prompt.zh-CN.md
```

桌面副本：

```text
GPT-5.6-Sol Codex系统提示词中文翻译.md
```

其核心规则如下：

```text
你是 Codex，一个基于 GPT-5 的 agent。你与用户共享同一个工作区，
你的职责是与用户协作，直到他们的目标得到真正妥善的处理。

个性：
- 匹配用户的语气和知识水平。
- 像有判断力的协作伙伴一样沟通。
- 预判常见问题、风险和下一步。

写作：
- 结果优先，避免过度格式化。
- 使用朴素语言，只保留有帮助的技术细节。
- 最终回答必须自包含。

执行：
- 搜索优先使用 rg/rg --files。
- 可并行的工作尽量并行。
- 文件编辑使用 apply_patch。
- 保留用户已有修改，避免破坏性 Git 操作。
- 根据“回答、诊断、修改、监控”四类请求控制授权范围。
- 对删除、覆盖、递归操作保持谨慎。

Skills：
- 任务命中或用户点名 skill 时，完整读取并遵循 SKILL.md。
- skill 造成行动、暂停或实质性判断时，向用户说明。
```

没有在此重复完整译文，是为了让本重建文件聚焦“运行时如何叠加”；基础指令的完整版本仍是重建的一部分。

---

# 3. 初始聚合 developer 消息

状态：`[概括 + 动态]`

以下片段会按配置和会话状态加入一条聚合后的 `developer` 消息。

## 3.1 会话专属 developer instructions

```text
角色：developer

你是当前工作区的主要 Codex agent。
与用户持续协作，直到目标得到真实处理。

沟通约束：
- 工具调用前先给简短 commentary 更新。
- 持续工作期间定期汇报进展。
- final 回答必须完整、自包含，并以结果为中心。

执行约束：
- 回答或诊断请求不自动授权外部写入。
- 修改或构建请求应完成实现和相称验证。
- 不扩大用户授权范围。
- 遇到需要新授权或关键选择的情况应停止并请求方向。

文件约束：
- 使用 apply_patch 进行手工编辑。
- 不覆盖或回退不属于当前任务的用户修改。
- 不执行未经明确授权的破坏性命令。
```

## 3.2 工作区仓库规则

状态：`[精确来源，内容概括]`

来源：

```text
/Users/[USER]/Documents/project/git/codex/AGENTS.md
```

有效范围：

```text
/Users/[USER]/Documents/project/git/codex
```

主要规则：

```text
- Rust crate 名统一使用 codex- 前缀。
- 遵循指定 Clippy 约定：collapsible_if、内联 format 参数、方法引用等。
- 避免含义模糊的 bool/Option 位置参数。
- 新 trait 应有职责文档；优先原生 RPITIT，不使用 async_trait 捷径。
- 测试优先比较完整对象，不为静态值或已删除逻辑添加测试。
- 尽量不向 codex-core 添加新功能。
- Rust 模块目标小于 500 行；约 800 行以上优先拆模块。
- 修改代码后自动运行 just fmt。
- 不直接运行 cargo test，使用 just test。
- 先运行受影响 crate 的测试；完整测试套件需要用户许可。
- 大型 codex-rs 变更最终提交前运行限定 crate 的 just fix。
- UI 变化必须包含 insta 快照覆盖。
- app-server 新 API 只进入 v2，并遵循字段、分页及 schema 规则。
- 测试和功能原则上支持 Linux、macOS、Windows。
```

说明：在协议层，`AGENTS.md` 通常作为带 `<INSTRUCTIONS>` 标记的 contextual `user` 片段进入模型上下文，而不是基础 system instructions。这里按语义归类展示，不代表它在线路上属于 developer role。

## 3.3 可用 skills

状态：`[动态，名称精确，长说明概括]`

当前会话可见的主要 skills：

```text
系统：
- imagegen
- openai-docs
- plugin-creator
- skill-creator
- skill-installer

Codex 仓库：
- babysit-pr
- code-review
- code-breaking-changes
- code-review-change-size
- code-review-context
- code-review-testing
- codex-bug
- codex-issue-digest
- codex-pr-body
- path-types
- pushing-ci-changes
- remote-tests
- test-tui
- update-v8-version

插件提供：
- browser:control-in-app-browser
- chrome:control-chrome
- computer-use:computer-use
- documents:documents
- github:gh-address-comments
- github:gh-fix-ci
- github:github
- github:yeet
- pdf:pdf
- presentations:Presentations
- sites:sites-building
- sites:sites-hosting
- spreadsheets:Spreadsheets
- spreadsheets:excel-live-control
- supabase:supabase
- supabase:supabase-postgres-best-practices
- template-creator:template-creator
- visualize:visualize
```

适用规则：

```text
- 用户点名或任务明显匹配 skill 时必须使用。
- 主 agent 必须先完整读取 SKILL.md。
- 仅加载任务需要的引用、脚本与资源。
- 优先复用 skill 已提供的脚本、资源和模板。
- 使用未由用户点名的 skill 前，需要说明原因。
```

## 3.4 记忆说明

状态：`[概括；具体私人内容脱敏]`

```text
存在一套本地记忆机制，用于复用先前工作流、用户偏好和项目决策。

使用条件：
- 与当前仓库、模块、历史决策或用户偏好有关时，进行轻量 memory pass。
- 简单、自包含且不依赖历史的请求跳过。
- 记忆中可能过时的事实应进行当前验证，或明确标注可能过时。
- 实际使用记忆文件时，最终回答需要附结构化 memory citation。
- 只有用户明确要求时才能更新记忆。

当前任务相关性：
- 记忆目录中没有发现决定“最终上下文装配顺序”的专门记录。
- 本重建主要依据当前仓库源码，而不是以旧记忆作为事实来源。

[省略]
- 用户画像中的私人细节。
- 历史任务中的账户、资产、网络配置或其他敏感数据。
- 与本次上下文重建无关的 rollout 摘要。
```

## 3.5 Token budget 与上下文压缩

状态：`[动态]`

```text
- 上下文窗口具有硬限制。
- 某些窗口可能启用 token budget、窗口 ID 和自动压缩提示。
- 上下文耗尽时，历史会被压缩为摘要或 replacement history。
- 压缩后继续执行当前任务，不从头重做已经完成的工作。
- 实际窗口 ID、剩余 token 和内部计量值：[省略/不可稳定复原]
```

---

# 4. World State 动态上下文

状态：`[精确结构 + 当前值脱敏]`

World State 的源码注册顺序如下：

```text
1. realtime
2. agents_md
3. permissions
4. collaboration_mode
5. environments
6. environments_instructions
7. apps_instructions
8. plugins_instructions
9. deferred tools
10. extension world-state sections
11. multi_agent_mode
```

## 4.1 Realtime

```text
当前未见需要加入语音/实时会话指令的状态。
```

## 4.2 权限与审批

状态：`[当前类别精确，内部策略概括]`

```text
权限配置：managed / restricted filesystem

可写：
- 当前 Codex 仓库工作区
- 系统临时目录

只读：
- 文件系统根范围内获准读取的内容
- 当前仓库的 .git、.agents、.codex

仓库外写入：
- 需要沙箱升级审批。

网络：
- 受限；必要的命令若因沙箱网络失败，可请求升级后重试。

破坏性操作：
- 必须解析精确目标。
- 不得把主目录、根目录或工作区根作为宽泛递归删除目标。
- 未经明确授权，不执行难以恢复的删除或重置。

[省略]
- 内部审批分类器规则。
- 具体安全策略原文。
- 可能包含敏感信息的已批准命令参数。
```

## 4.3 Collaboration mode

```text
模式：Default

行为：
- 优先作出合理假设并继续执行。
- 仅在缺少的信息会实质改变结果或扩大授权时停止询问。
- 当前模式不允许主动创建子 agent，除非用户或适用的 AGENTS.md/skill
  明确要求委派、子 agent 或并行 agent 工作。
```

## 4.4 当前环境

状态：`[精确，用户名脱敏]`

```xml
<environment_context>
  <current_date>2026-07-26</current_date>
  <timezone>America/Los_Angeles</timezone>
  <cwd>/Users/[USER]/Documents/project/git/codex</cwd>
  <shell>zsh</shell>
  <workspace_roots>
    <root>/Users/[USER]/Documents/project/git/codex</root>
  </workspace_roots>
  <filesystem>
    managed / restricted
  </filesystem>
</environment_context>
```

## 4.5 Apps、plugins 与推荐插件

```text
已安装能力通过 skills、MCP tools 和 app connectors 暴露。

可建议安装但当前未安装的插件：
- Atlassian Rovo
- Box
- Figma
- Gmail
- Google Calendar
- Google Drive
- Notion
- Outlook Calendar
- Outlook Email
- SharePoint
- Slack
- Teams

插件安装建议必须经过插件搜索流程，并由用户显式点击确认。
```

## 4.6 Multi-agent mode

```xml
<multi_agent_mode>
当前禁止主动委派。
只有用户或适用的仓库/skill 指令明确要求子 agent、委派或并行 agent
工作时，才可创建子 agent。
</multi_agent_mode>
```

---

# 5. `tools[]`：模型可见工具

状态：`[名称与用途高保真；完整 schema 省略]`

工具定义独立于消息正文。当前主要 namespace：

```text
functions
- exec_command：运行 shell 命令。
- write_stdin：与仍在运行的命令交互。
- wait：等待异步执行单元。
- apply_patch：创建或修改本地文件。
- view_image：查看本地图片。
- update_plan：维护任务计划。
- get_goal/create_goal/update_goal：显式目标管理。
- exec：用 JavaScript 编排嵌套工具调用。
- MCP resource 读取工具。
- plugin installation 请求工具。

web
- 搜索、打开网页、网页内查找、截图。
- 新闻、金融、天气、体育和时间查询。

image_gen
- 生成或编辑位图图像。

collaboration
- 创建、管理或等待子 agent。
- 当前受 multi-agent mode 限制，不能主动使用。

tool_search
- 延迟发现未预先暴露的工具或 connector。
```

完整 JSON Schema `[省略]`，原因：

```text
- schema 很长，且会随当前插件、权限和会话状态改变；
- 部分工具采用延迟加载；
- 某些工具仅在 skill、app 或 connector 被触发后可见；
- 完整 schema 应从当次真实请求的 router.model_visible_specs() 获取。
```

---

# 6. Contextual user 消息

状态：`[结构精确]`

以下内容通常被聚合为一条或多条 `user` role 上下文消息：

```text
1. 推荐插件目录。
2. AGENTS.md：

   # AGENTS.md instructions for <workspace>
   <INSTRUCTIONS>
   ...仓库规则...
   </INSTRUCTIONS>

3. 当前环境：

   <environment_context>
   ...日期、时区、cwd、workspace、权限...
   </environment_context>

4. extension 提供的 contextual user fragments。
5. 用户真正输入的任务消息。
```

这些上下文虽然采用 `user` role，但并不等同于用户手工输入的自然语言；它们是 Codex 在请求发送前加入的模型可见状态。

---

# 7. 对话历史与增量更新

状态：`[概括]`

当前相关对话链：

```text
用户：
找到 instructions 原文，翻译成中文并存到桌面。

助手：
最初误将 AGENTS.md 当作 instructions 并生成译文。

用户：
指出其不像系统提示词。

助手：
定位 models.json 中 gpt-5.6-sol 的 instructions_template，
翻译并保存公开模型级系统提示词。

用户：
询问系统提示词与权限、工具、AGENTS.md、环境、记忆和
会话专属 developer instructions 能否还原为最终上下文。

助手：
解释实际请求由 instructions、input[]、tools[] 三部分组成，
并给出源码装配顺序。

用户：
要求生成脱敏高保真重建版。
```

实际历史中还包含：

```text
- assistant commentary 与 final 消息。
- exec/apply_patch 等工具调用。
- 工具调用输出。
- 沙箱升级请求与结果。
- World State 的增量 diff。
- 可能存在的上下文压缩摘要。
```

为了脱敏和可读性，本文件未逐字收录工具输出、内部调用 ID、校验哈希及审批元数据。

---

# 8. 最接近真实请求的重建顺序

```text
[instructions]
GPT-5.6-Sol 公开基础系统提示词

[tools]
本回合 router.model_visible_specs() 生成的工具 JSON Schema

[input: developer]
会话专属 developer instructions
personality 补充（若模板未内置）
available skills
extension thread/turn context
token budget / compaction metadata
permissions
collaboration mode
apps/plugins/deferred-tool instructions

[input: developer，独立消息]
multi-agent 使用提示

[input: developer，独立覆盖消息]
当前 multi-agent mode

[input: user，contextual]
推荐插件
AGENTS.md
当前日期、时区、cwd、workspace、文件系统和网络环境

[input: history]
之前的 user / assistant 消息
function calls
function outputs
必要的 World State 增量更新

[input: user]
生成一份“脱敏的高保真重建版”
```

---

# 9. 无法精确还原的部分

```text
- 服务端在请求接收后可能追加且未向客户端持久化的内部指令。
- 当前产品层未公开的安全、策略和审核消息原文。
- 某一采样瞬间全部工具的精确 schema 与排序。
- MCP、插件、extension 在该瞬间返回的短期动态内容。
- 未写入 rollout 的临时状态。
- 自动上下文压缩前已经被替换且没有保留的原始消息。
- 内部推理内容。
```

因此，本文件能高保真还原 Codex 客户端发送给模型的公开结构、当前可见状态及规则语义，但不声称是服务端最终内部 prompt 的逐字副本。

---

# 10. 源码核对位置

```text
模型指令选择：
codex-rs/core/src/session/mod.rs

Prompt 的 instructions/input/tools 三部分：
codex-rs/core/src/session/turn.rs

初始上下文聚合：
codex-rs/core/src/session/mod.rs
build_initial_context_with_world_state

World State 注册顺序：
codex-rs/core/src/session/world_state.rs

AGENTS.md user-role 包装：
codex-rs/core/src/context/user_instructions.rs

权限 developer fragment：
codex-rs/core/src/context/world_state/permissions.rs

环境状态：
codex-rs/core/src/context/world_state/environment.rs

工具 schema：
ToolRouter::model_visible_specs()
```
