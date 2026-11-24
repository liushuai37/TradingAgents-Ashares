# TradingAgents-Ashares 开发规范（建议稿）

## 项目上下文

开始任务前阅读 `docs/project-handoff.md`、仓库 README 和现有规范。上游为 TauricResearch/TradingAgents，Fork 为 liushuai37/TradingAgents-Ashares。基准 commit 在 Phase 1 本地核实。

## 开发原则

- 保留原版金融角色、LangGraph、工具循环及 CLI/Python API；优先复用，最小侵入。
- 当前先执行 Phase 1 源码分析；随后 Phase 2 Web 控制台。未经确认不提前实现后续阶段或大规模重构。
- 区分已有能力、待验证能力和拟新增能力。源码分析必须注明实际路径、符号和 commit。
- 数据适配与研究方法解耦；计算和规则由确定性程序执行，AI 负责解释及综合研究。
- 保留数据来源、时间、口径、缺失状态；历史研究不得使用未来信息。
- 不将研究信号视为已验证盈利策略，不实现自动下单。
- 不在代码、文档、日志或 Git 中保存 API Key、密码等凭据；使用环境变量或安全配置。
- 不覆盖已有用户改动；不擅自推送、合并或重置分支。新增依赖需说明理由。
- 每次任务报告改动文件、测试结果、未验证项及后续建议；不得伪造测试或行情。

## 当前阶段交付

Phase 1：`docs/upstream-analysis.md`、`docs/extension-points.md`、`docs/phase-2-design.md`。先检查工作区和依赖，运行可行的最小示例及已有测试。缺少外部凭据时使用明确标注的 mock，记录阻塞项。

Phase 2：FastAPI + React/TypeScript/Vite/Ant Design，SSE 执行事件、任务管理、报告和 PostgreSQL 持久化。具体设计以 Phase 1 源码分析为依据。后续行情图表采用 KLineChart + ECharts，Pro 仅作参考。
