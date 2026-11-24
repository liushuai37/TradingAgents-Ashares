# Phase 1 · 原版源码分析

审计基准为本地 `develop` 的 `9fa1196fffbcfd9de5146d2dd2869df84ab99838`，与本地 `origin/main` 相同。origin 指向 liushuai37/TradingAgents-Ashares，upstream 指向 TauricResearch/TradingAgents。本地无 `upstream/main` 引用，未独立比对远端 HEAD。任务开始时 AGENTS.md 与交接文档已暂存，.idea/ 未跟踪。许可证见 [LICENSE](../LICENSE)（Apache 2.0）。

[README](../README.md) 提供 pip、Docker、CLI 和 Python API 启动说明。[pyproject.toml](../pyproject.toml) 要求 Python ≥3.11，声明 LangGraph、SQLite checkpoint、LangChain 模型客户端、pandas、yfinance、Typer 等依赖。dev extra 提供 pytest，默认排除 integration，[CI](../.github/workflows/ci.yml) 在 Linux 测试 Python 3.11–3.14。以下链接和符号均以该 commit 为准。

## 执行链路

1. **入口**。pyproject 的 `tradingagents = cli.main:app` 指向 [cli/main.py](../cli/main.py) 的 Typer 应用。其 `analyze` 进入 [cli/run.py](../cli/run.py) 的 `run_analysis`，由 `_build_run_config` 合并输入并构造 Graph，随后创建 State、配置回调和 checkpoint、迭代 `stream_run`、记录决策和选择导出。程序入口是 [trading_graph.py](../tradingagents/graph/trading_graph.py) 的 `TradingAgentsGraph.propagate(ticker, trade_date, asset_type, portfolio)`，日期需为非未来的 YYYY-MM-DD。它在 run config 和 checkpoint 作用域中执行 `_run_graph`，通常同步 invoke，debug 则消费流，最终给出 `(final_state, rating)`。
2. **组装**。`TradingAgentsGraph.__init__` 创建 quick/deep 客户端、Memory Log 和 ConditionalLogic，调用 [setup.py](../tradingagents/graph/setup.py) 的 `GraphSetup.setup_graph` 后编译。`create_run_state` 调用 [propagation.py](../tradingagents/graph/propagation.py) 的 `Propagator.create_initial_state`，注入证券身份与可选持仓。身份查询走 [context.py](../tradingagents/agents/context.py) 的 `resolve_instrument_identity`，独立于工具供应商路由。
3. **并行取证**。`setup_graph` 从 START 扇出到选中的分析师和 Memory Log，`add_edge(first_steps, "Bull Researcher")` 等待全部结束。[analyst_execution.py](../tradingagents/graph/analyst_execution.py) 的 `ANALYST_NODE_SPECS` 将四种键映射到节点、报告字段及工具。四者都用 `_analyst_graph`，各自保有消息历史，只向父图返回各自的报告字段。
4. **工具循环**。market、news、fundamentals 在 [turn.py](../tradingagents/agents/analysts/turn.py) 的 `take_turn` 调用绑定相同 TOOLS 的模型。`_tools_or_done` 检查最后的 tool_calls，进入 ToolNode 或结束。每条带调用的模型消息计为一轮，可含多个调用。默认 20 轮，到限转 wrap_up，把工具交互转成文本并用未绑定工具的模型汇总，随后直接结束。默认递归上限 100，构造时验证其大于 `2 * max_tool_rounds + 2`。`social` 无 ToolNode，[sentiment_analyst.py](../tradingagents/agents/analysts/sentiment_analyst.py) 的 `create_sentiment_analyst` 先直接取新闻、StockTwits 和 Reddit，再调用模型。
5. **综合研判**。[conditional_logic.py](../tradingagents/graph/conditional_logic.py) 按 count 和说话者路由。Bull → Bear 交替，达到 `2 × max_debate_rounds` 后进入 Research Manager → Trader。Aggressive → Conservative → Neutral 串行轮转，达到 `3 × max_risk_discuss_rounds` 后由 Portfolio Manager 结束。两个轮数默认为 1。辩手调用普通模型并更新辩论状态，两位 Manager 使用 deep，其余用 quick。Trader 与两位 Manager 通过 [structured.py](../tradingagents/agents/structured.py) 尝试结构化输出并保留文本回退，均无 ToolNode 循环。
6. **落盘与输出**。`record_decision` 写每标的/日期上的 JSON 状态文件并记待结算决策。同标的同日的文件路径可覆盖。成功才清除相应 checkpoint。[rating.py](../tradingagents/agents/rating.py) 的 `run_rating` 处理五档评级与不可解析时的 REVIEW。`save_reports` 是独立导出入口，[reporting.py](../tradingagents/reporting.py) 的 `write_report_tree` 和 [report_html.py](../tradingagents/report_html.py) 生成分节 Markdown、可选 HTML。

## 原版能力与适用边界

| 能力 | 源码依据 | 范围 |
| --- | --- | --- |
| 技术与量价 | [market_analyst.py](../tradingagents/agents/analysts/market_analyst.py) `TOOLS`、[tools.py](../tradingagents/agents/tools.py) `get_stock_data/get_indicators/get_verified_market_snapshot` | 行情、技术指标、快照。多周期与 A 股复权口径待验证 |
| 基本面 | [fundamentals_analyst.py](../tradingagents/agents/analysts/fundamentals_analyst.py) `TOOLS`、[router.py](../tradingagents/dataflows/router.py) `VENDOR_METHODS` | 财务摘要与三表，SEC EDGAR 提供美国申报口径，Yahoo 等按配置覆盖。A 股公告及估值口径待验证 |
| 新闻、宏观、情绪 | [news_analyst.py](../tradingagents/agents/analysts/news_analyst.py) `TOOLS`、`create_sentiment_analyst` | 标的与全球新闻、宏观及预测市场等输入、社交情绪。国内政策和市场广度为后续扩展 |
| State 与持仓 | [state.py](../tradingagents/agents/state.py) `AgentState(MessagesState)`、[portfolio.py](../tradingagents/portfolio.py) `PortfolioContext` | 分析师报告、辩论、计划、评级与上下文；可读取并渲染用户提供的持仓输入 |
| Checkpoint | [checkpointer.py](../tradingagents/graph/checkpointer.py) `get_checkpointer/thread_id/clear_checkpoint`、Graph 的 `begin_checkpoint/checkpoint_input/end_checkpoint` | 默认关闭；每证券 SQLite，以标的、日期及签名区分，签名含模型等配置、图形态和持仓指纹。恢复输入为 None；成功清理，失败可留档 |
| Memory | Graph 的 `_memory_step`、[log.py](../tradingagents/memory/log.py) `TradingMemoryLog`、[settlement.py](../tradingagents/memory/settlement.py) `settle_all_pending/fetch_returns` | Markdown 决策日志。并行结算到期决策，用持有窗口与基准收益生成反思；历史运行只读取当时已结算的经验。结算异常可记录说明并继续 |
| 历史决策评估 | [backtest.py](../tradingagents/backtest.py) `run_backtest/summarize` | 按标的与日期逐项研究，隔离日志，跳过已有记录，按评级汇总方向命中和平均超额，REVIEW 不计分。后续组合研究仍需资金账本、撮合、费用、交易约束等 |
| A 股代码 | [symbols.py](../tradingagents/dataflows/symbols.py) 上海后缀转换、[default_config.py](../tradingagents/default_config.py) benchmark_map | 已有 Yahoo 上海 .SS、深圳 .SZ 路径，上海 .SH 可归一化；覆盖、可用性、除权和历史数据完整性待实测 |

[date_window.py](../tradingagents/dataflows/date_window.py) 的 `as_of/as_of_window` 及未披露数据的 withholding 函数、情绪分析的历史窗口和 Memory 截止过滤是已有时间保护。历史身份查找、供应商口径、修订版本与逐时可用性须按来源单独验证，当前链路无法据此推出全市场无前视保证。Tushare、A 股板块轮动、资金、龙虎榜、制度和专项规则均留待后续阶段。

## 本地运行验证

Windows 隔离环境使用 CPython 3.12.14，安装项目及 dev extra 成功，主要解析版本为 TradingAgents 0.7.1、LangGraph 1.2.24、langchain-core 1.8.4、checkpoint SQLite 3.1.3 和 pytest 9.2.0。默认网络权限下访问 PyPI 返回 socket 10013，经允许联网后完成安装。未修改项目依赖声明。

- `python -m pytest -q tests/test_graph_end_to_end.py`，15 项通过。该套件通过 `ScriptedModel` 和 `offline` fixture 隔离模型与供应商，实际调用 `TradingAgentsGraph.propagate` 和流式路径，覆盖完整决策与落盘、结构化及文本回退、分析师并行、增量报告、工具轮次上限、checkpoint 恢复、Memory 结算和配置隔离。
- `tradingagents --help` 成功显示已安装入口及命令。
- `python -m pytest -q` 沙箱运行结果为 1406 通过、1 失败、6 跳过、1 取消选中，另有 100 项子测试通过。跳过项来自 Windows 与 POSIX 环境差异及可选 Bedrock SDK 缺失，默认筛除 integration。唯一失败为 `tests/test_chatgpt_auth.py::test_the_browser_answer_reaches_the_loopback_and_strays_are_ignored`，沙箱访问 127.0.0.1 被拒（WinError 10013），继而超时。允许本地回环后单独重跑该用例通过。
- 本次未配置可验证的真实模型和数据连接。测试价格、报告和模型响应来自 fixture，A 股行情、供应商 usage 和服务端 SSE 接入仍待验证。
