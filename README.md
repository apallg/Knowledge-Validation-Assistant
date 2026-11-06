# 企业知识库 RAG 助手自动化测试项目

一个用于展示软件测试与 AI 应用测试能力的**个人实践项目**。包含可运行的知识库问答服务、15 份虚构企业文档、50 条标注样本、接口与权限测试、质量评测和 JMeter 脚本。这里的制度均为虚构，不代表真实企业。

## 快速理解代码

1. 阅读 [`docs/test-plan.md`](docs/test-plan.md)，了解风险、指标和质量准入。
2. 阅读 [`app/rag.py`](app/rag.py) 的 `chat()`，看身份、检索、拒答、生成和引用校验如何串联。
3. 阅读 [`tests/cases/rag_cases.yaml`](tests/cases/rag_cases.yaml) 的样本及原文证据。
4. 阅读 [`tests/test_permissions.py`](tests/test_permissions.py) 和 [`tests/test_documents.py`](tests/test_documents.py)，看如何验证高风险链路。
5. 查看运行产生的 `reports/regression.html` 与 `reports/evaluation-accept.json`。

## 架构

```text
Markdown → 标题/段落分块 → 中文向量化 → Qdrant Local
                                             ↑ 权限过滤
用户问题 → Token 身份识别 → 问题向量化 ────────┘
                                       ↓ Top-3
                          无依据：拒答 / 有依据：模型生成
                                       ↓
                            答案＋授权片段引用
```

默认使用 Mock 生成器，检索和权限逻辑仍真实执行。真实生成器使用可配置的 Chat Completions 兼容接口。模型输出必须附带本次授权检索中的片段标识，否则返回 502。
`/chat` 返回的 `answer_source` 明确区分 `mock_extract`（规则选句）、`external_llm`（外部模型）和 `retrieval_gate`（检索后直接拒答）。`timings_ms.generation` 在 Mock 模式只是规则选句耗时，不能当作 LLM 推理时延。

## Windows 本机启动

需要 Python 3.11、网络下载依赖与向量模型。首次启动向量模型可能较慢。本项目默认只监听 `127.0.0.1:8000`。

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.lock
Copy-Item .env.example .env
.\.venv\Scripts\python.exe -m scripts.project start
```

另开一个 PowerShell 窗口，在同一目录导入示例文档：

```powershell
.\.venv\Scripts\python.exe -m scripts.project seed
```

重复运行 `seed` 不会产生重复片段。打开 <http://127.0.0.1:8000/docs>，页面顶部提供中文操作卡片；先调用 `/health`，再使用 `Authorize（授权）` 输入 `demo-employee-local` 等测试 Token。测试 Token 仅供本机演示，配置在 `.env` 中；API 密钥不要提交到仓库。

调整分块代码或示例文档后，运行 `python -m scripts.project seed --replace` 重建示例索引。

演示顺序：employee 查询“三天请假由谁审批？”→查询“公司食堂周五供应什么甜点？”→以 employee 身份查询 HR 合同内容→切换到 hr 再查询同一问题→查看评测报告。

## 测试与评测

```powershell
.\.venv\Scripts\python.exe -m scripts.project test
.\.venv\Scripts\python.exe -m scripts.project evaluate --split accept
```

`test` 运行无需外部模型 API 的回归，生成 `reports/regression.html`。首次执行会下载 `BAAI/bge-small-zh-v1.5`；模型缓存后可离线执行。`evaluate` 访问正在运行的服务，生成逐条 JSON；默认对冻结的 20 条验收样本评测。服务若为 Mock 模式，报告明确标注为 Mock，不能当作真实模型质量结论。

真实模型配置：

```dotenv
RAG_MODE=live
RAG_LLM_BASE_URL=https://your-provider.example/v1
RAG_LLM_API_KEY=your-secret
RAG_LLM_MODEL=your-model
```

配置后重启服务，先通过 Swagger 调用一条 `/chat` 验证供应商接口兼容，再执行 `evaluate`。`base_url` 应指向 API 根路径，程序会追加 `/chat/completions`。真实模型与 Mock 评测分别保存报告。真实模型调用关闭流式输出和自动重试，超时 60 秒。

评测结果中 `needs_review=true` 的回答需要人工核对原文依据。自动短语命中只是候选检查；包含正确词语的错误结论仍可能被标记为失败。
真实模式还会生成 `*-review.csv`。逐条填入 `supported`、`unsupported` 或 `unclear` 后，用 `python -m evaluation.review 原报告.json 复核表.csv --output reports/reviewed.json` 汇总；原报告保留不变。

## 性能测试

性能脚本使用 Apache JMeter 5.6.3；可从 [Apache JMeter 官方网站](https://jmeter.apache.org/download_jmeter.cgi) 下载后解压运行，无需全局安装。服务和示例文档就绪后，在 Mock 模式分别测 `/retrieve` 与 `/chat`，1、5、10 并发，每档 30 秒延迟加 120 秒测量：

```powershell
jmeter -n -t performance/local.jmx -Jthreads=5 -Jendpoint=retrieve -Jtoken=demo-employee-local -l reports/local-retrieve-5.jtl
py -3.11 performance/parse_jtl.py reports/local-retrieve-5.jtl
```

如需一次顺序运行六档并生成汇总报告：

```powershell
.\.venv\Scripts\python.exe -m performance.run_matrix --jmeter C:\path\to\jmeter.bat
```

可用 `--delay` 与 `--duration` 调整实验时长；报告会记录实际设置。
本次本机 5 秒延迟、20 秒测量的探索结果见 [`docs/performance-report.md`](docs/performance-report.md)，与正式 30/120 秒基线分开标注。

真实模型使用 `performance/live.jmx` 和 20 行问题文件，1 或 3 并发，总请求数 20；先确认 API 费用与限流：

```powershell
jmeter -n -t performance/live.jmx -Jthreads=3 -Jtoken=demo-employee-local -l reports/live-3.jtl
```

报告要附机器配置、文档规模、并发、测试时间和生成模型。端到端 JTL 不能直接说明向量库瓶颈；接口响应中的 `timings_ms` 提供向量化、检索、生成分段耗时。

## 简历使用边界

本项目应写为“个人实践项目”，项目日期采用实际开发日期。量化描述从生成的报告取数，不预写 260 条用例、80% 覆盖或生产级性能结论。三个案例在 [`docs/defect-cases.md`](docs/defect-cases.md) 标为故障注入；如果执行中发现真实缺陷，再单独保留失败与修复证据。

## 本次验证记录（2026-11-05）

- 回归：18 项通过；真实外部模型测试因未配置密钥而未执行。
- 50 条样本的 Mock 演示：46 条通过、4 条开发集问法失败；20 条验收集全部通过。受限文档泄露数为 0。这里的回答质量数字不能代替真实模型评测。
- JMeter 六档本机短时测试与 20 请求脚本检查均已执行；具体参数和数字见 [`docs/performance-report.md`](docs/performance-report.md)。
- 真实模型质量、人工事实核对和真实模型性能仍需配置供应商 API 后执行。

