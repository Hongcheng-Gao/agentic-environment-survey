# AI Agent 环境构建与模拟：技术全景调研

> **核心洞察**：无论是"更快地构建真实环境"还是"用模型模拟环境"，本质上要解决的是同一个问题 —— **Agent 训练和评估的环境瓶颈**。

---

## 一、问题本质

训练和评估 AI Agent 面临一个根本矛盾：

| 挑战 | 说明 |
|------|------|
| **环境稀缺** | 真实交互环境构建成本高、速度慢、覆盖窄 |
| **多样性不足** | 手工 benchmark 容易被"刷榜"，覆盖不了长尾场景 |
| **规模瓶颈** | 人工标注轨迹 + 手工编码环境无法规模化 |
| **迁移鸿沟** | 合成训练数据能否泛化到真实部署场景 |

**所有项目都在回答同一个问题：如何以低成本、大规模地为 Agent 提供可交互的训练/评估环境？**

---

## 二、技术路线分类

> **分类原则**：按**运行时产生环境状态的机制**来分类，而非按领域、构建方式或应用场景。

```
                    Agent 环境问题的解法图谱
                           │
          ┌────────────────┴────────────────┐
          │                                 │
    方向一：快速构建可执行环境            方向二：用模型模拟环境
    (Build Executable Envs)             (Model-Simulated Envs)
          │                                 │
    ┌─────┼─────┐                   ┌───────┴────────┐
    │     │     │                   │                │
   A类   B类   C类                 D类              E类
 LLM生  手工  真实              LLM运行         生成式
 成代码  编码  软件              时模拟器        世界模型
  环境  模拟  沙箱
```

### 五大类别速览

| 类别 | 核心机制 | 确定性 | 规模化 | 幻觉风险 | 代表项目 |
|------|----------|--------|--------|----------|----------|
| **A. LLM代码生成** | LLM 写代码→代码跑 | ✅ 高 | ✅ 高 | ❌ 无 | AWM, Agent-World, EnvGen |
| **B. 手工编码模拟** | 人写 Python/JS 仿真逻辑 | ✅ 高 | 中 | ❌ 无 | AppWorld, InterCode, τ-bench |
| **C. 真实软件沙箱** | 真实 OS/浏览器/代码库 in VM/Docker | ✅ 极高 | ❌ 低 | ❌ 无 | OSWorld, WebArena, SWE-bench |
| **D. LLM运行时模拟** | LLM 推理时实时生成状态转移 | ❌ 低 | ✅ 极高 | ⚠️ 高 | Simia, GenEnv, WebDreamer |
| **E. 生成式世界模型** | 神经网络预测下一帧/潜状态 | ❌ 中 | ✅ 高 | ⚠️ 中 | Genie, GameNGen, DreamerV3 |

---

## 三、A 类：LLM 代码生成环境

> LLM 在**构建时**生成可执行代码（Python/SQLite/JSON），运行时代码确定性执行。LLM 是"编译器"，不是"运行时"。

### A1. Snowflake — Agent World Model (AWM)

| 维度 | 内容 |
|------|------|
| **GitHub** | [Snowflake-Labs/agent-world-model](https://github.com/Snowflake-Labs/agent-world-model) |
| **论文** | arXiv:2602.10090 (ICML 2026) |
| **团队** | Snowflake AI Research + UNC-Chapel Hill |

**核心思路**：LLM 生成 Python 工具函数 + SQLite 数据库 schema，运行时完全确定性。用 MCP 协议暴露环境接口，支持大规模 RL 训练。

```
LLM Pipeline（构建时）→ Python 工具代码 + SQLite schema
                              ↓
              运行时：纯代码执行，无 LLM 参与
                              ↓
              MCP 协议 → Agent 交互 → 验证奖励
```

**关键数据**：
- 1,000 合成环境，每个约 35 个工具，共 10,000 任务
- 支持 1,024 并行训练实例
- BFCLv3: +12.11 points；τ²-bench: 39.03 Pass@1（发布时 SOTA）
- 产出模型：Arctic-AWM-4B / 8B / 14B

---

### A2. Agent-World (ByteDance + 人大)

| 维度 | 内容 |
|------|------|
| **主页** | [agent-tars-world.github.io](https://agent-tars-world.github.io/) |
| **论文** | arXiv:2604.18292 (2026.04) |

**核心突破**：AWM 的进化版 —— Agent 和环境**共同进化**（co-evolution）。
- LLM 挖掘真实 MCP/开源数据 → 生成 1,978 个环境、19,822 个工具
- 图结构任务生成（工具关系图随机游走）+ 程序化验证
- Agent 失败 → 诊断弱点 → 定向合成新任务 → 环境复杂化 → 循环
- 8B/14B 模型在 BFCL V4 和 23 个 benchmark 上超越 685B 模型和 GPT-5.2

---

### A3. EnvGen (COLM 2024)

| 维度 | 内容 |
|------|------|
| **论文** | arXiv:2403.12014 |
| **主页** | [envgen-llm.github.io](http://envgen-llm.github.io) |

LLM 生成 JSON 格式的环境配置（在 Crafter/Heist 游戏中）针对 Agent 弱点定向生成训练场景。
仅约 4 次 LLM 调用，比 GPT-4 直接做 agent 高 40%，使用数据量少 3.3×。

---

### A4. Eurekaverse (NVIDIA + CMU, 2024)

| 维度 | 内容 |
|------|------|
| **论文** | arXiv:2411.01775 |

LLM 生成 Python 代码定义 Isaac Gym 中的障碍跑课程（机器人跑酷），进化循环 + 性能反馈。
已在 Unitree Go1 四足机器人上完成 sim-to-real 部署，超越人工设计课程。

---

### A5. WorldCoder (Cornell, 2024)

| 维度 | 内容 |
|------|------|
| **论文** | arXiv:2402.12275 |

LLM 写 Python 程序对环境状态转移建模（LLM 写的就是 world model 代码），Agent 通过执行这个 Python 程序规划，而非每步都调用 LLM。比 deep RL 样本效率更高。

---

### A6. Synthetic Computers at Scale (2026)

| 维度 | 内容 |
|------|------|
| **论文** | arXiv:2604.28181 |

创建大规模"合成计算机"环境（含完整文件系统 + docx/xlsx/pptx 文档），双 Agent 模拟长时序生产力工作（8+ 小时 runtime，2000+ turns/run）。
Persona 可 10^9 级生成，产出丰富的经验学习信号。属于 LLM 生成内容驱动的合成环境。

---

## 四、B 类：手工编码领域模拟器

> 人工用 Python/JS 实现仿真逻辑（状态机、API 模拟、物理引擎），高度可控，但不可无限扩展。

### B1. AppWorld (ACL 2024 最佳资源论文)

| 维度 | 内容 |
|------|------|
| **GitHub** | [StonyBrookNLP/appworld](https://github.com/StonyBrookNLP/appworld) |
| **网站** | [appworld.dev](https://appworld.dev/) |

手工实现 9 个 App 的 Python API（Spotify, Venmo, Amazon 等）+ SQLite 持久化状态。
457 个 API，~100 虚拟用户，750 个任务。GPT-4o 仅完成约 49% 的"普通"任务。

---

### B2. InterCode (NeurIPS 2023)

| 维度 | 内容 |
|------|------|
| **GitHub** | [ZiyueWang25/intercode](https://github.com/ZiyueWang25/intercode) |
| **论文** | arXiv:2306.14898 |

Docker 容器内运行真实 Bash/SQL 解释器作为环境，Agent 接收真实 stdout/查询结果。
5 个环境（Bash, SQL, Python, CTF, SWE），将编程任务形式化为 POMDP。

---

### B3. WebShop (NeurIPS 2022)

| 维度 | 内容 |
|------|------|
| **GitHub** | [princeton-nlp/WebShop](https://github.com/princeton-nlp/WebShop) |
| **论文** | arXiv:2207.01206 |

程序化电商模拟器，抓取 118 万真实 Amazon 商品数据 + 自定义 web 环境 + 确定性奖励。
12,087 众包指令。人类 59%，早期最好 IL+RL 模型 29%（GPT-4 后来达 67%）。

---

### B4. ALFWorld (ICLR 2021)

| 维度 | 内容 |
|------|------|
| **GitHub** | [alfworld/alfworld](https://github.com/alfworld/alfworld) |
| **论文** | arXiv:2010.03768 |

THOR 模拟器 + 文字游戏接口，手工编码家务任务逻辑（导航、拿取、加热等）。
3,553 训练环境，6 类任务。文本训练可迁移到视觉具身任务。

---

### B5. ScienceWorld (EMNLP 2022)

| 维度 | 内容 |
|------|------|
| **GitHub** | [allenai/ScienceWorld](https://github.com/allenai/ScienceWorld) |
| **论文** | arXiv:2203.07540 |

手工编码物理/化学仿真引擎，200 个对象，25 个动作，内置热力学 + 电路仿真。
30 类任务，1,400+ 变体。GPT-4 状态预测约 60%。

---

### B6. τ-bench / τ²-bench (Sierra Research)

| 维度 | 内容 |
|------|------|
| **GitHub** | [sierra-research/tau-bench](https://github.com/sierra-research/tau-bench) |
| **论文** | arXiv:2406.12045 / arXiv:2506.07982 |

JSON 数据库作为状态 + Python API 工具操作 + LLM 模拟用户策略。
τ-bench：零售（115 任务）+ 航空（50 任务），GPT-4o <50% 成功率。
τ²-bench：扩展到电信领域，Dec-POMDP 双控制，GPT-5 在电信 96.7%。

---

### B7. MiniWob++ (OpenAI → Farama, 2017)

| 维度 | 内容 |
|------|------|
| **GitHub** | [Farama-Foundation/miniwob-plusplus](https://github.com/Farama-Foundation/miniwob-plusplus) |

100+ 手工编码 HTML/JS 微任务页面，合成 DOM，点击/输入/滚动动作。
Web Agent 基础 benchmark，高度可复现，近期基于像素的 Agent 已超越人类。

---

### B8. AgentBench (ICLR 2024)

| 维度 | 内容 |
|------|------|
| **GitHub** | [THUDM/AgentBench](https://github.com/THUDM/AgentBench) |

聚合 8 个环境（OS, DB, KG, 卡牌游戏, ALFWorld, WebShop, Web 浏览），Docker 执行 + 统一 API。
29 个 LLM 测试；GPT-4: 4.41/5，最好开源: 1.31/5，揭示商业与开源模型 3.37× 差距。

---

### B9. ToolBench / ToolLLM (ICLR 2024)

| 维度 | 内容 |
|------|------|
| **GitHub** | [OpenBMB/ToolBench](https://github.com/OpenBMB/ToolBench) |
| **论文** | arXiv:2307.16789 |

16,464 个真实 REST API（来自 RapidAPI Hub）+ DFSDT 搜索树。
49 个类别，126K 指令-解决方案对。ToolLLaMA ≈ ChatGPT 性能。

---

## 五、C 类：真实软件沙箱

> 真实软件（OS/浏览器/代码库）运行在 VM/Docker/模拟器中，没有任何仿真——Agent 与真实系统交互，ground truth 来自真实系统状态。

### C1. OSWorld (NeurIPS 2024)

| 维度 | 内容 |
|------|------|
| **GitHub** | [xlang-ai/OSWorld](https://github.com/xlang-ai/OSWorld) |
| **论文** | arXiv:2404.07972 |

真实 Ubuntu/Windows/macOS in QEMU VM，截图 + 无障碍树，PyAutoGUI 动作。
369 个任务，3 个 OS。人类 72%，GPT-4V 12%。首个 VM 级全 OS 开放任务 benchmark。

---

### C2. WebArena (ICLR 2024)

| 维度 | 内容 |
|------|------|
| **GitHub** | [web-arena-x/webarena](https://github.com/web-arena-x/webarena) |
| **论文** | arXiv:2307.13854 |

真实网站（Reddit, GitLab, OneStopShop, CMS）in Docker，函数正确性评估（检查真实数据库状态）。
812 个长时序任务。人类 78%，GPT-4 10.6%。

---

### C3. AndroidWorld (Google Research, 2024)

| 维度 | 内容 |
|------|------|
| **GitHub** | [google-research/android_world](https://github.com/google-research/android_world) |
| **论文** | arXiv:2405.14573 |

Pixel 6 模拟器内真实 Android 应用（ADB + SQLite 状态检查）+ 动态参数随机化。
116 个任务，20 个真实 App。M3A 基线 30.6%，SeeAct-V 44%。

---

### C4. SWE-bench / SWE-Gym / R2E-Gym

| 项目 | 论文 | 核心 |
|------|------|------|
| SWE-bench | arXiv:2310.06975 (ICLR 2024) | 真实 GitHub issue + 真实测试套件 in Docker；patch 能否通过 unit test 即为 oracle |
| SWE-Gym | arXiv:2412.21139 (2024) | 同 SWE-bench 但增加训练轨迹；500 轨迹 fine-tune → SWE-bench Verified +19% |
| R2E-Gym | COLM 2025 | 从 GitHub repo 自动提取测试构建 gym；8,100+ 问题，51% on SWE-bench Verified（开源 SOTA） |

---

### C5. BrowserGym (ServiceNow, 2024)

| 维度 | 内容 |
|------|------|
| **GitHub** | [ServiceNow/BrowserGym](https://github.com/ServiceNow/BrowserGym) |
| **论文** | arXiv:2412.05467 |

Playwright + Chromium 上的 gym API 层，统一 obs/action 空间，集成 MiniWob++, WebArena, VisualWebArena, WorkArena。底层运行真实浏览器。

---

### C6. WorkArena (ServiceNow, 2024)

| 维度 | 内容 |
|------|------|
| **GitHub** | [ServiceNow/WorkArena](https://github.com/ServiceNow/WorkArena) |
| **论文** | arXiv:2403.07718 |

通过浏览器自动化访问真实 ServiceNow 企业 SaaS 实例。
33 个任务，19,912 个唯一实例。简单任务约 75%，复杂工作流 9–19% vs 人类 97.9%。

---

### C7. Spider2-V (XLANG Lab, 2024)

| 维度 | 内容 |
|------|------|
| **GitHub** | [xlang-ai/spider2](https://github.com/xlang-ai/spider2) |
| **论文** | arXiv:2407.10956 |

真实数据工程工具（BigQuery, dbt, JupyterLab, Metabase）in cloud/VM。
494 个任务，20 个企业应用。SOTA VLM 14% 成功率。

---

### C8. MCP-Universe (Salesforce AI, 2025)

| 维度 | 内容 |
|------|------|
| **GitHub** | [SalesforceAIResearch/MCP-Universe](https://github.com/SalesforceAIResearch/MCP-Universe) |

11 个真实 MCP server（GitHub, Google Maps, Blender, Playwright, Yahoo Finance 等）。
231 个任务，6 个领域。GPT-5 43.7%，Claude 4 Sonnet 29.4%。评估器检查实时数据。

---

## 六、D 类：LLM 运行时模拟器

> LLM 在**推理时**充当环境，实时生成状态转移描述和工具响应。零工程成本，但有幻觉风险，长时序保真度下降。

### D1. Simia (Microsoft Research, 2025)

| 维度 | 内容 |
|------|------|
| **GitHub** | [microsoft/Simia-Agent-Training](https://github.com/microsoft/Simia-Agent-Training) |
| **论文** | arXiv:2511.01824 |

LLM（GPT-4o/o4-mini）接收 Agent 动作输出模拟状态转移 + 奖励。
Simia-SFT：1.5K 种子轨迹 → 合成 90K 训练数据（60× 扩增）。
Simia-RL：LLM 模拟环境 + GRPO 训练 Agent。
Qwen2.5-32B 在 τ²-bench 达 58.9（vs GPT-4o 54.2），开源模型比肩前沿。

---

### D2. GenEnv / Co-Evo (Gen-Verse, 2025)

| 维度 | 内容 |
|------|------|
| **GitHub** | [Gen-Verse/GenEnv](https://github.com/Gen-Verse/GenEnv) |
| **论文** | arXiv:2512.19682 |

Agent 策略 + 环境策略**共同进化**，α-课程奖励将难度对齐至 Agent 当前技能水平（近端发展区）。
比基线高 40.3%（API-Bank/ALFWorld/TravelPlanner），比 Gemini 2.5 Pro 离线数据少 3.3×。

---

### D3. WebDreamer (OSU NLP, TMLR 2025)

| 维度 | 内容 |
|------|------|
| **GitHub** | [OSU-NLP-Group/WebDreamer](https://github.com/OSU-NLP-Group/WebDreamer) |
| **论文** | arXiv:2411.06559 |

LLM 模拟候选动作后的网页状态，评分函数选最优动作再真实执行（"行动前先做梦"）。
VisualWebArena +33%，Mind2Web-Live +13%。

---

### D4. WMA Agents (ICLR 2025)

| 维度 | 内容 |
|------|------|
| **GitHub** | [Listever/WMA-Agents](https://github.com/Listever/WMA-Agents) |

轻量级 web 世界模型：LLM 预测"关键变化描述"（如"购物车已更新"）而非完整 HTML 重建。
Mind2Web-Live +23.8%，计算开销更低。

---

## 七、E 类：生成式世界模型

> 神经网络（扩散模型、视频 Transformer、RSSM）预测下一视觉帧或潜状态。面向具身/视觉 Agent，输出为像素或潜向量而非文本。

### E1. Genie 1 (Google DeepMind, 2024)

| 维度 | 内容 |
|------|------|
| **论文** | arXiv:2402.15391 |

时空视频 tokenizer + 潜动作模型 + MaskGIT 动态模型，从 3 万小时 2D 平台游戏视频训练，无监督推断潜动作。
11B 参数，8 个潜动作编码，16fps @ 160×90，从单张图像生成可交互游戏世界。

---

### E2. Genie 2 (Google DeepMind, Dec 2024)

| 维度 | 内容 |
|------|------|
| **主页** | [deepmind.google/genie-2](https://deepmind.google/discover/blog/genie-2-a-large-scale-foundation-world-model/) |

自回归潜扩散 + Transformer，单张图像 → 可导航 3D 环境，涌现物理（重力、碰撞、NPC 行为）。
画质接近 AAA 游戏，场景一致性约 1 分钟，专为 SIMA Agent 训练设计。

---

### E3. GameNGen (Google Research, 2024)

| 维度 | 内容 |
|------|------|
| **论文** | arXiv:2408.14837 |

Stable Diffusion v1.4 微调逐帧预测（以游戏动作为条件）。RL Agent 收集约 9 亿训练帧。
单 TPU 20 FPS，PSNR 29.4，人类无法区分与真实 DOOM 差异。首个完全由神经网络驱动的游戏引擎。

---

### E4. UniSim (UC Berkeley + DeepMind + MIT, ICLR 2024 Outstanding)

| 维度 | 内容 |
|------|------|
| **论文** | arXiv:2310.06114 |

视频扩散模型统一文本-图像数据、机器人控制、人类活动视频。T5 编码文本动作，归一化嵌入编码电机指令。
支持 8+ 连续动作，机器人零样本 sim-to-real 迁移。

---

### E5. DreamerV3 (Google DeepMind, 2023)

| 维度 | 内容 |
|------|------|
| **GitHub** | [danijar/dreamerv3](https://github.com/danijar/dreamerv3) |
| **论文** | arXiv:2301.04104 |

RSSM 潜状态空间模型：编码器 → 潜动态 → 解码器，actor-critic 在**想象**中规划。
固定超参数在所有领域通用。首个 AI 从零开始收集 Minecraft 钻石。Atari SOTA。

---

### E6. Neural Computers (2026)

| 维度 | 内容 |
|------|------|
| **论文** | arXiv:2604.06425 |

提出"神经计算机"概念：将传统计算机的计算、内存、I/O 统一在视频模型的学习状态中。
从指令+像素+用户动作 rollout 屏幕帧，在 CLI 和 GUI 上测试。
长期目标：Completely Neural Computer（CNC）—— 稳定执行 + 显式重编程 + 能力持久复用。

---

## 八、平台/聚合框架

> 本身不创建或模拟环境，而是提供统一的 gym 接口**包装**上述各类环境，降低使用门槛、支持 RL 训练。

| 项目 | 组织 | 年份 | 包装的环境 | 特点 |
|------|------|------|------------|------|
| **AgentGym** | 复旦 + 字节 | ACL 2025 | 14 个环境（WebShop, ALFWorld, SciWorld…） | AgentEvol 跨环境自演化；AgentTraj-L 6,130 条轨迹；7B 模型比肩 GPT-4 |
| **GEM** | axon-rl | 2024 | 24 个环境（语言游戏、数学、编程、QA） | OpenAI Gym 风格 API；异步向量化执行；兼容 PPO/REINFORCE/GRPO |
| **MLGym** | Meta | 2025 | 13 个 AI 研究任务（CV/NLP 等） | 开放式任务设计；评估 LLM 自动化科研能力 |
| **BALROG** | UCL+NYU+Oxford | 2024 | NetHack, BabyAI, Crafter… | 长时序游戏任务；VLM 比文本 LLM 表现更差（反直觉发现） |
| **Cradle** | BAAI+NTU+PKU | 2024 | 任何软件（像素+鼠标/键盘） | 无需 API；通用计算机控制框架；完成 40 分钟 RDR2 任务 |

---

## 九、技术路线深度对比

| 维度 | A LLM代码生成 | B 手工编码模拟 | C 真实软件沙箱 | D LLM运行时模拟 | E 生成式世界模型 |
|------|--------------|--------------|----------------|----------------|----------------|
| **运行时 oracle** | Python/SQLite | 手写 Python | 真实系统状态 | LLM 推理 | 神经网络预测 |
| **确定性** | ✅ 极高 | ✅ 极高 | ✅ 极高 | ❌ 低 | ⚠️ 中 |
| **规模化** | ✅ 极高 | 中 | ❌ 低 | ✅ 无限 | ✅ 高 |
| **幻觉风险** | ❌ 无 | ❌ 无 | ❌ 无 | ⚠️ 高 | ⚠️ 中 |
| **真实度** | 中-高 | 中 | ✅✅ 最高 | 中 | 中-高 |
| **构建成本** | 低（LLM 生成） | 高（人工） | 极高（基础设施） | 极低（换 prompt） | 极高（视频预训练） |
| **RL 训练友好** | ✅✅ | ✅ | ⚠️ | ⚠️ | ⚠️ |
| **最佳用途** | 大规模 tool-use RL | 受控 benchmark | 最终部署评估 | 快速原型/覆盖长尾 | 具身机器人/游戏 |

---

## 十、关键趋势

1. **A 类（LLM 代码生成）正在成为 RL 训练的主流**：确定性、可并行、无幻觉，AWM + Agent-World 验证了路线正确性
2. **自演化是前沿方向**：Agent-World 和 GenEnv 引入 co-evolution，类比 AlphaGo self-play
3. **MCP 成为 A 类环境的事实接口标准**：AWM、Agent-World 均以 MCP 暴露工具
4. **小模型 + 大环境多样性 > 大模型**：Agent-World 8B 超越 685B 和 GPT-5.2
5. **C 类（真实环境）表现仍远低于人类**：AppWorld ~49%，MCP-Universe ~43.7%，空间巨大
6. **D 类和 A 类互补**：D 类冷启动/快速迭代，A 类稳定 RL 训练；WebDreamer/WMA 展示 D 类用于 lookahead planning 的特殊价值

---

## 十一、参考资料

| 项目 | 论文链接 |
|------|------|
| AWM (Snowflake) | [arXiv:2602.10090](https://arxiv.org/abs/2602.10090) |
| Agent-World (ByteDance) | [arXiv:2604.18292](https://arxiv.org/abs/2604.18292) |
| EnvGen | [arXiv:2403.12014](https://arxiv.org/abs/2403.12014) |
| Eurekaverse | [arXiv:2411.01775](https://arxiv.org/abs/2411.01775) |
| WorldCoder | [arXiv:2402.12275](https://arxiv.org/abs/2402.12275) |
| Synthetic Computers | [arXiv:2604.28181](https://arxiv.org/abs/2604.28181) |
| AppWorld | [ACL 2024](https://appworld.dev/) |
| InterCode | [arXiv:2306.14898](https://arxiv.org/abs/2306.14898) |
| WebShop | [arXiv:2207.01206](https://arxiv.org/abs/2207.01206) |
| ALFWorld | [arXiv:2010.03768](https://arxiv.org/abs/2010.03768) |
| ScienceWorld | [arXiv:2203.07540](https://arxiv.org/abs/2203.07540) |
| τ-bench / τ²-bench | [arXiv:2406.12045](https://arxiv.org/abs/2406.12045) |
| ToolBench | [arXiv:2307.16789](https://arxiv.org/abs/2307.16789) |
| OSWorld | [arXiv:2404.07972](https://arxiv.org/abs/2404.07972) |
| WebArena | [arXiv:2307.13854](https://arxiv.org/abs/2307.13854) |
| AndroidWorld | [arXiv:2405.14573](https://arxiv.org/abs/2405.14573) |
| SWE-bench | [arXiv:2310.06975](https://arxiv.org/abs/2310.06975) |
| SWE-Gym | [arXiv:2412.21139](https://arxiv.org/abs/2412.21139) |
| R2E-Gym | [GitHub](https://github.com/R2E-Gym/R2E-Gym) |
| BrowserGym | [arXiv:2412.05467](https://arxiv.org/abs/2412.05467) |
| WorkArena | [arXiv:2403.07718](https://arxiv.org/abs/2403.07718) |
| Spider2-V | [arXiv:2407.10956](https://arxiv.org/abs/2407.10956) |
| MCP-Universe | [GitHub](https://github.com/SalesforceAIResearch/MCP-Universe) |
| Simia (Microsoft) | [arXiv:2511.01824](https://arxiv.org/abs/2511.01824) |
| GenEnv | [arXiv:2512.19682](https://arxiv.org/abs/2512.19682) |
| WebDreamer | [arXiv:2411.06559](https://arxiv.org/abs/2411.06559) |
| WMA Agents | [GitHub](https://github.com/Listever/WMA-Agents) |
| Genie 1 | [arXiv:2402.15391](https://arxiv.org/abs/2402.15391) |
| Genie 2 | [DeepMind Blog](https://deepmind.google/discover/blog/genie-2-a-large-scale-foundation-world-model/) |
| GameNGen | [arXiv:2408.14837](https://arxiv.org/abs/2408.14837) |
| UniSim | [arXiv:2310.06114](https://arxiv.org/abs/2310.06114) |
| DreamerV3 | [arXiv:2301.04104](https://arxiv.org/abs/2301.04104) |
| Neural Computers | [arXiv:2604.06425](https://arxiv.org/abs/2604.06425) |
| AgentGym | [GitHub](https://github.com/WooooDyy/AgentGym) |
| GEM | [arXiv:2510.01051](https://arxiv.org/abs/2510.01051) |
| MLGym | [arXiv:2502.14499](https://arxiv.org/abs/2502.14499) |
| BALROG | [arXiv:2411.13543](https://arxiv.org/abs/2411.13543) |
| Cradle | [arXiv:2403.03186](https://arxiv.org/abs/2403.03186) |

---

*调研时间：2026-05-14*
