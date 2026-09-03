---
name: "agent-dev"
description: "大模型 Agent 应用开发规范：Agent 框架选型（自研循环、LangGraph、OpenAI Agents SDK、AutoGen、低代码平台等等）、中间件、LLM 调用工程处理、结构化输出、Prompt 模板。开发任何 Agent/LLM 应用时使用。"
---

# 大模型 Agent 应用开发规范

LLM 应用工程模式。适用于 Agent 编排、LLM 调用类开发的领域细则。**规范本身框架无关**——先按场景选框架，再套用本规范。

**本规范下的所有 Python 代码同时遵循 python-style skill**（snake_case、中文注释、公开函数 docstring 含 Args/Returns）。

## 1. 成本与安全红线（任何框架，审查先查这三条）

1. **调用预算上限**：每个 Agent 必须有限次逻辑（LangChain 生态挂 `ModelCallLimitMiddleware`；OpenAI Agents SDK 用 `max_turns`；自研循环用调用计数器），限次耗尽走降级而非继续调用。**无上限的 Agent 禁止上线**——LLM 循环失控是真实的烧钱事故来源。
2. **外部资源超时按体积设定**：图片下载 `timeout=60`，视频下载 `subprocess.run(..., timeout=120)`。无超时的外部调用等于挂起整个工作流。
3. **敏感工具先审批**：退款、删除、对外发布类工具执行前挂起等人工确认（LangGraph 用 `interrupt()` + `Command(resume=...)`；其他框架用审批队列或人工确认节点）；未批准的安全默认是拒绝。

## 2. 框架选型（按场景，渐进式，禁止一步到位）

先问"工作流长什么样"，再选工具。**能不引框架就不引**——每层抽象都有调试成本：

| 场景特征 | 推荐方案 | 理由 |
|----------|---------|------|
| 单轮/简单工具调用 | 原生 SDK function calling + 自研 while 循环 | 百行内可读完、调试透明，框架是负资产 |
| 多步骤工作流、条件分支、需断点恢复/人工审批 | LangGraph | 状态机+checkpointer 原生支持挂起恢复 |
| 多 Agent 角色协作（讨论、互审） | AutoGen / CrewAI | 对话编排是其核心抽象 |
| 需求方自助搭建、逻辑简单 | Dify / Coze 等低代码平台 | 交付快，复杂逻辑仍回代码 |
| 已有框架的项目 | 沿用既有框架 | 选定技术栈后不随意更换 |

选型输出物：一行选型理由写进架构文档，说明被排除方案及原因（对照表见 eng-thinking skill）。

**选定 LangGraph 后的实现要点**（其他框架忽略本节，套用对应机制）：

- 状态机用 `TypedDict` 共享状态；`add_conditional_edges` 的路由映射表**显式声明**（`{"big": "big", "small": "small"}`），禁止隐式字符串返回
- 多平台分发用 `Send` 列表并行（如同时投递 `fetch_douyin` / `fetch_weibo`），不要串行轮询
- 管道型 Agent 的各阶段（账号定位 → 热点监控 → 内容生成）独立成模块，由 `main.py` 编排，任一阶段可单独运行和测试——**此条框架无关，所有方案适用**

## 3. 横切关注点（清单框架无关，实现按生态选）

重试、限次、摘要、审批一律独立于业务代码（中间件/guardrail/装饰器均可），**业务节点里出现 `while retry` 循环即是违规**：

| 横切点 | 职责 | 常见实现 |
|--------|------|---------|
| 限次熔断 | 防失控烧钱（红线 1） | LangChain `ModelCallLimitMiddleware` / SDK `max_turns` / 自研计数器 |
| 失败重试 | 指数退避，仅重试瞬时错误（限流/超时/5xx），4xx 不重试 | 中间件或装饰器，与限次联动 |
| 上下文治理 | 长对话自动摘要、清理旧工具结果，防 token 膨胀 | `SummarizationMiddleware` / `ContextEditingMiddleware` 或自研裁剪 |
| 人工审批 | 敏感工具执行前确认（红线 3） | `HumanInTheLoopMiddleware` / `interrupt()` / 审批队列 |

## 4. LLM 调用四要素（每个调用点逐项自查，框架无关）

| 要素 | 约束 |
|------|------|
| 结构化输出 | Pydantic schema 驱动（LangChain `with_structured_output`；OpenAI `response_format`/JSON mode），禁止裸字符串解析后正则抠字段 |
| 重试 | 走横切层，指数退避；校验失败可把错误反馈给模型自我纠正一轮 |
| 超时 | 模型调用与外部资源分别设定；注意 SDK 内置重试与自研重试叠加导致放大 |
| 流式 | 长任务逐步输出阶段进度（LangGraph `stream_mode="updates"`；其他框架用流式 API/进度回调） |

## 5. Prompt 与配置（框架无关）

- 每个工作流节点的 prompt **单独定义在 prompts.py**，固定分区模板（角色定义 → 输入数据 → 任务维度 → 输出要求），动态内容 f-string 注入；要求模型输出带序号的结构化内容便于下游解析
- 密钥和模型配置走独立 config 模块（pydantic-settings 读环境变量），业务代码只读 `setting.xxx`

## 6. 降级策略（框架无关）

外部依赖（视频下载、图片生成、LLM 调用）必须有降级链，失败不阻塞主流程：

- 三级降级示例：`videodl → yt-dlp → stub 占位`
- 重试耗尽后走确定性兜底（规则分类、模板回复、转人工），**降级结果也是合法输出**，不抛异常中断
