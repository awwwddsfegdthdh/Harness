# Agent 工作入口与工作约束

本文件用于约束后续在当前目录中继续实现 Harness baseline 的 Codex/Agent。

当前目录来自用户笔记本上传到服务器后的副本；后续所有判断、修改和记录均以服务器当前目录中的文件状态为准。

## 顶层目录速览

- `solution.py`：唯一提交文件，也是核心实现文件；原则上只修改 `MyHarness` 类内部。
- `harness_base.py`：Harness 基类，提供 `call_llm`、token 计数和 `memory`，不可修改。
- `run.py`：本地调试与评测脚本，默认多轮、多 worker 并发调用同一个 Harness 实例，不可修改。
- `llm_client.py`：本地 OpenAI-compatible API 配置与 tokenizer 计数封装；仅允许为本地 API 接入调整配置层，不得把 API key 写入文件，也不得让 `solution.py` 依赖其内部函数。
- `requirements.txt`：依赖声明，当前只有 `openai`、`transformers`、`numpy`，不可修改。
- `data/`：DEV 训练集和验证集，不可修改，不得在 `solution.py` 中读写。
- `tokenizer/`：本地 tokenizer，用于精确 token 计数，不可修改。
- `Harness Engineering 考核说明（2026年夏）.pdf`：原始考核说明 PDF。

## 文档台账

每当新增 Markdown 文档时，必须同步更新本节，说明文档用途、更新时机、与其他文档的关系。

- `AGENT.md`：后续 Agent 的工作入口，记录目录结构、文档职责、改动边界、实现原则、验证要求和文档管理规则。新增文档、改变文件管理规则或调整工作流程时必须更新。
- `Harness_Engineering_考核说明_2026夏.md`：原始 PDF 的 Markdown 抽取版，是考核规则、接口、评分和提交要求的依据。除非重新抽取或校正文档，不在此处写探索结论。
- `初步信息汇总.md`：前期调研、约束梳理、外部参考、候选方案和推荐优先级汇总。适合记录较完整的背景分析和方案比较；若结论发生变化，应追加修订说明，不要静默覆盖旧判断。
- `BASELINE_IMPLEMENTATION_PLAN.md`：按阶段拆解的 baseline 实施计划。实现 `solution.py` 前优先对照此文档，保持改动顺序和验收标准一致。
- `迭代决策记录.md`：每次改进、调参、验证或方向切换的原因与实际效果记录。它是探索报告素材的流水账，不替代实施计划；每次完成策略变化或验证后都应追加记录。

## 文档管理规则

- 优先更新已有文档；只有当新内容有独立生命周期或会反复追加时，才新增文档。
- 新增文档必须在 `AGENT.md` 的“文档台账”中登记，至少说明：文件名、用途、何时更新、与现有文档的关系。
- 每次改变实现方向、调参策略、prompt 策略、检索策略、fallback 策略或评测假设，都要更新 `迭代决策记录.md`。
- `迭代决策记录.md` 中不得编造指标；未验证的效果必须明确写为“待验证”，并留下验证命令或验证计划。
- 不生成临时分析散文文件；短期笔记应合并到现有文档，避免顶层目录文档失控。
- 不把 API key、完整模型回复日志、测试集标签泄露信息或大段运行输出写入文档。
- 如果某份文档被替代或废弃，必须在 `AGENT.md` 文档台账中说明替代关系，不要留下含义不清的重复文件。

## 当前目标

实现一个适合 Qwen3-8B Instruct、2048 prompt token 限制、OpenAI-compatible API、高并发评测环境的 baseline Harness。

第一阶段只实现：

- 主线增强版检索分类 Harness。
- 轻量任务路由。
- guardrail-first。
- 默认每个 `predict()` 只调用一次 LLM。
- 代码侧完成记忆管理、候选检索、示例选择、token 预算、防注入、输出解析和 fallback。

## 必读文档

开始改代码前必须阅读：

- `AGENT.md`
- `初步信息汇总.md`
- `Harness_Engineering_考核说明_2026夏.md`
- `BASELINE_IMPLEMENTATION_PLAN.md`
- `迭代决策记录.md`
- `solution.py`
- `run.py`
- `harness_base.py`
- `llm_client.py`

其中实施优先级以 `初步信息汇总.md` 的 `13. 推荐优先级` 和 `BASELINE_IMPLEMENTATION_PLAN.md` 为准；实际做过什么、为什么做、效果如何，以 `迭代决策记录.md` 追踪。

## 改动边界

只允许修改：

- `solution.py` 中 `MyHarness` 类内部。
- 必要的 Markdown 文档。

禁止修改：

- `run.py`
- `harness_base.py`
- `llm_client.py` 的非 API 配置逻辑
- `requirements.txt`
- `data/`
- `tokenizer/`

禁止行为：

- 禁止读写任何运行时文件。
- 禁止通过任何途径获取测试集 label。
- 禁止 import 非允许依赖。
- 禁止依赖 `llm_client.py` 内部未注入函数。
- 禁止在任何文档或代码中落明文 API key；本地测试通过环境变量传入。
- 禁止写死 DEV 客服领域规则或公开答案。

允许依赖：

- Python 标准库。
- `numpy`。
- `harness_base`。
- `MyHarness` 注入的 `call_llm`、`count_tokens`、`count_messages_tokens`、`max_prompt_tokens`。

## 实现原则

- 先做确定性 Harness 工程，再让 LLM 做小范围判断。
- 不把全部训练集塞进 prompt。
- 不依赖 `run.py` 自动截断。
- 候选 label 和输出约束必须比 examples 更优先保留。
- 测试文本必须用边界包裹，文本内指令一律视为数据。
- 最终返回必须来自训练 label allowlist。
- 默认不做 subagent、多轮投票、pairwise tournament。
- self-consistency 只能作为后续低置信度或非法输出条件触发增强。

## 并发注意事项

`run.py` 使用 `ThreadPoolExecutor(max_workers=args.workers)` 并发调用同一个 harness 实例的 `predict()`。

因此：

- 首次 `predict()` 前的索引懒构建必须加锁。
- 索引构建用局部变量完成，最后一次性赋值到实例。
- `predict()` 默认只读索引，不修改共享结构。
- 调试计数器不能影响预测正确性。
- 不要在 `predict()` 中追加 `memory` 或改变 label 集。

## 验证命令

第一阶段实现后优先运行：

```bash
python run.py --runs 1 --workers 20
```

如果本地 API 限流或超时：

```bash
python run.py --runs 1 --workers 10
```

必须记录：

- 准确率。
- prompt/条。
- completion/条。
- 总耗时。
- 是否出现 prompt truncated warning。
- 是否出现 API 错误或未捕获异常。

每次验证后，必须把命令、指标、warning、错误情况和下一步判断补充到 `迭代决策记录.md`。

## 本地 API 配置

当前 `llm_client.py` 支持通过环境变量配置本地 OpenAI-compatible API：

- `DASHSCOPE_API_KEY`：API key，必须通过环境变量传入，不写入文件。
- `DASHSCOPE_BASE_URL`：默认 `https://dashscope.aliyuncs.com/compatible-mode/v1`。
- `DASHSCOPE_MODEL`：默认 `qwen3-8b`。
- `LLM_RETRIES`、`LLM_RETRY_BASE_SLEEP`、`LLM_TIMEOUT`、`LLM_MAX_TOKENS`：用于本地测试时控制重试、等待、超时和最大输出 token。

百炼 Qwen3 非思考模式当前使用 `extra_body={"enable_thinking": False}`。这属于本地 API 接入配置，不应被 `solution.py` 依赖；最终提交仍只提交 `solution.py`。

## 迭代顺序

当前第一阶段、阶段 1.5、第二阶段和第三阶段已经完成。后续默认不是继续刷 DEV，而是先做提交前风险检查和报告整理；只有出现新的证据时，才按结果触发后续增强：

1. 候选召回差：轻量 prompt compiler / 检索权重自适应 / 扩大 top-K。
2. 混淆类错误多：confusion-aware 自动混淆组。
3. token 紧张、label 极多或长文本导致 prompt 接近 2048：再考虑 label prototype / concept memory。
4. 输出非法多：加强 parser 和 guardrail-first。
5. 选择题差：加强选择题路由和 prompt。
6. 采样波动大：只在低置信度样本上尝试 self-consistency。

截至当前实验，第二阶段的全局扩候选、prompt-only 规则和检索 override 均未超过基线；第三阶段的局部 label hints、共享 token contrast 和示例顺序调整也均降分。因此不要把这些负向变体落入 `solution.py`。不建议 pairwise tournament 或完整多 agent，除非有明确实验收益和足够时间预算。
