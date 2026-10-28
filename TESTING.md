# 测试说明

本文档记录 localagent 的测试体系：如何运行、每个测试模块覆盖什么、实际运行数据、以及测试没有覆盖的部分。

所有数字来自本地实际运行，运行环境：Windows 11，Python 3.13.15，标准库，无网络、无模型调用。

---

## 1. 运行方式

```bash
# 完整单元与集成测试（不联网、不调用模型）
python -m unittest discover -s tests -t . -p "test_*.py"

# 运行单个模块
python -m unittest tests.test_engine -v

# 本地工具链自检（15 项，与测试是两套独立的检查）
agent.cmd --self-test
```

推荐加上 `-W error::ResourceWarning`，把文件未关闭等资源警告当成失败：

```bash
python -W error::ResourceWarning -m unittest discover -s tests -t . -p "test_*.py"
```

---

## 2. 运行结果

### 2.1 总览

| 项目 | 结果 |
|---|---|
| 用例总数 | **59** |
| 通过 | 59 |
| 失败 / 错误 | 0 / 0 |
| `-W error::ResourceWarning` 下 | 通过 |
| 单次运行耗时（3 次） | 1945 ms / 1890 ms / 1866 ms |
| 自检（`--self-test`） | 15 项，全部通过 |

### 2.2 按模块

| 模块 | 用例数 | 通过 | 测试代码行数 |
|---|---|---|---|
| `test_core.py` | 16 | 16 | 117 |
| `test_engine.py` | 21 | 21 | 249 |
| `test_cli.py` | 5 | 5 | 60 |
| `test_interactive.py` | 10 | 10 | 127 |
| `test_regression.py` | 7 | 7 | 75 |
| **合计** | **59** | **59** | **628** |

测试代码行数不含 `tests/support.py`（62 行左右的夹具）与 `tests/__init__.py`（空文件）。

---

## 3. 测试夹具

`tests/support.py` 提供两个基类：

- **`WorkspaceCase`**：`setUp` 时在系统临时目录创建一个独立工作区，并构造真实的 `Ctx`、`OutputStore`、`VarBank`；`tearDown` 整体删除。每个用例互不影响，也不读取用户的真实配置或目录。
- **`QuietUI`**：`UI` 的子类，覆盖 `_write`，测试时不向 stdout 输出。

测试**不 mock** 核心逻辑：`writer`、`reader`、`varbank`、`Ctx.path` 都是真实执行的。只有模型调用被替换成假的 provider（见 3.3）。

### 3.1 文件系统隔离

所有 I/O 都发生在 `tempfile.mkdtemp(prefix="la_…")` 下。`tearDown` 使用 `shutil.rmtree(..., ignore_errors=True)`，清理失败不会让用例失败。

### 3.2 配置隔离

`WorkspaceCase` 通过 `load_config(cfg_path, self.ws, {"workspace": self.ws})` 构造配置，命令行覆盖优先。`test_regression.py` 中的 `ConfigPrecedence` 使用独立的临时目录测试优先级，不读取 `~/.localagent/config.json`。

### 3.3 模型替身

压缩测试需要一个"模型"。`test_engine.py` 定义了三个替身：

| 替身 | 行为 | 用途 |
|---|---|---|
| `_FailProvider` | 抛出 `ProviderError` | 验证失败时视图不变 |
| `_InterruptProvider` | 抛出 `KeyboardInterrupt` | 验证中断时视图不变、锁释放 |
| `_SummaryProvider(text)` | 返回固定文本 | 验证成功替换、超长拒绝、未达标标记 |

这些替身直接赋给 `RollingCompressor._provider`，绕过真实的 provider 工厂。

---

## 4. 各模块详情

### 4.1 `test_core.py`（16 项）

**变量池（VarBankTests，6 项）**

| 用例 | 验证的行为 |
|---|---|
| `test_nested_expansion_multi_level` | 三层嵌套 `{{A}}` 展开到底：`[{{MIDDLE}}]` → `[hello world]` |
| `test_unknown_reference_kept_verbatim` | 未定义的 `{{NOPE}}` 原样保留；`unknown_refs` 返回排序后的列表 |
| `test_cycle_does_not_hang` | `A → B → A` 循环引用在深度上限内返回，不挂起 |
| `test_invalid_name_rejected` | 含空格和 `!` 的变量名抛 `ValueError` |
| `test_persist_roundtrip` | 写入后用新实例读回，内容一致 |
| `test_from_file_stores_exact_bytes` | `set_from_file` 保存的字节与源文件一致（包括 `\r\n`） |

**writer（WriterTests，7 项）**

| 用例 | 验证的行为 |
|---|---|
| `test_create_then_replace_text` | `create` 后 `replace_text` 生效 |
| `test_overwrite_makes_backup` | `overwrite` 前生成且仅生成一个 `.bak.*` 备份 |
| `test_dry_run_does_not_touch_disk` | `dry_run=true` 不改动磁盘文件 |
| `test_replace_lines_keeps_others` | 替换第 2 行，第 1、3 行不变 |
| `test_delete_lines` | 删除第 2 行后结果正确 |
| `test_insert_at_line` | 在第 2 行前插入，原有行顺序保持 |
| `test_write_outside_workspace_rejected` | 工作区外路径抛 `PathError` |

**读取边界（ReadBoundaryTests，3 项）**

| 用例 | 验证的行为 |
|---|---|
| `test_read_outside_without_allowlist_refused` | 白名单为空时，工作区外读取被拒 |
| `test_read_outside_with_allowlist_ok_but_write_still_refused` | 加入白名单后可读；同一路径写入仍被拒 |
| `test_reader_reports_refusal_as_error` | `reader` 对拒绝返回 `is_error=True`，文本含"没有读取权限" |

### 4.2 `test_engine.py`（21 项）

**滚动压缩（RollingTests，7 项）**

| 用例 | 验证的行为 |
|---|---|
| `test_failure_leaves_view_byte_identical` | 压缩模型失败，视图经 `deepcopy` 比较完全不变 |
| `test_interrupt_propagates_and_view_unchanged` | `KeyboardInterrupt` 原样抛出，视图不变，锁随后可再次获取 |
| `test_success_replaces_head_and_keeps_tail` | 成功时首条变为 `_compressed` 摘要，最近一步与原文相等 |
| `test_summary_longer_than_source_rejected` | 摘要比原文长时返回 `None`，视图不变 |
| `test_unmet_target_still_replaces_but_flags` | `target_ratio=0.01` 时仍替换，但 `met_target=False` |
| `test_split_never_separates_tool_use_from_result` | 切分后前段没有悬空的 `tool_use` |
| `test_too_few_steps_not_compressed` | 步数 ≤ `keep_recent_steps` 时切分点为 0 |

**长源码分流（CodeSumTests，6 项）**

| 用例 | 验证的行为 |
|---|---|
| `test_symbols_have_correct_line_numbers` | Python 符号的每个 `def` 行号都指向真正的 `def` 行 |
| `test_summary_carries_doc_and_span` | 摘要包含函数名、首行文档、`varbank:` 引用 |
| `test_syntax_error_falls_back_without_crash` | 语法错误的 Python 文件退回 `regex` 解析，不抛异常 |
| `test_reader_routes_long_source_to_varbank` | 超过阈值的文件输出含"已分流"，长度小于原文 1/4，varbank 中全文无损 |
| `test_routed_range_fetch_matches_source` | 通过 `varbank(range)` 取回的 `def f2()` 与源文件一致 |
| `test_small_file_not_routed` | 低于阈值的文件返回中不含"已分流" |

**插话与并行（InboxTests 2 项，ParallelTests 4 项，外加 1 项辅助）**

| 用例 | 验证的行为 |
|---|---|
| `test_pause_forms` | `#stop` 与 `#stop,话` 的解析 |
| `test_inject_forms` | `#补充一句` 与普通消息的解析 |
| `test_read_only_calls_batched_together` | 只读调用合并为一批，有副作用的单独成批 |
| `test_side_effect_tools_never_parallel` | `do` / `writer` / `varbank set` 不并行 |
| `test_unknown_tool_serial` | 未知工具（如 MCP）串行 |
| `test_has_tool_result_helper` | 辅助函数的判定 |

**会话分组（SessionGroupTests，2 项）**

| 用例 | 验证的行为 |
|---|---|
| `test_children_follow_parent` | 子会话紧跟父会话，按时间排序 |
| `test_orphan_child_listed_as_main` | 父会话已删除的子会话作为主会话列出，`depth=0` |

### 4.3 `test_cli.py`（5 项）

| 用例 | 验证的行为 |
|---|---|
| `test_version` | `--version` 输出 `localagent 0.1.0`，退出码 0 |
| `test_list_tools_contains_all_builtin` | `--list-tools` 包含全部 11 个内建工具名 |
| `test_self_test_passes` | `--self-test` 退出码 0，输出含"通过"，不含 ✗ |
| `test_dump_prompt_exits_zero` | `--dump-prompt` 退出码 0，输出超过 200 字符 |
| `test_no_api_key_is_clean_error_not_traceback` | 清空密钥环境变量后，`--provider openai` 给出"没有 OpenAI 兼容 API key"，输出无 `Traceback` |

这一组通过子进程调用 `python -m localagent`，是唯一经过真实入口的测试。

### 4.4 `test_interactive.py`（10 项）

**`/cd` 与白名单（SwitchAndRootsTests，7 项）**

| 用例 | 验证的行为 |
|---|---|
| `test_cd_switches_every_stateful_piece` | 切换后工作区、输出仓库、记忆、会话存档的路径都位于新目录下 |
| `test_cd_to_missing_dir_raises_and_keeps_state` | 目标不存在时抛 `NotADirectoryError`，工作区不变 |
| `test_addroot_persists_to_agent_json` | `#addroot` 写回配置文件，`extra_read_roots` 含该目录 |
| `test_addroot_is_idempotent` | 重复添加只保留一条 |
| `test_delroot_removes` | `#delroot` 从运行时与配置中移除 |
| `test_addroot_rejects_nonexistent` | 不存在的目录不加入白名单 |

**`#chatlist`（ChatListTests，4 项）**

| 用例 | 验证的行为 |
|---|---|
| `test_child_listed_directly_after_parent` | 子会话紧跟在父会话之后 |
| `test_numbering_matches_grouped_order` | 列表第 2 行是子会话，编号与分组顺序一致 |
| `test_delete_removes_session` | `delete` 后 `get` 返回 `None` |
| `test_images_stripped_from_archive` | 存档中不含 5000 字符的图像 base64 |

### 4.5 `test_regression.py`（7 项）

| 用例 | 验证的行为 |
|---|---|
| `test_lf_file_stays_lf_after_replace` | LF 文件经 `replace_text` 后仍是 LF，不含 `\r` |
| `test_crlf_file_stays_crlf` | CRLF 文件经 `replace_text` 后仍是 CRLF |
| `test_atomic_write_does_not_translate_newlines` | `atomic_write_text` 写入的字节与输入完全一致 |
| `test_cli_overrides_file` | 命令行的 `model` 覆盖配置文件 |
| `test_file_overrides_default` | 配置文件覆盖内置默认值 |
| `test_default_when_no_file` | 配置文件不存在时取内置默认 `claude-opus-5-5` |
| `test_nested_merge_keeps_siblings` | 覆盖 `permissions.mode` 时，同级的 `dangerous` 不丢失 |

---

## 5. 发现过的问题

| 问题 | 发现方式 | 根因 | 修复 | 回归测试 |
|---|---|---|---|---|
| writer 把 LF 文件改写成 CRLF | 测试 `test_delete_lines` 等 4 项失败，期望 `a\nc\n`，实际 `a\r\nc\r\n` | `atomic_write_text` 用 `newline=None`，Windows 文本模式翻译换行 | 改为 `newline=""` | `test_lf_file_stays_lf_after_replace` 等 |
| 视图压缩会改动记录原文 | 手工单元检查（非自动化测试） | `_record` 让记录与视图共享对象 | 视图存深拷贝 | `test_failure_leaves_view_byte_identical` 间接覆盖 |
| varbank range 行号期望写错 | 测试 `test_routed_range_fetch_matches_source` 失败 | 测试代码假设 `def f2` 在第 5 行，实际在第 6 行 | 测试改为从源文件动态查行号 | 测试本身即是修复 |
| 测试中的文件未关闭 | `-W error::ResourceWarning` 下报警 | 测试代码的 `open(...).write(...)` | 改为 `with` | 运行时由警告检测 |

前两项是产品代码的缺陷；后两项是测试代码的缺陷。第三项中，我修改了断言的*期望值的计算方式*，断言内容本身（取回内容须包含 `def f2():` 与 `return 2`）没有放宽。

第一项之前的端到端评测和自检都没有发现，因为它们不比较文件字节。这是一个值得记住的测试盲区：**文本断言会隐藏换行差异**。

---

---

## 6. 没有覆盖的部分

以下行为不在自动化测试中，原因是需要真实模型、真实终端或特定系统组件。

| 领域 | 未覆盖的内容 | 替代手段 |
|---|---|---|
| 压缩摘要质量 | 摘要的召回、误导、幻觉 | `work/compress_eval.py`、`work/long_eval.py`、`work/compress_v2.py`（需要调用模型，结果见 `work/*.md`） |
| multibot / projecttree | 真实派发、并行、子会话内容 | 手工运行 |
| 交互中断 | `#stop`、`#stop,话` 在真实 stdin 上的时序 | 只测了解析函数 `parse_directive` |
| stdin 并发 | `UserInbox` 线程读取与主循环的交错 | 未测试 |
| 识图 | OCR 的实际识别结果、模型视觉输入 | 只测了能力判断与后端检测（`--self-test` 中"识图能力与 OCR"项只检查后端是否存在） |
| 多 provider | OpenAI 兼容协议的真实响应解析 | 只测了缺 key 时的报错，未测真实请求 |
| MCP / 插件 | 真实 stdio 服务的握手与工具调用 | 只在 `--self-test` 中检查数量 |
| 性能 | 大型目录索引、长会话的压缩耗时 | 未测试 |

这些缺口是有意保留的：把它们放进单元测试会让测试变慢、依赖网络或使结果不稳定。如果以后要补，优先级建议是：OpenAI 协议的响应解析（可以用固定的 SSE 字节流测试，不需要网络）、以及 `UserInbox` 的线程行为。

---

## 7. 复现与扩展

新增测试时遵守以下约定：

1. 继承 `WorkspaceCase`，不要直接读写用户目录。
2. 文件断言优先比较字节（`open(p, "rb")`），而不是文本，避免换行差异被隐藏。
3. 模型相关逻辑用替身 provider（参考 `_SummaryProvider`），不调用网络。
4. 期望的行号、数量从源数据动态计算，不要硬编码。
5. 每次修复产品缺陷，都要补一条回归测试，并在第 5 节记录。

运行前后都执行 `-W error::ResourceWarning`。


---

## 8. 压缩前后上下文快照

以下快照由 `work/make_snapshots.py` 生成，完整内容见 `work/context_snapshots.md`。A–D 是确定性的机制快照（固定摘要文本，不代表模型质量），E 是已有的真实模型输出。

### 8.1 滚动压缩（快照 A）

输入：`long_view()`（与 `test_engine.py` 相同），`keep_recent_steps=1`，`target_ratio=0.9`。

| | 条数 | 估算 tokens |
|---|---|---|
| 压缩前 | 7 | 1019 |
| 压缩后 | 3 | 39（摘要）+ 最近一步 |

统计：`ratio=0.038`，`met_target=True`，`replaced_messages=5`。

压缩后的视图结构：

```
[0] assistant [_compressed]: text: '[压缩摘要]\n目标：修复 parser 的空值判断…'
[1] assistant: tool_use do {"command": "x"}
[2] user: tool_result(2 字): 'ok'
```

被替换的 5 条（两次工具调用及其结果）不再出现在视图中；最近一步与压缩前逐字段相等。

**注意**：这里的 `command` 是夹具占位符 `x`，不是真实命令。这个快照说明的是替换的位置，不说明摘要能保留多少信息。

### 8.2 轮次折叠（快照 B）

输入：一个用户请求，两次工具调用（`do` 与 `writer`），一句结论。

| | 条数 | 估算 tokens |
|---|---|---|
| 折叠前 | 6 | 616 |
| 折叠后 | 2 | 84 |

折叠后的 assistant 文本：

```
[用户请求] 修复 parser 的空值判断
[动作]
  - do: x
  -   ↳ def parse_config(x):（1680 字符）
  - writer replace_text lib/parser.py —— 增加 None 保护
  -   ↳ 已写入 lib/parser.py（33 字符）
[结论] 已修复 None 输入的崩溃。
```

`writer` 的 `purpose` 说明（"增加 None 保护"）保留在动作行中。工具结果只保留首行和字符数。

### 8.3 长写入入 varbank（快照 C）

输入：`writer(overwrite)`，`content` 为 1360 字符。

压缩前视图中的参数：`"content": "<1360 字符原文>"`

压缩后视图中的参数：`"content": "[varbank:CTX_LIB_PARSER_PY_CONTENT]"`，`purpose` 保留。

变量池中的原文与写入的 `content` 逐字节相等：**True**。

### 8.4 长源码分流（快照 D）

输入：800 个函数的 `big.py`，22179 字符，约 6145 tokens。

| | 字符 | 估算 tokens |
|---|---|---|
| 压缩前（整文件进上下文） | 22179 | 约 6167 |
| 压缩后（结构摘要） | 2437 | 约 680 |

比例：约 11%。摘要在 2400 字符处截断，只列出前 106 个函数，并附上截断说明与 `varbank:CODE_BIG_PY` 引用。完整符号表和全文都在变量池中。

这个例子说明：**符号数量多时，摘要会被截断**。截断后，模型需要用 `varbank` 按行取回，而不是依赖摘要。

### 8.5 真实模型摘要（快照 E）

来源：`work/long_eval_report.md` 中 L3（上限 300 字）第 1 次级联结果，输入是 85 条消息、约 6255 tokens 的调试会话。这是已有的输出，本次没有重新调用模型。

该摘要的完整内容见 `work/context_snapshots.md` 第 E 节。它保留了 `round`、`Decimal`、`test_export_precision`、`payments` 等关键事实，但这些结论是否准确，需要依据 `work/long_eval_report.md` 中的召回与误导判定来看，不能只凭这一段文字判断。

### 8.6 快照的局限

- A、B、C、D 使用固定的或规则生成的摘要，它们验证的是**结构**（替换位置、引用、字节相等），不验证摘要的**语义质量**。
- 快照 A 的 `command` 是占位符，所以它无法展示摘要对真实命令的保留能力。
- 快照 E 是一次级联的结果，不代表所有运行都会得到同样的摘要。
