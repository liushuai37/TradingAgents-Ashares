# Phase 1 · 扩展点与边界

源码基准 `9fa1196fffbcfd9de5146d2dd2869df84ab99838`。运行验证见[原版源码分析](upstream-analysis.md)，实施顺序见 [Phase 2 方案](phase-2-design.md)。

| 切面 | 现有实现与定位 | 复用及阶段 |
| --- | --- | --- |
| 模型 | [factory.py](../tradingagents/llm_clients/factory.py) `create_llm_client/create_tier_client/build_llm_kwargs` 区分原生 Anthropic、Google、Azure、Bedrock 及 [openai_client.py](../tradingagents/llm_clients/openai_client.py) 中兼容供应商；quick/deep 可分别设置模型、供应商及端点。默认和环境变量见 [default_config.py](../tradingagents/default_config.py) | Phase 2 通过服务端配置档复用，保留供应商特有参数，凭据从服务端解析 |
| 工具 | [tools.py](../tradingagents/agents/tools.py) 的模块级工具，[analyst_execution.py](../tradingagents/graph/analyst_execution.py) 的静态 specs，[setup.py](../tradingagents/graph/setup.py) 的 ToolNode 与各分析师 TOOLS | Phase 2 使用原工具集；Phase 4 若引入 Registry，从同一工具集合派生绑定和执行，测试名称、参数/schema、重复注册、错误语义与同步/异步 |
| 数据路由 | [router.py](../tradingagents/dataflows/router.py) 的 TOOLS_CATEGORIES、VENDOR_METHODS、`get_vendor/route_to_vendor`。工具级覆盖分类，显式有序链过滤不支持方法的供应商，default 使用该方法全部可用供应商。区分无数据、供应商不可用和可传播异常；宏观/预测市场可降级 | Phase 2 仅暴露已知配置。Phase 3 可先以映射增补适配器，测试真实覆盖及缺失。身份查询、情绪预取部分来源及 Memory 结算另有路径，要逐一纳入来源和时点审计。供应商路由负责数据能力，工具注册负责模型可调用工具 |
| 配置隔离 | [config.py](../tradingagents/dataflows/config.py) 的 `run_config/run_config_context` 使用 ContextVar，`get_config` 优先读取运行配置；Graph 构造时仍调用进程级 `set_config`，Graph 字段含可变 checkpoint 状态 | Phase 2 每次执行独立 Graph 和进程；工作目录按尝试隔离；共享 Memory 单独串行化并验证写入恢复 |
| 监控 | [trading_graph.py](../tradingagents/graph/trading_graph.py) `stream_run` 消费带子图的 values/tasks，外部只接到消息和状态片段；顶层 task start/end/errors 未透传。[propagation.py](../tradingagents/graph/propagation.py) `get_graph_args` 可注入回调；[stats_handler.py](../cli/stats_handler.py) 统计模型/工具开始及响应 usage。CLI 还有据报告推断的状态 | Phase 2 从原始任务流取节点事件，从回调取模型/工具事件；直接供应商预取不会自动产生工具事件。逐供应商验证 usage、事件形状、父子关联和重复回调 |
| 异常、取消、恢复 | Graph 通过同步 invoke/stream；流在 finally 关闭。运行异常可传给调用者，工具错误与供应商降级另有语义，Memory 错误可保留研究。checkpoint 生命周期由 Graph 管理 | 当前源码未见研究任务级取消、重试记录或服务恢复 API。Phase 2 在进程与数据库层实现；SQLite 节点续跑与 Web 尝试重试分别建模 |

按 `tradingagents/`、`cli/` 中的定义和调用点审计，`create_*_analyst` 等角色工厂和 `GraphSetup.setup_graph` 已存在，动态 Agent Factory/Registry、Workflow Registry、Skill Loader、MCP Client、通用 Hook Manager 未见对外入口。模型供应商工厂、LangChain callbacks、工具静态映射提供明确的复用基础。Phase 4 可先用显式 Python 工厂注册新流程，随后按实际需求加入 Skill 资料和 MCP 适配。回调用于观测，强制校验放在确定性工具或明确路由节点。Phase 5 再将专项知识、流程、确定性规则、边界样例和确认时间独立版本化。
