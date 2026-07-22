# Detector Team — AI 司法系统设计文档

**日期**: 2026-07-22
**状态**: Draft → 待用户审阅

---

## 1. 产品概述

Detector Team 是一个 AI 驱动的事实核查与争议分析系统。用户输入核心矛盾/主张，系统自动搜索证据、拆解论点、组织多 Agent 辩论、投票裁决，输出结构化判决。

**核心理念**：将司法证据标准应用于网络争议——多源二手资料交叉验证，加权可信度，多 Agent 对抗性辩论还原事实。

---

## 2. 用户旅程

```
1. 用户输入争议 → "武汉大学图书馆事件中，女生指控是否成立？"
2. 系统搜索证据 → 多源检索，提取证据单元，标记可信度等级
3. 自动拆解论点 → 争议焦点 → 子论点 → 论据 → 论证
4. Multi-Agent Debate → 主张方 vs 反对方，3 轮辩论 + 法官裁决
5. 输出判决 → 结构化 JSON：结论、置信度、每个主张的存亡、决定性因素
6. (Phase 2) 决策树 UI → 交互式证据→论点→结论映射
```

---

## 3. 系统架构

```
┌──────────────────────────────────────────────────────┐
│                    CLI Interface                       │
│              python -m detector-team debate            │
│                 --claim "争议主张"                      │
│                 --sources web,academic                 │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│              Evidence Retriever                        │
│  - AnySearch 多源搜索                                  │
│  - Content extraction (关键页面全文)                    │
│  - 可信度分级: primary / secondary / supporting / weak │
│  - 输出: EvidenceUnit[] (claim_id + source + tier)    │
└──────────────────────┬───────────────────────────────┘
                       │ evidence_units
                       ▼
┌──────────────────────────────────────────────────────┐
│           Debate Orchestrator (LangGraph)              │
│                                                       │
│  StateGraph 节点:                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────┐│
│  │ Opening  │→│ Rebuttal │→│ Closing  │→│ Judge ││
│  │ Statements│ │  Round   │ │ Statements│ │Verdict││
│  └──────────┘  └──────────┘  └──────────┘  └───────┘│
│                                                       │
│  Agent 角色:                                           │
│  - 主张方 (Proponent): Claude Sonnet                  │
│  - 反对方 (Opponent): DeepSeek / Qwen                 │
│  - 法官 (Judge): Gemini / 第三个模型家族              │
│                                                       │
│  终止条件:                                             │
│  - 硬上限: 3 轮                                        │
│  - 语义稳定: 连续 2 轮 embedding 距离 < 0.05           │
│  - 法官置信度 ≥ 0.9                                    │
│  - Wald SPRT 序贯检验                                 │
└──────────────────────┬───────────────────────────────┘
                       │ structured verdict
                       ▼
┌──────────────────────────────────────────────────────┐
│                 Output Layer                           │
│  - JSON 判决 (Verdict schema)                          │
│  - JSONL 完整辩论记录                                   │
│  - CLI 格式化输出                                      │
│  - (Phase 2) SSE → React Flow UI                      │
└──────────────────────────────────────────────────────┘
```

---

## 4. 数据模型

### EvidenceUnit

```python
class EvidenceUnit(BaseModel):
    source_id: str              # 唯一标识
    source_url: str             # 来源 URL
    source_type: Literal["official_statement", "news_report",
                         "academic_paper", "eyewitness",
                         "social_media", "video", "document"]
    quoted_text: str            # 引用原文
    credibility_tier: Literal["primary_authority",    # 一手权威
                               "secondary_authority",  # 二手权威
                               "supporting",           # 辅助
                               "weak"]                 # 弱证据
    retrieval_timestamp: str
```

### Claim

```python
class Claim(BaseModel):
    claim_id: str               # e.g. "pro_1"
    statement: str              # 主张内容
    claim_type: Literal["factual", "interpretation", "inference"]
    evidence: list[EvidenceUnit]
    confidence: float           # 0.0 - 1.0
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
    conclusion: str             # 最终结论
    confidence: float           # 法官置信度
    reasoning: str              # 法官推理过程
    decisive_factor: str        # 决定性因素
    claim_outcomes: dict[str, bool]  # claim_id → 通过审查?
    debate_stats: DebateStats
```

### DebateStats

```python
class DebateStats(BaseModel):
    total_rounds: int
    claims_proposed: int
    claims_survived: int
    consensus_score: float
    termination_reason: str
    model_usage: dict[str, int]  # model → tokens
```

---

## 5. Debate 协议 (3-Round Protocol)

### Round 1: Opening Statements
- 双方各自陈述立场
- 每方提出 top-N 主张，每个绑定证据
- 法官不做评判，仅记录

### Round 2: Rebuttal
- 双方向对方的具体主张发起反驳
- 必须指向特定 claim_id
- 可用新证据支持反驳
- 法官标记：sustained / overruled / unclear

### Round 3: Closing Statements
- 双方强化存活的论点
- 不可提出新证据（仅可使用前两轮已引入的证据）
- 回应法官在 Round 2 标记为 unclear 的问题

### Final: Judge Deliberation
- 法官多维评估：事实准确性 + 证据质量 + 逻辑一致性 + 回应充分性
- 输出最终 Verdict
- 识别 decisive_factor（决定案件走向的关键因素）

---

## 6. 关键设计原则（来自学术文献）

| 原则 | 来源 | 实现方式 |
|------|------|----------|
| **异构模型** | Khan et al. 2024; Choi et al. 2025 | 主张方、反对方、法官使用不同模型家族 |
| **禁止初始投票** | Ersoz 2026 ("12 Angry AI") | Round 1 不开票，防止锚定效应 |
| **证据绑定** | GAVEL (Xu et al. 2026) | 每个 Claim 必须绑定 EvidenceUnit，无证据的主张标记为 speculation |
| **置信度校准** | MARGIN (Armstrong 2026) | 不使用裸置信度，运行时在线校准 |
| **非单调停止** | Multiple studies | 3 轮硬上限 + 语义稳定检测，防止过度辩论退化 |
| **多维判决** | D2D (Han et al. 2025) | 法官从 4 维度打分，不搞"谁赢了"整体判断 |

---

## 7. MVP 范围

### Phase 1 (当前) — CLI Debate Engine

```
detector-team/
├── pyproject.toml
├── README.md
├── docs/
│   ├── PITFALLS.md
│   └── superpowers/specs/2026-07-22-detector-team-design.md
├── src/detector_team/
│   ├── __init__.py
│   ├── cli.py                  # CLI 入口: python -m detector_team debate
│   ├── models.py               # Pydantic 数据模型
│   ├── evidence/
│   │   ├── __init__.py
│   │   ├── retriever.py        # AnySearch 搜索 + content extraction
│   │   └── credibility.py      # 可信度分级逻辑
│   ├── debate/
│   │   ├── __init__.py
│   │   ├── orchestrator.py     # LangGraph StateGraph
│   │   ├── agents.py           # Agent 定义 + system prompts
│   │   ├── judge.py            # 法官评估逻辑
│   │   ├── consensus.py        # 投票/共识/终止条件
│   │   └── transcript.py       # JSONL 辩论记录
│   ├── prompts/
│   │   ├── proponent.py        # 主张方 system prompt
│   │   ├── opponent.py         # 反对方 system prompt
│   │   └── judge.py            # 法官 system prompt
│   └── config.py               # 配置: models, rounds, thresholds
└── tests/
    ├── test_orchestrator.py
    ├── test_agents.py
    └── fixtures/
        └── sample_claim.json
```

### Phase 2 (后续)
- React Flow 决策树 UI
- ChainOfThought Agent 推理展示
- SSE 实时 debate 流
- 证据到结论交互式映射

### Excluded (不做)
- 实时法律咨询（需要执业资格）
- 真实案件判决（辅助分析工具，非替代司法）
- 自建搜索索引（使用 AnySearch API）
- 用户系统/认证

---

## 8. 技术约束

- **Python 3.11+**, pyproject.toml
- **LangGraph** 作为 debate orchestrator
- **AnySearch** 作为唯一搜索源（符合 CLAUDE.md 要求）
- **至少 2 个不同 LLM 提供商**用于 Agent 角色（Claude + DeepSeek/Gemini）
- **Pydantic v2** 验证所有数据结构
- **JSONL** 记录完整辩论过程
- **CLI-first**: `python -m detector_team debate --claim "..."`

---

## 9. 成功标准

- [ ] 输入争议主张，输出结构化判决（JSON）
- [ ] 3 轮辩论完整执行，不崩溃
- [ ] 每条主张绑定至少一个证据单元
- [ ] 法官输出多维评估（非单一分数）
- [ ] JSONL 完整记录可供人工审计
- [ ] CLI 可用: `pip install -e . && python -m detector_team debate --claim "..."`
