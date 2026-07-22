# Detector Team — AI 司法系统设计文档

**日期**: 2026-07-22
**状态**: 已更新（CC Workflow → OpenOPC 路线）

---

## 1. 产品概述

Detector Team 是一个 AI 驱动的事实核查与争议分析系统。用户输入核心矛盾/主张，系统自动搜索证据、拆解论点、组织多 Agent 对抗性辩论、投票裁决，输出结构化判决。

**核心理念**：将司法证据标准应用于网络争议——多源二手资料交叉验证，加权可信度，异构模型对抗性辩论还原事实。

---

## 2. 用户旅程

```
1. 用户输入争议 → "武汉大学图书馆事件中，女生指控是否成立？"
2. 系统搜索证据 → 多源检索 (AnySearch)，提取证据单元，标记可信度等级
3. 自动拆解论点 → 争议焦点 → 子论点 → 论据 → 论证
4. Multi-Agent Debate → 主张方 vs 反对方，3 轮辩论 + 法官裁决
5. 输出判决 → 结构化 JSON：结论、置信度、每个主张的存亡、决定性因素
6. (Phase 2) 决策树 UI → React Flow 证据→论点→结论映射
```

---

## 3. 架构路线：Claude Code Workflow → OpenOPC

### 为什么不用 LangGraph

LangGraph、AutoGen、CrewAI 都是通用多 agent 框架，但 Claude Code **本身就是多 agent 运行时**——有 Agent tool（spawn agent），有 Workflow（编排），有 Skills（角色定义），有 AnySearch MCP（证据搜索）。不需要引入第四个编排层。

研究确认的框架数据（agent-harness.ai, 2026）：

| 框架 | Token Overhead | Debate Loop 支持 | 确定性 | 适合 |
|------|---------------|-----------------|--------|------|
| **Claude Code Workflow** | 最低（原生工具） | ✅ pipeline/parallel | 高（脚本驱动） | 辩论协议验证 |
| LangGraph | +9% | ✅ 原生 | 高 | 独立 Python 部署 |
| CrewAI | +18% | 有限 | 低（LLM 选 speaker） | 快速原型 |
| AutoGen | +31% | ✅ | 低（非确定性选 speaker） | 自由对话 |

**结论：Claude Code Workflow 是最快的验证路径，OpenOPC 是最佳的产品化路径。**

### Phase 1: Claude Code Skills + Workflow（验证协议）

```
┌──────────────────────────────────────────────────────┐
│                   User Input                           │
│           "争议主张 + 搜索范围 + 证据偏好"              │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│            Evidence Retrieval Skill                    │
│  - AnySearch MCP 多源搜索                              │
│  - Content extraction (关键页面全文)                    │
│  - 可信度分级: primary / secondary / supporting / weak │
│  - 输出: EvidenceUnit[] (结构化 JSON)                  │
└──────────────────────┬───────────────────────────────┘
                       │ evidence_units
                       ▼
┌──────────────────────────────────────────────────────┐
│         Debate Orchestrator (Workflow Script)          │
│                                                       │
│  pipeline([evidence],                                  │
│    opening_statement,   # 双方开案陈述                   │
│    rebuttal_round,      # 质证反驳 (最多 N 轮)          │
│    closing_statement,   # 结案陈词                       │
│    judge_verdict        # 法官多维裁决                   │
│  )                                                     │
│                                                       │
│  Agent 角色 (每个是一个 Skill):                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐       │
│  │ Proponent  │  │  Opponent  │  │   Judge    │       │
│  │ (Claude)   │  │ (DeepSeek/ │  │ (Gemini/   │       │
│  │            │  │  Qwen)     │  │ 第三个家族) │       │
│  └────────────┘  └────────────┘  └────────────┘       │
│                                                       │
│  公平规则: 同轮双方不能互看对方当前发言                   │
│  证据规则: 每个主张必须绑定 EvidenceUnit                  │
│  终止规则: 硬上限 + Wald SPRT + 语义稳定                  │
└──────────────────────┬───────────────────────────────┘
                       │ structured verdict
                       ▼
┌──────────────────────────────────────────────────────┐
│                 Output Layer                           │
│  - JSON 判决 (Verdict schema)                          │
│  - JSONL 完整辩论记录                                   │
│  - 人类可读摘要                                         │
└──────────────────────────────────────────────────────┘
```

### Phase 2: OpenOPC 提取（独立产品）

Phase 1 验证辩论协议有效后：

| Claude Code 组件 | → | OpenOPC 组件 |
|---|---|---|
| 3 个 Skills (proponent/opponent/judge) | → | 3 个 Company Seats (角色固化) |
| Workflow 脚本 (pipeline 编排) | → | Task Graph (流程固化) |
| AnySearch MCP | → | OpenOPC Plugin (搜索插件) |
| JSONL transcript | → | OPC Memory Layer (组织记忆) |
| — | → | Office UI (独立前端) |

**收益**：`pip install` 独立部署，Office UI 可视化，外部可分享，不依赖 Claude Code 运行时。

---

## 4. 数据模型

### EvidenceUnit

```python
class EvidenceUnit(BaseModel):
    source_id: str
    source_url: str
    source_type: Literal["official_statement", "news_report",
                         "academic_paper", "eyewitness",
                         "social_media", "video", "document"]
    quoted_text: str
    credibility_tier: Literal["primary_authority",
                               "secondary_authority",
                               "supporting",
                               "weak"]
    retrieval_timestamp: str
```

### Claim

```python
class Claim(BaseModel):
    claim_id: str               # e.g. "pro_R1_C1"
    statement: str
    claim_type: Literal["factual", "interpretation", "inference"]
    evidence: list[EvidenceUnit]
    confidence: float           # 0.0 - 1.0 (需 MARGIN 校准)
    rebuts: str | None          # 反驳的目标 claim_id
```

### DebateTurn

```python
class DebateTurn(BaseModel):
    role: Literal["proponent", "opponent"]
    round: int
    position: Literal["opening", "rebuttal", "closing"]
    claims: list[Claim]
    timestamp: str
```

### Verdict

```python
class Verdict(BaseModel):
    conclusion: str
    confidence: float           # 校准后的系统置信度
    reasoning: str
    decisive_factor: str
    claim_outcomes: dict[str, bool]  # claim_id → 通过审查?
    dimensions: dict[str, float]     # 多维评分
    debate_stats: DebateStats
```

### DebateStats

```python
class DebateStats(BaseModel):
    total_rounds: int
    claims_proposed: int
    claims_survived: int
    consensus_score: float
    termination_reason: str      # hard_cap / wald_sprt / semantic_stability
    co_failure_beta: float       # 所有模型同时出错的概率上限
    model_usage: dict[str, int]  # model → tokens
```

---

## 5. Debate 协议（3-Round Protocol + 自适应终止）

### Round 1: Opening Statements
- **公平规则**：双方不能互看对方当前轮发言，仅基于上一轮（初始为证据集）
- 每方提出 top-N 主张，每个绑定 EvidenceUnit（GAVEL 模式）
- **不投票**（防锚定——Ersoz 2026 发现 17/18 陪审团因初始投票僵局）

### Round 2: Rebuttal
- 双方对对方的主张逐条反驳（必须指向 claim_id）
- 可引入新证据支持反驳
- 法官标记每个反驳：sustained / overruled / unclear
- 事实核查：无证据的主张标记为 speculation（deb8flow 模式）

### Round 3: Closing Statements
- 双方强化存活论点
- **不可引入新证据**
- 回应法官 Round 2 标记的 unclear 问题

### Final: Judge Deliberation
**多维评估**（非单一"谁赢了"——D2D 模式）：

| 维度 | 权重 | 评估内容 |
|------|------|----------|
| 事实准确性 | 35% | 主张与证据的一致性 |
| 证据质量 | 30% | 来源可信度、证据链完整性 |
| 逻辑一致性 | 20% | 推理无矛盾、无跳跃 |
| 回应充分性 | 15% | 对对方质疑的回应程度 |

### 终止条件（分层）

| 机制 | 触发条件 | 动作 |
|------|----------|------|
| **硬上限** | round == 3 | 强制进入法官裁决 |
| **Wald SPRT** | 共识分数的序贯似然比越界 | 提前终止（节省 3.7x LLM 调用——Morandi 2026）|
| **语义稳定** | 连续 2 轮 embedding 距离 < 0.05 | 进入裁决 |
| **论据枯竭** | 连续 2 轮无新 claim | 进入裁决 |

---

## 6. 关键设计原则（来自学术文献）

| 原则 | 来源 | 严重性 | 实现方式 |
|------|------|--------|----------|
| **禁止初始投票** | Ersoz 2026; 17/18 僵局 | ⛔ 致命 | Round 1 不投票，辩论完成前不表态 |
| **异构模型** | Kim et al. ICML 2025; Chen 2026 | ⛔ 致命 | 主张方/反对方/法官使用不同模型家族 |
| **Co-failure beta** | Chen 2026; 67 models 测试 | ⚠️ 关键 | 首次运行量化并报告 beta 上限；beta 地板案件需人工审核 |
| **证据绑定** | GAVEL (Xu et al. 2026) | ⛔ 致命 | 每个主张绑定 EvidenceUnit，无证据 = speculation |
| **在线置信度校准** | MARGIN (Armstrong 2026); 闭锁 37-78% Raw-to-Oracle gap | ⚠️ 关键 | 不使用裸置信度，运行时在线校准 |
| **非单调停止** | Zhang & Hu 2026; Q(T) 退化曲线 | ⚠️ 关键 | Wald SPRT + 语义稳定，不超过 3 轮 |
| **多维判决** | D2D (Han et al. 2025); 5 维评估 | ✅ 重要 | 4 维打分，不搞 holistic "谁赢了" |
| **稀疏通信** | Li et al. EMNLP 2024; all-to-all 不必要 | ✅ 重要 | 双方仅看对方上一轮输出，不看同轮 |
| **RLHF 锚定** | Ersoz 2026; GPT-4o 完全免疫说服 | ⚠️ 关键 | 避免用高 RLHF 模型做需要灵活性的角色 |
| **少数派保护** | He et al. 2026; 25% 案例少数派正确 | ⚠️ 关键 | 部署 Minority Sentinel 检测多数派错误的 case |

---

## 7. 实现计划

### Phase 1: Claude Code Skills（本周）

```
~/.claude/skills/detector-team/
├── SKILL.md                          # 主 skill: 入口 + 路由
├── proponent/
│   └── SKILL.md                      # 主张方 agent 角色 + system prompt
├── opponent/
│   └── SKILL.md                      # 反对方 agent 角色 + system prompt
├── judge/
│   └── SKILL.md                      # 法官 agent 角色 + 多维评估 prompt
├── evidence-retriever/
│   └── SKILL.md                      # AnySearch 搜索 + 可信度分级
├── workflows/
│   └── debate.md                     # Workflow 脚本 (pipeline 编排)
├── schemas/
│   └── models.md                     # EvidenceUnit, Claim, Verdict schemas
└── docs/
    ├── RESEARCH.md                   # 研究文献汇总
    └── PITFALLS.md                   # 固化记录
```

### Phase 2: OpenOPC 提取（协议验证后）

- debate skills → OPC company seats
- Workflow script → OPC task graph
- AnySearch MCP → OPC plugin
- JSONL → OPC memory layer
- + Office UI

### Phase 3: React Flow UI（独立部署后）

- React Flow 决策树：证据→论点→结论映射
- ChainOfThought 组件：Agent 推理过程展示
- SSE 实时 debate 流

### Excluded（不做）
- 真实案件判决替代（辅助分析，非替代司法）
- 自建搜索索引（AnySearch）
- 用户系统/认证
- 实时连续辩论（先用回合制验证协议）

---

## 8. 相关竞品与参考

### 学术法庭模拟
- **AgentCourt** (ACL 2025, GitHub 93⭐) — 1000 案例训练律师 agent，对抗进化
- **PROClaim** — 原告/被告/法官/3 人陪审团 + Progressive RAG
- **AgentsCourt** (EMNLP 2024) — SimuCourt 基准：420 中国判决文书
- **D3-Judge** — 50 陪审员 + 自适应停止，节省 40% token
- **AgenticSimLaw** (2026) — 少年法庭 7 轮辩论，90 模型组合 benchmark

### Debate 框架
- **deb8flow** (LangGraph, 39⭐) — 事实核查驱动辩论，3 次失败自动判负
- **Agent4Debate** (ICASSP 2026, 39⭐) — 4 专家 agent/方，200 场 Elo 排名，AI 超人类辩论者
- **xDebate** — 多 agent 推理/辩论框架
- **DebateGraph** — LangGraph + DAG 辩论

### 论证可视化
- **Argdown** (1K+⭐, MIT) — 文本→论证图，v2.0 支持 Markdown 嵌入
- **Kialo Edu** — 分层正反论点树 + 旭日图
- **consensus** (lukeslp, MIT) — 8+ 模型实时投票树，SSE 流，D3.js

---

## 9. 成功标准

- [ ] 输入争议主张，3 agent（主张方/反对方/法官）完成完整辩论
- [ ] 每条主张绑定至少一个 EvidenceUnit
- [ ] 法官输出 4 维评分 + 结构化 Verdict
- [ ] 辩论 JSONL 完整记录，可人工审计
- [ ] Co-failure beta 量化并报告
- [ ] 辩论协议在 5+ 不同类型争议上验证有效（非 trivial 判决）
