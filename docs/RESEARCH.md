# Detector Team — 研究文献汇总

基于 2026-07-22 三个研究方向（学术框架 + UI + Agent 通信）的 50+ 论文和 25+ 开源仓库。

---

## 核心学术论文

### Multi-Agent Debate

| 论文 | 出处 | 关键发现 |
|------|------|----------|
| **Du et al. 2023** — Multiagent Debate | MIT/Google Brain | 奠基论文：多 LLM debate 超越单模型和自我反思 |
| **Choi et al. 2025** — Debate or Vote | NeurIPS 2025 | Debate 是鞅过程，不改善期望正确性；多数投票是主力 |
| **Khan et al. 2024** — Persuasive LLMs | ICML 2024 Best Paper | 对抗专家+非专家裁判，76%(LLM)/88%(人类) 准确率 |
| **Ersoz 2026** — 12 Angry AI Agents | arXiv:2605.01986 | 17/18 陪审团僵局；RLHF 对齐导致锚定；禁止初始投票 |
| **Zhang & Hu 2026** — Optimal Stopping | - | Debate 质量非单调：Q(T) = a·log(T+c) − b·T；疲劳退化 |
| **Morandi 2026** — Wald SPRT | - | 序贯检验停止省 3.7x LLM 调用，97% 准确率 |
| **Kim et al. ICML 2025** — Correlated Errors | ICML 2025 | 350+ LLM 测试：更大更好的模型有更高错误相关性 |
| **Chen 2026** — Co-Failure Ceiling | arXiv:2606.27288 | 67 模型测试：beta 地板 5-13%，任何投票机制无法逾越 |
| **Wu et al. ICLR 2025** — Generative Monoculture | ICLR 2025 | 对齐训练显著缩小输出多样性，不足以被采样策略修复 |
| **He et al. 2026** — Minority Sentinel | - | 25% 分歧案例少数派正确；非 LLM 分类器 81.2% 检测精度 |
| **Armstrong 2026** — MARGIN | - | 在线置信度校准，关闭 37-78% Raw-to-Oracle gap |
| **Li et al. EMNLP 2024** — Sparse Communication | Google/DeepMind | all-to-all 通信不必要，稀疏拓扑可比或更优 |
| **Kaesberg et al. ACL 2025** | ACL 2025 | 更多 agent 有帮助，更多轮次反降性能 |
| **Elahi & Di Eugenio 2026** | arXiv:2606.13591 | 聚合置信度比单个 agent 置信度更有区分力 |

### 法庭/司法 AI

| 系统 | 出处 | 特点 |
|------|------|------|
| **AgentCourt** | ACL 2025, 93⭐ | 1000 案例训练律师，对抗进化，专业律师评估 |
| **AgentsCourt** | EMNLP 2024 | SimuCourt: 420 中国判决文书，3 类案件 |
| **PROClaim** | 2026 | 原告/被告/法官/3 陪审团，P-RAG，角色互换 |
| **D3-Judge** | - | 50 陪审员 + 自适应停止，40% token 节省 |
| **DebateCV** | 2026 | Debate-SFT 训练裁判，超越零样本裁判 |
| **D2D** | EMNLP 2025 | 5 阶段辩论，5 维评估，领域专家 profile |
| **RADAR** | 2026 | 角色锚定（政客/科学家/法官），检测半真半假 |
| **GAVEL** | ACL 2026 | Evidence Contract + 机械化审查，溯源感知 |
| **AgenticSimLaw** | 2026 | 少年法庭，7 轮辩论，90 模型组合 benchmark |

### 框架对比

| 框架 | Token Overhead | 确定性 | 适合 |
|------|---------------|--------|------|
| **Claude Code Workflow** | 最低 | 高 | 辩论协议验证（最快） |
| LangGraph | +9% | 高 | 独立 Python 部署 |
| CrewAI | +18% | 低 | 快速原型 |
| AutoGen | +31% | 低 | 自由对话 |

### 辩论格式

| 格式 | 来源 | 结构 |
|------|------|------|
| **Standard 3-Phase** | deb8flow, xDebate, Debate Arena | Opening → Rebuttal(N) → Closing → Judge |
| **Oxford Union** | haive-games, moa-debate | Proposition → POI → Opposition → Devil's Advocate → Judge |
| **Parliamentary** | haive-games | 时间限制，中断规则，主持人 |
| **Competitive** | Agent4Debate (ICASSP 2026) | 4 专家/方(Searcher→Analyzer→Writer→Reviewer)，200 场 Elo |
| **Legislative** | DebateSim (2025) | 5 轮，引用国会记录，跨轮一致性 |

---

## UI 参考

### 论证映射
- **Argdown** (1K+⭐, MIT): 文本→论证图，v2.0 Markdown 嵌入
- **Kialo Edu**: 分层正反论点树 + 旭日图 + 讨论小地图
- **Arguman** (1.4K⭐): because/but/however 三前提模型

### 调查/证据板
- **Maltego**: 实体-关系图，多布局，书签分类
- **Graphistry**: GPU 加速 + Louie.ai 自然语言查询
- **consensus** (lukeslp, MIT): 8+ 模型实时投票树，D3.js，SSE

### 可视化库
- **React Flow** (xyflow) v12: 自定义节点/边，ELK.js 自动布局
- **Reaflow** (2.5K⭐): ELKJS 布局，React 工作流编辑器
- **Cytoscape.js**: 最丰富的图分析工具包
- **AgentTrace UI** (MIT): Agent 推理追踪，时间线+图视图+渐进披露

---

## 关键设计原则速查

| # | 原则 | 来源 | 严重性 |
|---|------|------|--------|
| 1 | 禁止初始投票 | Ersoz 2026 | ⛔ 致命 |
| 2 | 异构模型家族 | Kim et al. ICML 2025 | ⛔ 致命 |
| 3 | 证据绑定 (GAVEL) | Xu et al. 2026 | ⛔ 致命 |
| 4 | 在线置信度校准 (MARGIN) | Armstrong 2026 | ⚠️ 关键 |
| 5 | Wald SPRT 自适应停止 | Morandi 2026 | ⚠️ 关键 |
| 6 | 同轮隔离（公平规则） | Claude Code debate skill | ✅ 重要 |
| 7 | 多维判决（非 holistic） | D2D 2025 | ✅ 重要 |
| 8 | Co-failure beta 量化 | Chen 2026 | ⚠️ 关键 |
| 9 | 少数派保护 | He et al. 2026 | ⚠️ 关键 |
| 10 | 稀疏通信 | Li et al. EMNLP 2024 | ✅ 重要 |
