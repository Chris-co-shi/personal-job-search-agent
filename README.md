# Personal Job Search Agent

> 面向求职者的 AI Agent：自动发现、筛选、理解、排序和解释招聘岗位，在用户确认后执行投递，并持续跟踪求职流程。

## 项目定位

Personal Job Search Agent 站在**求职者**一侧，目标不是做招聘方 ATS，也不是做单纯的 JD 分析器。

系统希望把高重复的求职流程从：

```text
人工刷岗位 -> 人工读 JD -> 人工判断 -> 人工投递
```

逐步演进为：

```text
用户求职画像
    ↓
Job Discovery
    ↓
岗位有效性 / 活跃度判断
    ↓
去重与 Logical Job 聚合
    ↓
规则过滤
    ↓
AI JD 结构化理解
    ↓
Candidate Matching
    ↓
Scoring / Ranking
    ↓
分类、解释与风险提醒
    ↓
User Approval
    ↓
Auto Application
    ↓
Application Tracking
```

最终决策权属于用户。V1 中未经用户确认，不执行实际投递。

---

## 核心产品原则

### 1. 求职者优先

所有分析、排序和提醒都以“这个岗位对当前求职者是否值得投入时间”为中心，而不是服务招聘方筛选候选人。

### 2. 原始事实与 AI 派生结果分离

招聘平台原始数据必须保留。AI 的结构化提取、匹配判断、风险推断都是派生数据，不能覆盖原始事实。

```text
RawSourceJob != LogicalJob != JobAnalysis
```

### 3. FACT / INFERENCE / UNKNOWN 分离

系统必须区分：

- `FACT`：平台或 JD 明确提供的信息；
- `INFERENCE`：AI 基于事实得到的推断；
- `UNKNOWN`：当前数据不足，不能猜测。

### 4. LLM 与传统程序明确分工

适合普通程序完成的逻辑不交给 LLM，例如：

- 数值比较；
- 硬规则过滤；
- 状态流转；
- 去重规则；
- 排序公式；
- 数据 CRUD。

LLM 主要负责：

- 非结构化 JD 理解；
- 技能 / 经验语义匹配；
- 风险语义识别；
- 推荐理由与解释；
- 缺失信息识别。

### 5. 人在回路（Human-in-the-loop）

Agent 可以发现、分析、推荐和准备投递，但 V1 中真实投递必须经过用户确认。

### 6. 不绕过平台安全机制

平台接入采用 `Source Adapter + Authorized Session` 的方式设计。系统不以绕过验证码、设备验证、访问控制或平台风控为能力目标。

---

## V1 范围

### User / Job Seeker

- 多份简历；
- 意向岗位；
- 薪资要求；
- 工作地点意向；
- 行业与工作方式偏好；
- 目标公司清单；
- 明确的硬性排除条件；
- 招聘平台账号连接状态。

### Job Discovery

目标支持国内主流招聘来源，并通过统一 Adapter 隔离平台差异。

计划中的来源包括：

- BOSS 直聘；
- 猎聘；
- 智联招聘；
- 前程无忧 51Job；
- 拉勾；
- 后续可扩展公司招聘官网等来源。

用户侧主要提供两种 Discovery Trigger：

1. **关键词 / Search Intent**：系统结合用户画像形成搜索条件；
2. **Job URL**：解析来源后仍进入同一套 Job Discovery Pipeline。

产品侧不以“粘贴任意 Raw JD”作为主要入口，但工程测试必须保留 Mock / Fixture Source，避免核心能力测试依赖真实招聘平台。

### Job Discovery Pipeline

```text
SearchIntent / Job URL
        ↓
Source Adapter
        ↓
RawSourceJob
        ↓
Validity Detection
        ↓
Activity Signals
        ↓
Normalize
        ↓
Duplicate Detection
        ↓
Duplicate Group
        ↓
Logical Job
        ↓
Raw Job Pool
```

Discovery 的职责到 Raw Job Pool 为止。它不负责最终推荐、匹配评分或投递决策。

### Job Validity

用于回答“岗位当前是否仍然有效”：

```text
ACTIVE
CLOSED
EXPIRED
UNKNOWN
```

### Job Activity

用于回答“岗位当前有多值得优先关注”。活跃度只能根据平台能够观测到的 Signal 计算，例如：

- 发布时间；
- 更新时间 / 刷新时间；
- 招聘者活跃信息；
- 是否仍可投递；
- Source-specific activity signals。

没有证据时保持 `UNKNOWN`，不让 LLM 猜测。

### Duplicate / Logical Job

重复岗位**不删除原始记录**。

```text
RawSourceJob A ─┐
RawSourceJob B ─┼── DuplicateGroup ──> LogicalJob
RawSourceJob C ─┘
```

计划支持：

```text
EXACT
PROBABLE
POSSIBLE
```

多个来源可能是同一个现实岗位的不同观测。后续 AI 分析原则上针对 `LogicalJob` 执行，避免重复调用、重复推荐和重复提醒。

### Analysis / Matching / Ranking

后续 Pipeline：

```text
LogicalJob
    ↓
Hard Filter
    ↓
AI JD Parser
    ↓
Candidate Matcher
    ↓
Scoring
    ↓
Ranking
    ↓
Explain / Classify / Risk Detection
    ↓
User Review
```

重要约束：

- LLM 不直接随意给出一个最终“91 分”；
- LLM 负责输出结构化判断及 Evidence；
- 最终 Score / Ranking 由可解释、可调整的策略计算；
- `MatchScore` 与 `RankingScore` 分离。

### User Approval / Application

推荐给用户后的基础动作：

```text
APPLY
SAVE
DEFER
IGNORE
```

只有 `APPLY` 才进入投递流程。

```text
APPROVED
    ↓
Resume Selection
    ↓
Application Preparation
    ↓
User Authorization
    ↓
Apply
    ↓
Application Tracking
```

---

## 当前不做

为了避免第一阶段变成基础设施项目，以下能力暂不进入当前实现范围：

- 招聘方 ATS / HR 候选人管理；
- 未经用户授权的全自动投递；
- 绕过招聘平台验证码、设备验证或风控；
- 多 Agent 架构；
- Knowledge Graph；
- 为使用 LangGraph 而强行引入 LangGraph；
- MCP 大量集成；
- 完整职业规划系统；
- 完整 AI 学习系统。

后续是否引入 LangGraph、RAG、Memory、MCP、Eval 等，由真实业务问题驱动。

---

## 初始工程分层

当前仅创建最小业务骨架，目录代表业务职责，不代表已经冻结最终架构。

```text
personal-job-search-agent/
├── src/
│   └── personal_job_search_agent/
│       ├── domain/          # 领域对象与规则
│       ├── discovery/       # Job Discovery 与 Source Adapter
│       ├── analysis/        # JD Parsing / Matching / Explain
│       ├── ranking/         # Scoring / Ranking
│       ├── application/     # User Approval / Application workflow
│       └── shared/          # 真正跨模块共享的基础能力
├── tests/
├── .env.example
├── .gitignore
├── pyproject.toml
└── README.md
```

暂不提前建立大量 Model、Service、Repository、Agent 类。领域边界确认后逐步增加。

---

## 技术基线

- Python `>= 3.13`
- `uv`：Python 项目与依赖管理
- `pytest`：测试
- `ruff`：静态检查 / 格式化

LLM SDK、数据库、Web Framework、LangGraph 等暂不预设，等对应业务需求进入实现阶段后再选择。

### 初始化

```bash
git clone https://github.com/Chris-co-shi/personal-job-search-agent.git
cd personal-job-search-agent
uv sync --dev
```

运行测试：

```bash
uv run pytest
```

代码检查：

```bash
uv run ruff check .
```

格式化：

```bash
uv run ruff format .
```

---

## 当前阶段

**Phase 0 — Product Scope & Domain Boundary**

当前重点不是堆功能，而是依次明确：

1. `RawSourceJob` 与 `LogicalJob` 的边界；
2. 多来源 Job 如何去重与聚合；
3. Job Discovery Adapter 的统一契约；
4. User / Resume / Preference 与 Job 的关系；
5. Analysis / Match / Ranking 的职责边界；
6. 用户确认与自动投递的人机边界。

完成边界设计后，再开始第一轮领域模型和代码实现。
