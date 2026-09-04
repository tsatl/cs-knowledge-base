# Agent 学习框架：MCTA

> 本文用于组织 Agent 相关知识，作为后续持续扩展的总纲与目录。
>
> MCTA 并不是唯一的 Agent 分类标准，而是一套适合学习、整理和自进化研究的工作框架：
>
> **M — Model，C — Context，T — Tools，A — Architecture**

---

## 一、MCTA 总体思想

一个 Agent 的能力，不只是由底层大模型决定。

更完整地看，可以把 Agent 拆成四个相互配合的部分：

```text
M — Model
模型
→ Agent 本身“会什么”

C — Context
上下文
→ Agent 当前“知道什么、被要求怎么做”

T — Tools
工具
→ Agent 能调用哪些外部能力

A — Architecture
架构
→ Model、Context、Tools 如何被组织成完整工作流
```

可以用一句话概括：

> **Model 决定能力上限，Context 决定当前已知信息和行为指令，Tools 扩展外部行动能力，Architecture 决定模型、上下文和工具如何被组织成完整的 Agent 工作流。**

可以把一个 Agent 简化表示为：

```text
Agent = Model + Context + Tools + Architecture
```

四者不是完全独立的：

```text
Model
  │
  │ 使用 Context 推理
  ↓
Context
  │
  │ 在 Architecture 中被组织
  ↓
Architecture
  │
  ├── 调用 Model
  ├── 读取 / 更新 Context
  └── 调用 Tools
           ↓
        Environment
           ↓
        Feedback
           ↓
     下一轮 Agent 行为
```

因此，MCTA 可以同时作为：

```text
Agent 的知识组织框架
Agent 的系统拆分框架
Self-Evolving Agent 的可进化对象框架
```

后续学习时，可以不断问四个问题：

```text
M：模型本身具备什么能力？
C：模型当前拿到了什么信息和指令？
T：Agent 能调用什么外部能力？
A：这些能力如何被组织、调度和循环执行？
```

---

# M — Model

## 1. 定义

Model 是 Agent 的基础模型层。

它提供最底层的：

```text
语言理解
知识能力
代码能力
逻辑推理
数学推理
多模态理解
生成能力
指令遵循能力
```

例如：

```text
LLM
VLM
Reasoning Model
Multimodal Model
```

可以简单理解为：

> **Model 决定 Agent 的基础能力和能力上限。**

---

## 2. Model 层主要学习什么

### 大模型基础

```text
Transformer
Attention
Token
Embedding
Context Window
Pretraining
Inference
Sampling
```

### 模型能力

```text
语言理解
知识问答
代码生成
数学推理
长文本理解
多模态能力
Function Calling
Structured Output
```

### 推理模型

重点关注：

```text
Reasoning Model
Test-Time Compute
Long Reasoning
Self-Consistency
Verifier
Reward Model
```

注意区分：

```text
模型原生是否具备较强推理能力
→ Model

如何组织推理过程
→ 更偏 Architecture

通过什么提示方式诱导推理
→ 更偏 Context
```

---

## 3. Model 的训练与优化

常见方法：

```text
Pretraining
SFT
Instruction Tuning
RLHF
DPO
PPO
GRPO
Distillation
Continual Learning
Self-Training
Fine-tuning
LoRA
```

如果 Agent 的进化直接修改模型参数：

```text
Model₀
  ↓
训练 / 强化学习 / 蒸馏
  ↓
Model₁
```

可以理解为：

```text
Model Evolution
```

这一层通常：

```text
修改最深
成本最高
能力变化最根本
```

---

## 4. Model 层核心问题

```text
模型为什么具备推理能力？
普通 LLM 和 Reasoning Model 有什么区别？
Context Window 如何限制 Agent？
模型参数越大是否一定越适合 Agent？
如何选择不同任务对应的模型？
什么时候需要微调，什么时候只需要 Prompt？
模型能力如何评测？
模型如何进行多模态输入输出？
Model Routing 是什么？
```

---

## 5. Model 层建议子目录

```text
model/
├── llm-basics.md
├── transformer.md
├── inference.md
├── reasoning-model.md
├── multimodal-model.md
├── fine-tuning.md
├── rl-for-llm.md
├── model-routing.md
└── model-evaluation.md
```

---

# C — Context

## 1. 定义

Context 是模型在当前一次 Agent 运行中能够看到和使用的信息。

可以理解为：

> **Context 决定 Agent 当前知道什么，以及当前应该如何行为。**

Context 可以进一步拆成：

```text
Context
├── Prompt
├── Memory
├── Conversation History
├── Retrieved Knowledge
├── Current Observation
└── Task State
```

其中最核心的两个部分通常是：

```text
Prompt
+
Memory
```

---

## 2. Prompt

Prompt 不只是用户输入的一句话。

在 Agent 中，广义 Prompt 可以包含：

```text
System Prompt
Developer / Policy Instructions
User Prompt
Task Prompt
Role Definition
Few-shot Examples
Output Format
Constraints
Reasoning Instructions
Tool Instructions
```

Prompt 的作用主要是：

```text
定义角色
定义目标
定义约束
定义行为规则
定义输出格式
定义推理方式
```

例如：

```text
“回答问题”
```

和：

```text
“先分析任务，再拆解步骤；
需要外部信息时调用工具；
执行后验证结果；
失败时反思并重新尝试。”
```

即使 Model 相同，Agent 行为也可能完全不同。

---

## 3. Prompt Engineering

可以继续学习：

```text
Zero-shot
Few-shot
Chain-of-Thought
Role Prompting
Structured Prompt
Prompt Template
System Prompt Design
Prompt Compression
Prompt Optimization
Automatic Prompt Optimization
```

注意：

```text
CoT 的提示文本
→ Context

真正把推理拆成多个阶段和循环
→ Architecture
```

---

## 4. Memory

Memory 用于让 Agent 不只依赖当前一轮输入。

常见分类：

```text
Short-Term Memory
Long-Term Memory
Working Memory
Episodic Memory
Semantic Memory
Procedural Memory
```

可以简单理解：

```text
短期记忆
→ 当前任务和当前会话

长期记忆
→ 跨任务保存的信息

情景记忆
→ 过去发生过什么

语义记忆
→ 已经沉淀出的知识

程序性记忆
→ 如何完成某类任务
```

---

## 5. Memory 的基本过程

```text
产生信息
   ↓
Memory Write
   ↓
Memory Store
   ↓
Memory Retrieve
   ↓
Relevant Memory
   ↓
加入 Context
   ↓
Model 推理
```

Memory 本身属于 Context。

但下面这些问题更偏 Architecture：

```text
什么时候写入 Memory？
什么时候检索？
检索多少条？
如何更新？
什么时候遗忘？
如何避免错误记忆长期传播？
```

因此可以记：

```text
Memory Content
→ Context

Memory Workflow
→ Architecture
```

---

## 6. RAG 与 Context

RAG 本质上也是一种 Context 构建机制：

```text
User Query
   ↓
Retrieve
   ↓
Relevant Documents
   ↓
加入 Context
   ↓
Model
```

重点：

```text
Embedding
Vector Database
Chunking
Retrieval
Reranking
Hybrid Search
Context Compression
Agentic RAG
Knowledge Graph RAG
```

其中：

```text
检索到什么内容
→ Context

如何决定什么时候检索、是否继续检索
→ Architecture
```

---

## 7. Context 层核心问题

```text
Prompt 和 Context 有什么区别？
Memory 和 RAG 有什么区别？
短期记忆和长期记忆怎么设计？
Agent 为什么需要 Memory？
如何避免 Context 太长？
如何做 Context Compression？
错误 Memory 会不会导致幻觉级联？
什么时候应该写入长期记忆？
如何进行 Memory Retrieval？
Prompt 是否可以自动进化？
```

---

## 8. Context 层建议子目录

```text
context/
├── prompt.md
├── prompt-engineering.md
├── memory.md
├── long-term-memory.md
├── context-management.md
├── rag.md
├── agentic-rag.md
└── context-evolution.md
```

---

# T — Tools

## 1. 定义

Tools 是 Agent 可以调用的外部能力。

如果只有 Model：

```text
Model
→ 主要负责理解、推理、生成
```

加入 Tools 后：

```text
Agent
→ 可以真正查询、计算、执行和操作外部世界
```

常见工具：

```text
Search
Browser
Calculator
Python
Shell
Database
Code Executor
REST API
GitHub
Email
Calendar
MCP Server
Computer Use
```

因此：

> **Tools 决定 Agent 能把内部推理转化成哪些外部行动。**

---

## 2. Tool Calling

基本流程：

```text
User Task
   ↓
Model 判断是否需要工具
   ↓
选择 Tool
   ↓
生成 Tool Arguments
   ↓
执行 Tool
   ↓
返回 Observation
   ↓
Model 继续推理
```

这是 Agent 最核心的循环之一：

```text
Reason
  ↓
Act
  ↓
Observe
  ↓
Reason
```

---

## 3. Tool 层主要学习什么

### Tool Definition

```text
工具名称
工具描述
参数 Schema
返回值
错误信息
```

### Tool Selection

```text
什么时候应该调用工具？
应该选择哪个工具？
是否应该同时调用多个工具？
```

### Tool Composition

```text
Search
  ↓
Browser
  ↓
Extract
  ↓
Python
  ↓
Report
```

多个工具可以组合成：

```text
Tool Chain
Workflow
SOP
Skill
```

---

## 4. MCP

MCP 可以理解为：

```text
Agent / Model
     ↓
统一协议
     ↓
External Tools / Resources
```

重点可学习：

```text
MCP Client
MCP Server
Tools
Resources
Prompts
Transport
Permissions
```

它解决的是：

```text
工具如何标准化接入 Agent
```

---

## 5. Tool Evolution

Tool 不只是静态工具集合。

更高级的 Agent 可以：

```text
发现工具
选择工具
组合工具
生成工具
修改工具
评测工具
淘汰工具
```

例如：

```text
多个重复步骤
   ↓
Agent 发现固定模式
   ↓
生成新的 Skill / Tool
   ↓
以后直接复用
```

这属于：

```text
Tool Evolution
```

---

## 6. Tool 层核心问题

```text
Tool Calling 是如何实现的？
Function Calling 和 Agent 有什么区别？
Tool Schema 为什么重要？
工具调用失败怎么处理？
Tool Selection 如何做？
如何减少无意义工具调用？
MCP 解决了什么问题？
Skill 和 Tool 有什么区别？
Agent 能否自己生成工具？
Browser / Code / Search Tool 如何组合？
工具权限和安全如何控制？
```

---

## 7. Tools 层建议子目录

```text
tools/
├── tool-calling.md
├── function-calling.md
├── tool-selection.md
├── tool-composition.md
├── mcp.md
├── skills.md
├── browser-agent.md
├── code-agent.md
└── tool-evolution.md
```

---

# A — Architecture

## 1. 定义

Architecture 是 MCTA 中负责组织其他组件的一层。

它决定：

> **Model、Context 和 Tools 如何被连接、调度、循环和协同。**

可以简单理解为：

```text
Agent 的工作机制
+
控制流
+
工作流
+
系统拓扑
```

如果：

```text
M = 大脑能力
C = 当前脑中的信息
T = 手脚
```

那么：

```text
A = 整个 Agent 如何思考和行动的工作机制
```

---

## 2. Architecture 包含什么

常见内容：

```text
Reasoning Workflow
Planning
Task Decomposition
Routing
Tool-Calling Loop
Reflection
Verification
Critic
Retry
State Machine
Workflow
Multi-Agent
Agent Communication
Agent Topology
Termination Condition
```

Architecture 重点解决：

```text
先做什么？
后做什么？
是否需要拆任务？
什么时候调用工具？
什么时候验证？
失败后是否重试？
是否需要反思？
什么时候结束？
多个 Agent 如何通信？
```

---

## 3. Reasoning 与 Architecture

Reasoning 不能简单等同于 Architecture。

更准确地说：

```text
模型本身会不会推理
→ Model

通过 Prompt 告诉模型怎么推理
→ Context

将推理拆成多个阶段、节点和循环
→ Architecture
```

例如：

```text
“请一步一步思考”
→ Context

Planner
  ↓
Reasoner
  ↓
Tool
  ↓
Verifier
  ↓
Reflection
  ↓
重新规划
→ Architecture
```

因此在 MCTA 中：

> **Architecture 更关注 Reasoning Workflow，而不是模型原生 Reasoning Capability。**

---

## 4. 常见 Agent Architecture

### ReAct

```text
Thought
  ↓
Action
  ↓
Observation
  ↓
Thought
```

核心：

```text
推理
+
工具调用
+
环境反馈
```

### Plan-and-Execute

```text
Task
 ↓
Planner
 ↓
Plan
 ↓
Executor
 ↓
Step 1
 ↓
Step 2
 ↓
...
```

特点：

```text
先规划
再执行
```

### Reflection

```text
Execute
  ↓
Result
  ↓
Evaluate
  ↓
Reflect
  ↓
Revise
  ↓
Execute Again
```

核心：

```text
增加反馈闭环
```

### Verifier / Critic

```text
Generator
   ↓
Candidate
   ↓
Verifier / Critic
   ↓
Accept / Revise
```

主要用于：

```text
降低错误
减少幻觉
提高推理可靠性
```

---

## 5. Multi-Agent Architecture

常见组织方式：

### Pipeline

```text
Agent A
  ↓
Agent B
  ↓
Agent C
```

### Manager-Worker

```text
        Manager
      /    |         ↓     ↓     ↓
Worker A Worker B Worker C
```

### Debate

```text
Agent A ──┐
          ↓
Agent B → Judge
          ↑
Agent C ──┘
```

### Network

```text
Agent A ↔ Agent B
  ↕          ↕
Agent C ↔ Agent D
```

Multi-Agent Architecture 重点研究：

```text
角色
通信
路由
任务分配
共享 Memory
协作
竞争
投票
Judge
共识
终止机制
```

---

## 6. Architecture 与幻觉级联

复杂 Agent 系统中：

```text
早期错误
  ↓
后续节点继承
  ↓
继续推理
  ↓
继续调用工具
  ↓
错误放大
```

可能形成：

```text
Hallucination Cascade
幻觉级联
```

Architecture 层可以通过：

```text
Verifier
Cross-check
Independent Judge
Reflection
Source Validation
Confidence Gate
```

降低这种问题。

---

## 7. Architecture Evolution

Architecture 本身也可以进化。

例如：

```text
旧结构：

Agent
 ↓
Tool
 ↓
Answer
```

发现错误率高后：

```text
新结构：

Agent
 ↓
Tool
 ↓
Verifier
 ↓
Reflection
 ↓
Answer
```

或者：

```text
Single Agent
```

演化成：

```text
Planner
  ↓
Executor
  ↓
Reviewer
```

因此 Architecture Evolution 可以包括：

```text
Workflow Evolution
Routing Evolution
Reasoning Workflow Evolution
Agent Role Evolution
Multi-Agent Topology Evolution
```

---

## 8. Architecture 层核心问题

```text
Agent 和 Workflow 有什么区别？
ReAct 是什么？
Planning 为什么重要？
Reflection 是否真的有效？
Verifier 如何设计？
如何实现任务分解？
Single-Agent 和 Multi-Agent 如何选择？
Agent 之间如何通信？
如何避免幻觉级联？
如何设置 Retry 和 Termination？
什么时候需要 Human-in-the-loop？
Architecture 是否可以自动进化？
```

---

## 9. Architecture 层建议子目录

```text
architecture/
├── agent-basics.md
├── reasoning.md
├── react.md
├── planning.md
├── reflection.md
├── verifier.md
├── workflow.md
├── multi-agent.md
├── agent-communication.md
├── hallucination-cascade.md
└── architecture-evolution.md
```

---

# 总结

MCTA 可以作为一套长期维护的 Agent 学习框架：

```text
M — Model
→ 基础能力
→ 模型本身会什么

C — Context
→ Prompt + Memory + 当前信息
→ 当前知道什么、被要求怎么做

T — Tools
→ 外部能力
→ 能调用什么、能执行什么

A — Architecture
→ 控制流与系统组织方式
→ Model、Context、Tools 如何协同
```

最终可以记成：

> **Model 决定能力上限，Context 决定当前已知信息和行为指令，Tools 扩展外部行动能力，Architecture 决定模型、上下文和工具如何被组织成完整的 Agent 工作流。**

从 Self-Evolving Agent 的角度：

```text
Agent Execution
      ↓
Environment
      ↓
Feedback
      ↓
Evaluation
      ↓
Optimization
      ↓
┌─────┬─────┬─────┬─────┐
↓     ↓     ↓     ↓
M     C     T     A
```

即：

```text
Model Evolution
Context Evolution
Tool Evolution
Architecture Evolution
```

---

# 推荐的笔记目录

建议把本文件作为 Agent 知识总纲：

```text
ai/
└── agent/
    ├── README.md
    │
    ├── model/
    │   ├── llm-basics.md
    │   ├── reasoning-model.md
    │   └── model-evolution.md
    │
    ├── context/
    │   ├── prompt.md
    │   ├── memory.md
    │   ├── rag.md
    │   └── context-evolution.md
    │
    ├── tools/
    │   ├── tool-calling.md
    │   ├── mcp.md
    │   ├── skills.md
    │   └── tool-evolution.md
    │
    └── architecture/
        ├── reasoning.md
        ├── planning.md
        ├── react.md
        ├── reflection.md
        ├── verifier.md
        ├── multi-agent.md
        ├── hallucination-cascade.md
        └── architecture-evolution.md
```

其中：

```text
README.md
→ 只维护 MCTA 总纲和导航

具体知识
→ 放到四个子目录继续扩展
```

这样后续学习任何 Agent 概念时，只需要先判断：

```text
它主要改变的是 Model？
Context？
Tools？
还是 Architecture？
```

再放入对应目录即可。
