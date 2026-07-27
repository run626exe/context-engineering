# Rust/codex-rs

在存放 Rust 代码的 `codex-rs` 文件夹中：

- crate 名称统一以 `codex-` 为前缀。例如，`core` 文件夹中的 crate 名为 `codex-core`。
- 使用 `format!` 时，只要变量可以直接内联到 `{}` 中，就始终这样做。
- 执行本文指令前，如果仓库依赖的命令（如 `just`、`rg` 或 `cargo-insta`）尚未安装，请先安装。
- 绝不要添加或修改任何与 `CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR` 或 `CODEX_SANDBOX_ENV_VAR` 有关的代码。
  - 你在沙箱中运行；每当使用 `shell` 工具时，都会设置 `CODEX_SANDBOX_NETWORK_DISABLED=1`。现有代码中凡使用 `CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR` 的部分，都是在考虑这一事实后编写的。它常用于提前退出作者知道受沙箱限制而无法运行的测试。
  - 同样，通过 Seatbelt（`/usr/bin/sandbox-exec`）启动进程时，子进程会设置 `CODEX_SANDBOX=seatbelt`。需要自行运行 Seatbelt 的集成测试无法在 Seatbelt 内运行，因此适当检查 `CODEX_SANDBOX=seatbelt` 并提前退出也是有意设计。
- 始终依照 https://rust-lang.github.io/rust-clippy/master/index.html#collapsible_if 合并可以合并的 `if` 语句。
- 只要可能，始终依照 https://rust-lang.github.io/rust-clippy/master/index.html#uninlined_format_args 内联 `format!` 参数。
- 只要可能，依照 https://rust-lang.github.io/rust-clippy/master/index.html#redundant_closure_for_method_calls 使用方法引用而非闭包。
- 避免使用布尔值或含义模糊的 `Option` 参数，以免调用者被迫写出 `foo(false)` 或 `bar(None)` 这类难读的代码。若能使调用点具备自解释性，应优先使用枚举、具名方法、newtype 或其他符合 Rust 惯例的 API 形式。
- 如果无法进行上述 API 调整，但仍需在 Rust 中使用简短的位置字面量调用，请遵循 `argument_comment_lint` 约定：
  - 按位置传递 `None`、布尔值、数字字面量等含义不透明的实参时，在前面添加精确的 `/*param_name*/` 注释。
  - 如果一个方法只有一个非 `self` 参数，且方法名与参数名相同，则可豁免，例如对 `fn enabled(&self, enabled: bool)` 调用 `.enabled(false)`。
  - 字符串或字符字面量无需添加此类注释，除非注释确实能显著提升清晰度；lint 有意豁免这些字面量。
  - 注释中的参数名必须与被调用函数签名中的名称完全一致。
  - 可运行 `just argument-comment-lint` 在本地执行检查。它由 Bazel 驱动，因此若 Bazel 尚未预热，首次运行可能较慢；增量运行通常应少于 15 秒。多数情况下，最好更新 PR 并让 CI 负责检查（或提交 PR 后在后台异步运行）。注意，CI 会检查三个平台，而本地运行不会。
- 只要可能，让 `match` 语句穷尽所有情况，避免通配符分支。
- 新增 trait 时，应包含文档注释，说明其职责以及实现方应如何使用。
- 不鼓励在 Rust trait 中使用 `#[async_trait]` 和 `#[allow(async_fn_in_trait)]`。
  - 优先使用原生 RPITIT trait 方法，并为返回的 future 显式添加 `Send` 约束，参考 `3c7f013f9735` / `#16630`。
  - 推荐的 trait 形式：
    `fn foo(&self, ...) -> impl std::future::Future<Output = T> + Send;`
  - 实现只要满足该契约，仍可使用 `async fn foo(&self, ...) -> T`。
  - 不要把 `#[allow(async_fn_in_trait)]` 当作避免显式写出 future 契约的捷径。
- 编写测试时，优先比较整个对象是否相等，而不是逐字段比较。
- 不要为静态定义的值添加测试。
- 不要为已删除的逻辑添加反向测试。
- 不要向 `docs/` 文件夹添加一般产品文档或面向用户的文档。Codex 官方文档位于其他地方。下文所述 app-server API 文档属于例外。
- 优先使用私有模块，并显式导出 crate 的公共 API。
- 如果修改 `ConfigToml` 或其嵌套配置类型，请运行 `just write-config-schema` 更新 `codex-rs/core/config.schema.json`。
- 处理 MCP 工具调用时，优先使用 `codex-rs/codex-mcp/src/mcp_connection_manager.rs` 来处理工具和工具调用的变更。尽量缩小修改范围，利用现有抽象，避免把代码贯穿多层函数调用。
- 不要无必要地调用 `reset_client_session`；让增量检查逻辑决定是否复用上一次请求。
- 如果修改 Rust 依赖（`Cargo.toml` 或 `Cargo.lock`），请从仓库根目录运行 `just bazel-lock-update` 以刷新 `MODULE.bazel.lock`，并在同一变更中包含该锁文件更新。CI 会验证锁文件是否漂移。
- Bazel 不会自动让源代码树中的文件可供 Rust 编译期文件访问。如果新增 `include_str!`、`include_bytes!`、`sqlx::migrate!` 或类似的构建期文件/目录读取，请更新该 crate 的 `BUILD.bazel`（`compile_data`、`build_script_data` 或测试数据）；否则即使 Cargo 通过，Bazel 也可能失败。
- 不要创建仅被引用一次的小型辅助方法。
- 对异步工作添加 tracing 时，请在函数或方法定义上使用 `#[tracing::instrument(...)]`，不要在调用点通过 `.instrument(...)` 给 future 附加 span。添加 instrumentation 前，先检查被调用者或它立即委托的实现方法是否已经被 instrument。
- 避免大型模块：
  - 优先新增模块，而不是继续扩大现有模块。
  - Rust 模块应尽量控制在 500 行代码以内（不计测试）。
  - 如果文件超过约 800 行，除非有充分且已记录的理由，否则应把新功能放进新模块，而不是继续扩展该文件。
  - 此规则尤其适用于容易吸引无关修改的高频文件，例如 `codex-rs/tui/src/app.rs`、`codex-rs/tui/src/bottom_pane/chat_composer.rs`、`codex-rs/tui/src/bottom_pane/footer.rs`、`codex-rs/tui/src/chatwidget.rs`、`codex-rs/tui/src/bottom_pane/mod.rs` 以及类似的核心编排模块。
  - 从大型模块提取代码时，应把相关测试及模块/类型文档一并移向新实现，让不变量靠近拥有它们的代码。
  - 除非改动很简单，否则避免向 `codex-rs/tui/src/chatwidget.rs` 添加新的独立方法；优先使用新模块/文件，让 `chatwidget.rs` 专注于编排。
- 运行 Rust 命令（如 `just fix` 或 `just test`）时要耐心，绝不要尝试通过 PID 杀死进程。Rust 锁可能导致执行变慢，这是预期行为。

在此仓库任何位置完成代码修改后，自动在 `codex-rs` 目录运行 `just fmt`；无需请求许可。此外还要运行测试：

1. 不要直接运行 `cargo test`。使用 `just test`，使测试遵循仓库默认设置。
2. 对实际修改的项目运行对应测试。例如，如果修改了 `codex-rs/tui`，运行 `just test -p codex-tui`。
3. 上述测试通过后，如果修改涉及 common、core 或 protocol，请运行完整测试套件 `just test`。日常本地运行避免使用 `--all-features`，因为它会扩大构建矩阵，并可能显著增加 `target/` 的磁盘占用；只有确实需要完整 feature 覆盖时才使用。项目级或单项测试可以直接运行，无需询问用户；运行完整测试套件前则应先征得用户同意。

在最终提交较大的 `codex-rs` 变更前，请在 `codex-rs` 目录运行 `just fix -p <project>` 修复代码中的 lint 问题。优先通过 `-p` 限定范围，避免缓慢的全 workspace Clippy 构建；只有修改了共享 crate 时才不带 `-p` 运行 `just fix`。运行 `fix` 或 `fmt` 后不要再次运行测试。

## `codex-core` crate

随着时间推移，`codex-core` crate（位于 `codex-rs/core/`）已经变得臃肿。因为它是最大的 crate，往往直接向 `codex-core` 添加新内容，比重构出所需库代码、让新代码既不依赖 `codex-core` 也不继续增大它更容易。

因此：**抵制向 codex-core 添加代码！**

尤其在引入新概念、功能或 API 时，向 `codex-core` 添加内容前应考虑：

- 是否存在一个比 `codex-core` 更适合承载新代码的现有 crate。
- 是否到了为新功能向 Cargo workspace 引入新 crate 的时候；必要时重构现有代码以实现这一点。

同样，在代码评审中，对于会无必要地向 `codex-core` 添加代码的 PR，不要犹豫，应明确提出异议。

## 代码评审规则

### Crate API 表面积

尽量缩小 crate API 表面积。避免不断增加仅供测试使用的辅助函数。

### 模型可见上下文

Codex 会维护一份在推理请求中发送给模型的上下文（消息历史）。

1. 不得重写历史——上下文必须以增量方式构建。
2. 避免频繁更改上下文，以免造成缓存未命中。
3. 不得有无界项目——注入模型上下文的所有内容必须有边界和硬上限。
4. 单个项目不得超过 10K token。
5. 新增的单个项目若可能超过 1K token，必须标记为 P0；此类项目需要额外人工评审。
6. 所有注入片段都必须定义为 `core/context` 中的 struct，并实现 `ContextualUserFragment` trait。

### 破坏性变更

检查以下外部集成表面是否存在破坏性变更：

- app-server API
- 原始 response item 事件（`rawResponseItem/*`），即使仍处于实验阶段
- CLI 参数
- 配置加载
- 从现有 rollout 恢复会话

### 测试编写指南

对于 agent 变更，优先使用集成测试而非单元测试。集成测试位于 `core/suite`，并使用 `test_codex` 创建 Codex 测试实例。

改变 agent 逻辑的功能**必须**添加集成测试：

- 列出需要测试的主要逻辑变更和面向用户的行为。

如果确实需要单元测试，请放入专用测试文件（`*_tests.rs`）。避免在主实现中加入仅供测试使用的函数。

检查是否已有辅助工具可让测试更精简、可读。

### 变更规模指南（800 行）

除机械式改动外，变更总行数不应超过 800 行。复杂逻辑变更应控制在 500 行以内。

如果变更更大，应研究能否拆成可评审的阶段，并找出可以先合入的最小连贯阶段。拆分建议应基于实际 diff、依赖关系和受影响的调用点。

## TUI 样式约定

参见 `codex-rs/tui/styles.md`。

## TUI 代码约定

- 使用 ratatui 的 `Stylize` trait 提供的简洁样式辅助方法。
  - 基础 span：使用 `"text".into()`
  - 带样式 span：使用 `"text".red()`、`"text".green()`、`"text".magenta()`、`"text".dim()` 等
  - 优先使用这些形式，不要直接构造 `Span::styled` 和 `Style`
  - 示例：补丁摘要中的文件行
    - 推荐：`vec!["  └ ".into(), "M".red(), " ".dim(), "tui/src/app.rs".dim()]`

### TUI 样式（ratatui）

- 优先使用 `Stylize` 辅助方法：能用 `"text".dim()`、`.bold()`、`.cyan()`、`.italic()`、`.underlined()` 时，就不要手动构造 `Style`。
- 优先使用简单转换：span 使用 `"text".into()`，行使用 `vec![…].into()`；如果类型推断存在歧义（如 `Paragraph::new`/`Cell::from`），使用 `Line::from(spans)` 或 `Span::from(text)`。
- 计算样式：如果 `Style` 是运行时计算的，可以使用 `Span::styled`（也可使用 `Span::from(text).set_style(style)`）。
- 避免硬编码白色：不要使用 `.white()`；优先使用默认前景色（即不指定颜色）。
- 链式调用：通过链式组合辅助方法提高可读性，例如 `url.cyan().underlined()`。
- 单个项目：优先使用 `"text".into()`；仅当目标类型不明显，或使用 `.into()` 会要求额外类型注解时，才使用 `Line::from(text)` 或 `Span::from(text)`。
- 构建行：当目标类型明确且无需额外类型注解时，用 `vec![…].into()` 构造 `Line`；否则使用 `Line::from(vec![…])`。
- 避免无意义改写：没有明确的可读性或功能收益时，不要在等价形式之间重构（`Span::styled` ↔ `set_style`，`Line::from` ↔ `.into()`）；遵循当前文件的局部约定，也不要仅为了满足 `.into()` 而引入类型注解。
- 紧凑性：优先选择经 rustfmt 后能保持单行的形式；如果 `Line::from(vec![…])` 和 `vec![…].into()` 中只有一个不会换行，就选它。如果两者都会换行，则选择换行更少的形式。

### 文本换行

- 对普通字符串始终使用 `textwrap::wrap` 换行。
- 如果要对 ratatui `Line` 换行，请使用 `tui/src/wrapping.rs` 中的辅助函数，如 `word_wrap_lines` / `word_wrap_line`。
- 需要缩进换行内容时，应尽量使用 `RtOptions` 的 `initial_indent` / `subsequent_indent` 选项，而不是编写自定义逻辑。
- 如果有一组行，需要给所有行添加前缀（第一行和后续行可使用不同前缀），请使用 `line_utils` 的 `prefix_lines` 辅助函数。

## 测试

### 测试模块组织

- 新增测试模块时，应把内容定义在单独的同级文件中，不要内联到实现文件里。
- 使用显式的 `#[path = "..._tests.rs"]` 属性，使测试文件名清晰且易于定位：

  ```rust
  #[cfg(test)]
  #[path = "parser_tests.rs"]
  mod tests;
  ```

- 此要求只适用于新增测试模块。不要仅为了符合本约定而移动或重写现有的内联 `#[cfg(test)] mod tests { ... }` 模块。

### 快照测试

本仓库使用快照测试（通过 `insta`），尤其在 `codex-rs/tui` 中，用于验证渲染输出。

**要求：**任何影响用户可见 UI 的变更（包括新增 UI）都必须包含相应的 `insta` 快照覆盖：如果没有现有快照测试就新增，如果已有则更新。将快照更新作为 PR 的一部分进行评审并接受，使 UI 影响易于审查，并让未来 diff 保持直观。

有意修改 UI 或文本输出时，按以下步骤更新快照：

- 运行测试以生成更新后的快照：
  - `just test -p codex-tui`
- 检查待处理项：
  - `cargo insta pending-snapshots -p codex-tui`
- 直接阅读仓库中生成的 `*.snap.new` 文件评审变更，或预览某个文件：
  - `cargo insta show -p codex-tui path/to/file.snap.new`
- 仅当你确实打算接受此 crate 的全部新快照时，运行：
  - `cargo insta accept -p codex-tui`

如果没有该工具：

- `cargo install --locked cargo-insta`

### 基准测试

可通过 `just bench` 运行 Cargo 基准测试；编写新基准测试时使用 `divan` crate。

使用 `just bench-smoke` 以单次迭代试运行基准测试，确保它可以工作。

### 测试断言

- 测试应使用 `pretty_assertions::assert_eq` 以获得更清晰的差异输出。如果测试模块尚未导入，请在顶部导入。
- 只要可能，优先进行深度相等比较。对整个对象执行 `assert_eq!()`，不要逐字段比较。
- 避免在测试中修改进程环境；优先从上层传入由环境派生的标志或依赖。

### 在测试中启动 workspace 二进制文件（Cargo 与 Bazel）

- 当测试需要启动第一方二进制文件时，优先使用 `codex_utils_cargo_bin::cargo_bin("...")`，不要使用 `assert_cmd::Command::cargo_bin(...)` 或 `escargot`。
  - 在 Bazel 下，二进制文件和资源可能位于 runfiles 中；使用 `codex_utils_cargo_bin::cargo_bin` 可解析出即使在 `chdir` 后仍保持稳定的绝对路径。
- 在 Bazel 下定位 fixture 文件或测试资源时，避免使用 `env!("CARGO_MANIFEST_DIR")`。优先使用 `codex_utils_cargo_bin::find_resource!`，使路径在 Cargo 和 Bazel runfiles 下都能正确解析。

### 集成测试

#### codex_core 集成测试

- 编写端到端 Codex 测试时，优先使用 `core_test_support::responses` 中的工具。
- 默认使用 `TestCodexBuilder::build_with_auto_env()`，确保新测试可在 app/exec 位于不同操作系统的情况下运行。详情参见 `$remote-tests`。
- 所有 `mount_sse*` 辅助函数都会返回 `ResponseMock`；请保留它，以便对发往 `/responses` 的 POST 请求体执行断言。
- 当测试只应发出一个 POST 时，使用 `ResponseMock::single_request()`；需要检查捕获的全部 `ResponsesRequest` 时，使用 `ResponseMock::requests()`。
- `ResponsesRequest` 提供辅助方法（`body_json`、`input`、`function_call_output`、`custom_tool_call_output`、`call_output`、`header`、`path`、`query_param`），使断言可针对结构化 payload，而无需手动深入 JSON。
- 使用提供的 `ev_*` 构造函数和 `sse(...)` 构建 SSE payload。
- 优先使用 `wait_for_event`，不要使用 `wait_for_event_with_timeout`。
- 优先使用 `mount_sse_once`，不要使用 `mount_sse_once_match` 或 `mount_sse_sequence`。

- 典型模式：

  ```rust
  let mock = responses::mount_sse_once(&server, responses::sse(vec![
      responses::ev_response_created("resp-1"),
      responses::ev_function_call(call_id, "shell", &serde_json::to_string(&args)?),
      responses::ev_completed("resp-1"),
  ])).await;

  codex.submit(Op::UserTurn { ... }).await?;

  // 如有需要，对请求体进行断言。
  let request = mock.single_request();
  // 使用 request.function_call_output(call_id) 或 request.json_body() 等进行断言。
  ```

#### app-server 集成测试

- 测试应通过 app-server 的公共 JSON-RPC API 进行。
- 使用与 core 集成测试类似的服务器 mock。
- 默认使用 `TestAppServer::builder().build()` 和 `TestAppServer::send_thread_start_request_with_auto_env()`，确保新测试可在 app/exec 位于不同操作系统的情况下运行。详情参见 `$remote-tests`。

## App-server API 开发最佳实践

这些指南适用于 `codex-rs` 中的 app-server 协议工作，尤其是：

- `app-server-protocol/src/protocol/common.rs`
- `app-server-protocol/src/protocol/v2.rs`
- `app-server/README.md`

### 核心规则

- 所有活跃 API 开发都应在 app-server v2 中进行。不要向 v1 添加新的 API 表面积。
- 保持 payload 命名一致：
  请求 payload 使用 `*Params`，响应使用 `*Response`，通知使用 `*Notification`。
- RPC 方法应以 `<resource>/<method>` 形式暴露，且 `<resource>` 保持单数（例如 `thread/read`、`app/list`）。
- 在线路格式中，字段应始终以 camelCase 暴露，并使用 `#[serde(rename_all = "camelCase")]`；除非 tagged union 或显式兼容性要求需要针对性重命名。
- 在线路格式中，字符串枚举值也应始终使用 camelCase，并配置相匹配的 serde 和 TS `rename_all = "camelCase"` 注解；除非显式兼容性要求需要针对性重命名。
- 例外：config RPC payload 应使用 snake_case，以映射 `config.toml` 的键（参见 `app-server-protocol/src/protocol/v2.rs` 中的 config read/write/list API）。
- v2 请求、响应和通知类型始终设置 `#[ts(export_to = "v2/")]`，让生成的 TypeScript 落到正确命名空间。
- v2 API payload 字段绝不要使用 `#[serde(skip_serializing_if = "Option::is_none")]`。
  例外：有意不带参数的客户端到服务器请求可使用：
  `params: #[ts(type = "undefined")] #[serde(skip_serializing_if = "Option::is_none")] Option<()>`。
- 保持 Rust 和 TS 的线路重命名一致。如果字段或 variant 使用 `#[serde(rename = "...")]`，请添加相应的 `#[ts(rename = "...")]`。
- 对可辨识联合，在两种序列化器中都使用显式 tag：
  `#[serde(tag = "type", ...)]` 和 `#[ts(tag = "type", ...)]`。
- API 边界优先使用普通 `String` ID（需要时在内部完成 UUID 解析/转换）。
- 时间戳应为整数 Unix 秒（`i64`），并命名为 `*_at`（例如 `created_at`、`updated_at`、`resets_at`）。
- 对实验性 API 表面积：
  使用 `#[experimental("method/or/field")]`；需要字段级 gating 时派生 `ExperimentalApi`；当一个方法只有部分字段为实验性时，在 `common.rs` 中使用 `inspect_params: true`。

### 客户端到服务器请求 payload（`*Params`）

- 每个可选字段都必须标注 `#[ts(optional = nullable)]`。不要在客户端到服务器请求 payload（`*Params`）之外使用该注解。
- 可选集合字段（如 `Vec`、`HashMap`）必须使用 `Option<...>` 加 `#[ts(optional = nullable)]`。不要用 `#[serde(default)]` 表达可选集合，也不要在 v2 payload 字段上使用 `skip_serializing_if`。
- 当布尔字段省略即表示 `false` 时，使用 `#[serde(default, skip_serializing_if = "std::ops::Not::not")] pub field: bool`，不要使用 `Option<bool>`。
- 新增 list 方法时，默认实现游标分页：
  请求字段为 `pub cursor: Option<String>` 和 `pub limit: Option<u32>`；
  响应字段为 `pub data: Vec<...>` 和 `pub next_cursor: Option<String>`。

### 开发工作流

- API 行为变化时，更新 app-server 文档/示例（至少更新 `app-server/README.md`）。
- API 形状变化时重新生成 schema fixture：
  `just write-app-server-schema`
  （如果影响实验性 API fixture，还要运行 `just write-app-server-schema --experimental`）。
- 使用 `just test -p codex-app-server-protocol` 验证。
- 避免添加仅断言 `common.rs` 中单个请求字段实验性标记的样板测试；应依靠 schema 生成/测试和行为覆盖。

## Python 开发最佳实践

### 忽略 Python 2 兼容性

本项目使用 Python 3+。不要使用 `__future__` 模块。

如果需要考虑不同 Python 3.xx 小版本之间的功能兼容性，请查看最近的 `pyproject.toml` 中的 `requires-python` 字段，以确定支持的最低运行时版本。

## 平台支持

除非功能明确限定于某一操作系统，否则测试和功能必须支持 Linux、macOS 和 Windows。

Codex 支持在不同操作系统上运行相互连接的 app-server 和 exec-server。有关这类配置的集成测试详情，请参见 `$remote-tests` skill。
