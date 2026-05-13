# PPT: Agentic 环境构造与环境模拟
## Agentic Environment Construction & Simulation

---

## Slide 1: 封面

**Title**: Agentic 环境构造与环境模拟
**Subtitle**: 从合成环境到大规模Agent训练的前沿研究综述
**Date**: 2026.05

---

## Slide 2: 目录 (Outline)

1. 背景与动机 (Background & Motivation)
2. 核心问题：为什么需要环境构造？(Why Environment Construction?)
3. 代表性工作概览 (Representative Works)
4. Agent World Model (AWM) — Snowflake
5. Synthetic Computers at Scale — 长时序生产力模拟
6. Neural Computers — 神经计算机范式
7. Simia — LLM作为环境模拟器
8. Gym系列框架 (AgentGym / GEM / MLGym)
9. 技术路线对比 (Technical Approach Comparison)
10. 未来展望 (Future Outlook)

---

## Slide 3: 背景与动机

### Agent训练面临的核心瓶颈

| 瓶颈 | 描述 |
|------|------|
| **环境稀缺** | 真实环境构建成本高、覆盖度低 |
| **交互有限** | 静态数据集无法提供多轮交互反馈 |
| **多样性不足** | 手动构建的环境难以覆盖真实世界的多样性 |
| **可扩展性差** | 传统benchmark数量有限、难以规模化 |

### 范式转变
- **从 Static Dataset → Interactive Environment**
- **从 Human-crafted → Synthetically Generated**
- **从 Single-turn Eval → Multi-turn RL Training**

---

## Slide 4: 核心问题

### 为什么需要"环境构造"?

```
传统路径:  人工定义环境 → Agent交互 → 评估
                ↓ 成本高、数量少、覆盖不足
                
新范式:    自动合成环境 → 大规模Agent RL训练 → 持续进化
                ↓ 可扩展、多样、可验证
```

**关键挑战:**
1. 如何自动化生成**可执行**的环境？
2. 如何保证环境的**真实性**与**多样性**？
3. 如何设计**可验证**的奖励信号？
4. 如何**规模化**到数千甚至数百万环境？

---

## Slide 5: 代表性工作全景图

```
┌─────────────────────────────────────────────────────────────┐
│              Agentic 环境构造与模拟 技术图谱                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Code-Driven  │  │ LLM-as-Sim   │  │ Neural Sim   │     │
│  │  合成环境     │  │  LLM模拟环境  │  │  神经模拟器   │     │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤     │
│  │ • AWM        │  │ • Simia      │  │ • Neural     │     │
│  │ • Synthetic  │  │ • AgentGym   │  │   Computers  │     │
│  │   Computers  │  │              │  │ • World      │     │
│  │              │  │              │  │   Models     │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                             │
│  ┌──────────────────────────────────────────────────┐      │
│  │        Gym-like 标准化框架                         │      │
│  │   GEM / AgentGym / MLGym                         │      │
│  └──────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

---

## Slide 6: Agent World Model (AWM)

### 基本信息
- **来源**: Snowflake AI Research + UNC-Chapel Hill
- **论文**: arXiv:2602.10090 (ICML 2026)
- **Stars**: 345+ on GitHub

### 核心思想
> 用LLM合成pipeline自动生成**可执行**的、基于SQL数据库的tool-use环境，通过MCP接口暴露，用于大规模多轮Agentic RL训练

### 关键数据
- **1,000** 个合成环境
- 每个环境包含: SQLite数据库 + MCP接口 + 验证器
- 发布模型: Arctic-AWM-4B / 8B / 14B

---

## Slide 7: AWM — 合成Pipeline

### 五步合成流程

```
Step 1: Scenario Generation (场景生成)
    ↓  从种子集生成1000个独特场景
Step 2: Task Generation (任务生成)
    ↓  每场景10个任务，作为环境需求
Step 3: Database Synthesis (数据库合成)
    ↓  Schema定义 + 初始数据填充
Step 4: Interface Synthesis (接口合成)
    ↓  API Spec → MCP环境代码
Step 5: Verification Synthesis (验证合成)
    ↓  SQL-based LLM Judge / Code-based Judge
```

### 核心优势
- **Code-driven状态转换**: 避免LLM幻觉，保证状态一致性
- **MCP统一接口**: 标准化的tool调用协议
- **可扩展**: 10→526个环境，性能持续提升
- **可验证**: 双重验证机制确保正确性

---

## Slide 8: AWM — 架构图

```
┌─────────────────────────────────────────────────┐
│                 AWM Synthesis Pipeline            │
├─────────────────────────────────────────────────┤
│                                                  │
│  Seed Scenarios → LLM Generator → 1000 Scenarios │
│       ↓                                          │
│  Tasks/Requirements → DB Schema → SQLite DBs     │
│       ↓                                          │
│  API Specs → MCP Server Code → Executable Envs   │
│       ↓                                          │
│  Verification Code (SQL Judge / Code Judge)       │
│                                                  │
├─────────────────────────────────────────────────┤
│              Runtime Architecture                 │
├─────────────────────────────────────────────────┤
│                                                  │
│  Agent ←→ MCP Protocol ←→ Environment Server     │
│                              ↓                   │
│                         SQLite Database           │
│                         (State Store)            │
│                              ↓                   │
│                         Verifier                  │
│                    (Reward Signal)                │
└─────────────────────────────────────────────────┘
```

---

## Slide 9: Synthetic Computers at Scale

### 基本信息
- **论文**: arXiv:2604.28181 (2026)
- **标题**: Synthetic Computers at Scale for Long-Horizon Productivity Simulation

### 核心思想
> 创建大规模**合成计算机环境**（含完整文件系统和文档），模拟长时序用户生产力工作，产生丰富的学习信号

### 关键创新
1. **合成计算机**: 逼真的文件夹层级 + 内容丰富的文档(docx, xlsx, pptx)
2. **双Agent模拟**:
   - Agent A: 基于计算机环境创建生产力目标
   - Agent B: 作为用户在计算机上完成长周期工作
3. **超长时序**: 每次模拟 > 8小时 agent runtime, 平均 > 2000 turns
4. **可扩展至十亿级**: Personas可大规模生成

---

## Slide 10: Synthetic Computers — 方法论

```
┌────────────────────────────────────────────────────────────┐
│            Synthetic Computers Pipeline                      │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  1. Persona Generation                                     │
│     └── 多样化职业/角色/背景 (可扩展到10亿)                 │
│                                                            │
│  2. Computer Environment Synthesis                         │
│     ├── 文件夹层级结构                                      │
│     ├── 文档/表格/演示文稿                                  │
│     └── 用户特定的工作上下文                                 │
│                                                            │
│  3. Long-Horizon Simulation                                │
│     ├── Goal Agent → 生成月级别生产力目标                    │
│     └── Worker Agent → 在计算机上完成目标                    │
│         ├── 文件系统导航                                    │
│         ├── 与模拟协作者交互                                 │
│         └── 产出专业文档                                    │
│                                                            │
│  4. Learning Signal Extraction                             │
│     └── 经验学习 → Agent性能提升 (in-domain & out-of-domain)│
└────────────────────────────────────────────────────────────┘
```

### 实验规模
- 1,000 个合成计算机
- 每次模拟: 8+ 小时 / 2000+ turns
- 显著提升agent在生产力任务上的表现

---

## Slide 11: Neural Computers (NC)

### 基本信息
- **论文**: arXiv:2604.06425 (2026)
- **标题**: Neural Computers

### 核心思想
> 提出**神经计算机(Neural Computers)**概念：将传统计算机的计算、内存、I/O统一在一个学习到的运行时状态中

### 方法
- 将NC实例化为**视频模型**
- 从指令、像素、用户动作中roll out屏幕帧序列
- 在CLI和GUI场景中学习I/O对齐和短时序控制

### 长期目标: Completely Neural Computer (CNC)
- 稳定执行
- 显式重编程
- 能力持久复用

### 当前挑战
- 已获得: 基础接口原语、I/O对齐、短时序控制
- 待解决: 例程复用、受控更新、符号稳定性

---

## Slide 12: Simia — LLM作为环境模拟器

### 基本信息
- **来源**: Microsoft Research
- **论文**: arXiv:2511.01824 (2025)
- **标题**: Simulating Environments with Reasoning Models for Agent Training

### 核心思想
> 用LLM直接作为环境模拟器，消除对手工构建环境的依赖

### 双框架设计

| 框架 | 方法 | 描述 |
|------|------|------|
| **Simia-SFT** | 监督微调 | 从1.5k种子轨迹合成90k训练数据 |
| **Simia-RL** | 强化学习 | LLM模拟环境提供状态转换+奖励 |

### 关键成果
- Qwen2.5-32B (Simia训练) → **58.9** on tau2-Bench
- GPT-4o → 54.2 (被超越!)
- Qwen3-8B (Simia) → **44.0** on OfficeBench vs GPT-4o的31.1

### 优势
- 环境无关的训练方法
- 极低成本的扩展性
- 小模型超越大模型

---

## Slide 13: Gym系列标准化框架

### AgentGym (ACL 2025)
- 7个真实场景 / 14个环境 / 89个任务
- 统一框架支持评估+训练
- 实时反馈 + RL自我提升

### GEM: A Gym for Agentic LLMs (2025)
- 类似OpenAI Gym的LLM标准化框架
- 24个环境、异步向量化执行
- 兼容5种RL训练框架(PPO, GRPO, REINFORCE等)
- 支持单轮+多轮交互

### MLGym (Meta, 2025)
- 13个AI研究任务
- 开放式环境、自动化研究流程
- 评估前沿LLM的研究能力

---

## Slide 14: 技术路线对比

| 维度 | AWM | Synthetic Computers | Neural Computers | Simia | AgentGym/GEM |
|------|-----|--------------------|--------------------|-------|--------------|
| **环境来源** | LLM合成代码 | LLM合成文件系统 | 从I/O学习 | LLM直接模拟 | 人工+扩展 |
| **状态表示** | SQL Database | 文件系统+文档 | 视频帧(像素) | LLM隐状态 | 多种 |
| **接口** | MCP Tool-use | 文件系统+CLI | CLI/GUI | API调用 | 标准化接口 |
| **可执行性** | 完全可执行 | 完全可执行 | 模拟执行 | 模拟执行 | 完全可执行 |
| **规模** | 1000环境 | 1000计算机 | 概念验证 | 90k轨迹 | 14-24环境 |
| **时序** | 多轮(中) | 超长(2000+turns) | 短时序 | 多轮(中) | 单/多轮 |
| **验证** | Code+LLM Judge | 目标完成度 | 视觉对齐 | LLM Judge | 任务完成率 |
| **训练** | Agentic RL | 经验学习 | 视频预训练 | SFT + RL | 多种RL |
| **幻觉风险** | 低(代码驱动) | 低(文件实体) | 低(像素级) | 中(LLM模拟) | 低 |

---

## Slide 15: 关键技术趋势

### Trend 1: 从"环境构建"到"环境合成"
- 人工 → 半自动 → 全自动合成pipeline
- AWM: 5步CLI命令自动生成完整环境

### Trend 2: 从"评估环境"到"训练环境"  
- Benchmark → Training Ground
- 环境不仅用于评测，更用于持续RL训练

### Trend 3: 从"固定环境"到"无限环境"
- 10个 → 1000个 → 理论上无限
- 环境多样性直接驱动Agent能力提升

### Trend 4: 从"短时序"到"超长时序"
- 单轮问答 → 多轮交互 → 2000+ turns长时序
- 模拟真实工作的复杂性

### Trend 5: 代码驱动 vs LLM模拟 的互补
- 代码驱动: 精确、可验证、无幻觉
- LLM模拟: 灵活、低成本、快速部署

---

## Slide 16: 未来展望与开放问题

### 开放问题
1. **规模化验证**: 如何在百万级环境中保证质量？
2. **Sim-to-Real Gap**: 合成环境训练的Agent能否迁移到真实世界？
3. **环境复杂度**: 如何生成需要深度推理的复杂环境？
4. **安全性**: 大规模自动合成的环境如何避免有害内容？
5. **效率**: 大规模环境生成+RL训练的计算成本优化

### 前瞻方向
- **自适应环境生成**: 根据Agent弱点动态生成针对性环境
- **多Agent协作环境**: 模拟真实世界的团队协作
- **持续进化**: 环境和Agent共同进化(co-evolution)
- **通用世界模型**: 统一的物理+数字世界模拟

---

## Slide 17: 总结

### 核心观点

> **环境是Agent能力的基石** — 没有足够丰富的环境，就没有足够强大的Agent

### 三大技术路线并存

1. **Code-Driven Synthesis** (AWM, Synthetic Computers)
   - 可靠、可验证、可扩展
   
2. **LLM-as-Simulator** (Simia)
   - 灵活、低成本、快速迭代

3. **Neural Simulation** (Neural Computers)
   - 端到端学习、面向未来

### 关键Takeaway
- 环境数量与多样性是提升Agent能力的关键杠杆
- 自动化合成pipeline极大降低了环境构建成本
- 代码驱动+LLM验证 是当前最可靠的技术组合
- 2026年是Agentic环境模拟的爆发年

---

## Slide 18: 参考文献

1. **Agent World Model** — Wang et al., 2026 (ICML 2026)
   - Paper: https://arxiv.org/abs/2602.10090
   - Code: https://github.com/Snowflake-Labs/agent-world-model

2. **Synthetic Computers at Scale** — 2026
   - Paper: https://arxiv.org/abs/2604.28181

3. **Neural Computers** — 2026
   - Paper: https://arxiv.org/abs/2604.06425

4. **Simia: Simulating Environments with Reasoning Models** — Microsoft, 2025
   - Paper: https://arxiv.org/abs/2511.01824
   - Code: https://github.com/microsoft/Simia-Agent-Training

5. **AgentGym** — ACL 2025
   - Site: https://agentgym.github.io/

6. **GEM: A Gym for Agentic LLMs** — 2025
   - Paper: https://arxiv.org/abs/2510.01051
   - Code: https://github.com/axon-rl/gem

7. **MLGym** — Meta, 2025
   - Paper: https://arxiv.org/abs/2502.14499
   - Code: https://github.com/facebookresearch/MLGym

8. **Scaling Environments for Interactive Agentic Experience Collection** — Survey, 2025
   - Paper: https://arxiv.org/abs/2511.09586

---

## 附录: 补充材料

### AWM 详细CLI命令

```bash
# 完整合成
awm gen all --target_count 1000

# 环境启动
awm env start --scenario e_commerce_33 --port 8001

# Agent交互
awm agent --task "show me top 10 products" --mcp_url http://localhost:8001/mcp

# 验证
awm verify --input outputs/agents/<timestamp> --mode sql
```

### 相关开源资源
- Awesome_Scaling_Environments: https://github.com/lukahhcm/Awesome_Scaling_Environments
- meta-pytorch/OpenEnv: https://github.com/meta-pytorch/OpenEnv
- Huggingface AWM-1K: https://huggingface.co/datasets/Snowflake/AgentWorldModel-1K
