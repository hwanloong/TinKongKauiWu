# TinKongKauiWu

> 实验性项目。接口、配置和行为都可能随时变化，不保证向后兼容。

[**TinKongKauiWu**](https://github.com/hwanloong/TinKongKauiWu) 是一个本地运行的命令行 agent 执行器。它让大模型通过一组受控工具操作本机的文件、命令和图片，并尝试用几种方式控制上下文长度。

- 运行环境：Windows 10/11，Python 3.10+
- 依赖：仅标准库
- 入口：`agent.cmd`（或 `python -m localagent`）

---

## 目录

1. [快速开始](#快速开始)
2. [压缩算法](#压缩算法)
3. [测试](#测试)
4. [已知限制](#已知限制)
5. [目录结构](#目录结构)

---

## 快速开始

```bash
agent.cmd --self-test                 # 本地自检，不联网
agent.cmd -p "列出工作区里最大的 5 个文件"   # 单次任务
agent.cmd                             # 交互模式，/help 查看命令
```

模型与密钥在 `agent.json` 中配置。密钥读取顺序：`api_key` → `ANTHROPIC_API_KEY` → `ANTHROPIC_AUTH_TOKEN`。切换到 OpenAI 兼容协议：`agent.cmd --provider openai`。

完整的功能说明与验收标准见 [PRODUCT.md](PRODUCT.md)。

---

## 压缩算法

压缩分三件事，互不替代：

| 机制 | 作用对象 | 是否调用模型 |
|---|---|---|
| 滚动压缩 | 已完成的旧步骤（对话与工具调用） | 是 |
| 长文本入 varbank | writer/varbank 的长写入参数 | 否 |
| 长源码分流 | `reader` 读取的长源文件 | 否 |

### 1. 记录与视图分离

Agent 维护两份消息列表：

- `messages`：完整记录，原样保存到 `sessions/`，不压缩。
- `context`：模型实际看到的视图，允许压缩。

`_record()` 写入时对视图做**深拷贝**。如果两者共享同一对象，压缩视图会把记录里的原文也改掉。这是一个会造成数据丢失的错误，测试中有专门的回归用例。

### 2. 滚动压缩（`rolling.py`）

触发条件：视图的估算 token 数 ≥ `compress.trigger_tokens`（默认 120000）。每次请求前检查一次。

**切分点**：只在一个完整步结束的位置切分，即某条 `tool_result` 消息之后。这样切开后，前段里不会留下没有结果的 `tool_use`。保留最近 `keep_recent_steps`（默认 4）步的原文，其余进入压缩区。步数不足时不压缩。

**压缩模型**：通过 `compress.model` / `compress.provider` 单独指定，默认沿用主模型。`thinking_depth` 默认 `low`，`max_output_tokens` 默认 4000，压缩请求不带 adaptive thinking。

**提示词**（生产版，`SUMMARY_SYSTEM`）要求输出固定结构：

```
[压缩摘要]
目标：
已完成：  - <文件>:<位置> —— <改了什么>
关键结论：（最多 5 条）
未完成：
约束：
```

并要求：不复述完整输出，不编造，保留所有 `[varbank:NAME]` 引用，总长度 600 字以内。

**原子替换**：摘要在内存中完整生成后，才一次性替换 `context` 的前段。以下情况视图保持压缩前的状态：

- 压缩模型调用失败；
- 摘要为空；
- 摘要不比原文短（直接拒绝）；
- 压缩过程中收到 `KeyboardInterrupt`（异常原样向上传播，锁释放）。

**目标比例**：`target_ratio`（默认 0.25）。压缩后比例超过它时，仍然替换，但记录 `met_target=false` 并给出告警。这是有意的：超标的摘要比不压缩更好，而且要让你看见它超标了。

### 3. 轮次折叠（`compaction.py`）

每轮结束后，视图里本轮的工具细节被替换为三行：

```
[用户请求] …
[动作] - do: …  ↳ 结果首行（字符数）
[结论] …
```

这一步是**规则拼接，不调用模型**。它的作用是让每轮结束后的视图体积可预测，但信息密度低于滚动压缩的模型摘要。

### 4. 长文本入 varbank

writer、varbank 的参数中，长于 800 字符的 `content` / `value` / `new` / `old` / `text` / `command` 字段：

- 完整原文存入 varbank，变量名由路径和字段名生成（如 `CTX_LIB_PARSER_PY_CONTENT`），变量来源记录为 `工具.模式 路径 | purpose`；
- 视图里的 tool_use 参数替换为 `[varbank:NAME]`；
- 记录中保留原文；
- 工具真正执行时用的仍是完整参数，所以写入结果不受影响。

`writer` 的 `purpose` 字段是可选的，它会进入变量来源和工具结果。模型省略它时，摘要里就没有改动说明。

### 5. 长源码分流（`codesum.py`）

`reader` 在 `auto` / `all` 模式下读取超过 `codesum.threshold_chars`（默认 12000 字符）的文件时：

- 全文存入 varbank，名为 `CODE_<路径>`；
- 返回结构摘要：文件规模、类与函数清单、签名、`[L起-止]` 行号范围、首行文档字符串。

结构摘要的上限是 2400 字符。Python 文件用 `ast` 精确解析；其它语言用正则抽取常见声明。解析失败时退回到头尾预览，全文仍在 varbank 中。

取回函数体：`varbank(action="get", name="CODE_…", mode="range", start=, end=)`。

`reader` 传入 `numbered=false` 可以去掉行号。行号每行约占 7 个字符，只读理解时去掉它，小文件的上下文开销会接近原文。

### 6. 压缩效果：已测过的部分

以下数字来自 `work/` 下的评测脚本，是一次性的实验结果，**不是产品保证**。

| 测试 | 结果 | 备注 |
|---|---|---|
| 长源码分流（`loop.py` 23 KB） | 视图留存约 10% | 全文无损入 varbank，按行取回与源文件逐行一致 |
| 长源码分流（`main.py` 31 KB） | 视图留存约 6% | 同上 |
| 小源码（<12 KB），`numbered=false` | 留存约 98% | 未触发分流 |
| 5 个短场景，生产提示词 | 召回 100%，误导 0，压缩比约 43% | 样本少，模型同源判分 |
| 85 条消息的调试会话，4 段级联 | 300 字上限可用；150 字不可用 | 召回 ≥ 90% 为通过线，每档只跑 2 次 |

需要注意的边界：

- 长会话的评测只跑了一个会话，结论不能外推。
- 召回的判定同时用了子串匹配和同源模型判分。同源判分可能偏宽松。
- 压缩比在这些场景里区分力很弱，主要看召回。

### 7. 未采用的方案（实验中）

`work/compress_v2.py` 评测过一种"状态账本"格式（GOAL / STATE / ROOT_CAUSE / OPEN / EVIDENCE / DONE）。在 5 段级联、4 次重复、判分模型换成另一个模型的条件下，它与旧中文提示词的召回接近，但**压缩度没有优势**，且评测样本只有一个会话。**它没有接入产品代码，当前运行的仍是旧中文提示词。** 是否采用需要更大的样本再判断。

---

## 测试

### 运行

```bash
python -m unittest discover -s tests -t . -p "test_*.py"
agent.cmd --self-test
```

第一条是完整的单元与集成测试（约 2 秒，不联网、不调用模型）。第二条是本地工具链的自检（15 项）。

### 测试模块

| 文件 | 覆盖内容 |
|---|---|
| `tests/test_core.py` | 变量池的嵌套展开、未知引用、循环引用、名字校验、持久化；writer 的各模式、备份、dry_run、越界拒绝；读取白名单的边界 |
| `tests/test_engine.py` | 滚动压缩的原子性（失败、中断、超长摘要、未达标）、切分不拆开配对、长源码分流与按行取回、插话解析、会话分组、并行分批 |
| `tests/test_cli.py` | `--version`、`--list-tools`、`--self-test`、`--dump-prompt`，以及缺少 API key 时的友好报错（不出 traceback） |
| `tests/test_interactive.py` | `/cd` 切换全部有状态组件、`#addroot`/`#delroot` 持久化与幂等、`#chatlist` 的挂载与编号、存档中图像不落盘 |
| `tests/test_regression.py` | 换行符回归（LF 文件不得变 CRLF）、配置优先级 |

`tests/support.py` 为每个用例提供一个临时工作区，结束后删除。测试不读取你的真实配置或目录。

### 测试抓到过的问题

| 问题 | 根因 | 修复 |
|---|---|---|
| writer 把 LF 文件改写成 CRLF | `atomic_write_text` 使用 `newline=None`，Windows 文本模式翻译换行 | 改为 `newline=""`，加回归测试 |
| 视图压缩误伤记录原文 | 记录与视图共享同一对象 | `_record` 对视图深拷贝 |

第一个问题之前的端到端评测和自检都没有发现，因为它们不比较文件字节。

### 测试没有覆盖的部分

以下行为不在单元测试中，需要真实模型或真实终端：

- 压缩摘要的语义质量（只能靠 `work/` 下的评测，且需要调用模型）。
- `multibot` / `projecttree` 的真实派发与并行。
- 交互中 `#stop` / `#stop,话` 的真实中断时序（只测了解析函数）。
- 任何依赖 Windows OCR 或 tesseract 的识图路径（只测了能力判断和后端检测）。

---

## 已知限制

- 仅支持 stdio 传输的 MCP。
- 子智能体（`multibot`）的数量和成本没有上限。
- 轮次折叠是规则拼接，不是模型摘要。
- 长源码结构摘要只对 Python 精确，其它语言为正则近似。
- `numbered=false` 与 `writer.purpose` 都依赖模型遵循提示，代码不强制。
- 压缩的语义准确度没有通过大样本验证。
- 没有并行的写操作，所有写入顺序执行。

---

## 目录结构

```
localagent/          源码
  tools/             内建工具（do read wait filesearch reader writer varbank sheller memory image vision）
  providers/         anthropic / openai 适配
  main.py            入口、命令分发、自检
  loop.py            主循环、记录与视图
  rolling.py         滚动压缩
  compaction.py      轮次折叠、长文本入 varbank
  codesum.py         长源码结构摘要
  varbank.py         变量池
  search.py          文件索引
  sessions.py        会话存档
  subagents.py       multibot / projecttree
  parallel.py        多工具调度
tests/               单元与集成测试
work/                评测脚本与报告（实验用，不属于产品代码）
PRODUCT.md           产品规格
agent.cmd            启动器
agent.json           配置
```
