# A — Architecture

> **Architecture 决定 Model、Context、Tools 如何被组织成完整 Agent。**

Architecture 主要解决：

```text
怎么开始任务
怎么规划
什么时候调用 Tool
什么时候读取 Memory
如何处理失败
是否需要反思
如何验证结果
什么时候结束
多个 Agent 如何协作
```

核心主线：

```text
Observation
   ↓
Reasoning / Planning
   ↓
Action / Tool Call
   ↓
Observation
   ↓
Reflection / Verification
   ↓
继续 or 结束
```

---

# Agent 基本架构

## Agent Loop

最基础的 Agent Loop：

```text
Observe
  ↓
Think
  ↓
Act
  ↓
Observe
  ↓
...
```

可以写成：

```text
环境 / 用户输入
      ↓
   Observation
      ↓
     Model
      ↓
Reasoning / Decision
      ↓
 Action / Tool Call
      ↓
 Tool Result / Environment
      ↓
   新 Observation
```

---

## Observation

Observation 是：

```text
Agent 当前看到的环境反馈
```

例如：

```text
用户消息
网页结果
工具返回
文件内容
API 结果
错误信息
```

Observation 最终进入：

```text
Context
```

---

## Reasoning

Reasoning：

```text
模型根据当前 Context
决定下一步该做什么
```

包括：

```text
理解问题
任务拆解
选择工具
判断结果
决定是否继续
```

需要区分：

```text
模型本身的推理能力
→ Model

多阶段推理流程
→ Architecture
```

---

## Action

Action：

```text
Agent 对外部世界执行的动作
```

例如：

```text
调用 Search
读文件
运行代码
调用 API
发起子 Agent
```

---

## Feedback

执行后得到：

```text
Observation / Result
```

然后：

```text
加入 Context
   ↓
重新推理
```

形成闭环。

---

# 经典 Agent 范式

## ReAct

ReAct：

```text
Reasoning
+
Acting
```

经典流程：

```text
Thought
  ↓
Action
  ↓
Observation
  ↓
Thought
  ↓
...
```

核心思想：

> **边思考、边行动、边根据环境反馈调整。**

优点：

```text
动态适应
能利用工具
能够根据结果纠错
```

缺点：

```text
调用次数多
成本较高
可能循环
```

适合：

```text
搜索
API
实时信息
复杂交互任务
```

---

## Plan-and-Solve

流程：

```text
Plan
 ↓
Solve
```

核心：

```text
先规划完整步骤
再逐步执行
```

优点：

```text
结构清晰
任务拆解稳定
```

缺点：

```text
初始计划错误
可能影响整个任务
动态性较弱
```

适合：

```text
数学推理
明确流程
结构化复杂任务
```

---

## Reflection

流程：

```text
Execution
   ↓
Reflection
   ↓
Refinement
   ↓
再次执行 / 输出
```

核心：

```text
先做
再检查
再修改
```

适合：

```text
代码
写作
分析
高质量输出
```

缺点：

```text
成本高
延迟高
可能过度反思
```

---

## Plan-and-Execute

可以理解：

```text
Planner
   ↓
生成计划
   ↓
Executor
   ↓
逐项执行
```

相比 Plan-and-Solve：

```text
更强调
规划模块与执行模块分离
```

---

## Verifier / Critic

Verifier：

```text
检查结果是否正确
```

Critic：

```text
指出问题并给出改进意见
```

常见结构：

```text
Generator
   ↓
Result
   ↓
Verifier / Critic
   ↓
Pass ?
├── Yes → 输出
└── No  → 修改
```

---

# Agent 工作流

## Planning

Planning：

```text
把复杂任务拆成多个步骤
```

例如：

```text
目标
 ↓
子任务 1
 ↓
子任务 2
 ↓
子任务 3
 ↓
最终结果
```

---

## Routing

Routing：

```text
根据任务类型
选择不同模型 / Tool / Skill / Agent
```

例如：

```text
代码问题
→ Code Agent

检索问题
→ Search Agent

文件问题
→ File Tool
```

---

## Retry

Tool 调用可能失败：

```text
超时
参数错误
服务异常
结果为空
```

Architecture 需要定义：

```text
是否重试
重试几次
是否换工具
是否降级
是否停止
```

---

## Reflection

Architecture 层的 Reflection 是：

```text
主动安排“执行后检查”
```

而不是简单依赖模型一次完成。

---

## Verification

Verification：

```text
用外部规则
另一个模型
测试
计算器
搜索结果
```

验证输出。

目标：

```text
降低错误率
减少幻觉
```

---

## Termination

Agent 必须有停止条件。

例如：

```text
任务完成
达到最大步骤
达到预算限制
连续失败
用户取消
```

否则可能：

```text
无限循环
```

---

# Memory Architecture

## MemoryManager

Memory 内容属于：

```text
Context
```

MemoryTool 属于：

```text
Tools
```

MemoryManager 则属于：

```text
Architecture
```

因为它负责：

```text
什么时候写
什么时候读
什么时候忘
什么时候整合
```

---

## Memory Retrieval

检索策略可以考虑：

```text
语义相关性
时间
重要性
任务类型
```

流程：

```text
当前任务
 ↓
Memory Search
 ↓
候选记忆
 ↓
筛选
 ↓
加入 Context
```

---

## Memory Write

并不是所有内容都值得保存。

可以根据：

```text
重要性
长期价值
重复频率
用户显式要求
```

决定是否写入。

---

## Forget

忘记机制：

```text
删除过期内容
删除低价值内容
删除重复内容
修正错误内容
```

避免：

```text
Memory 无限膨胀
```

---

## Consolidation

将多条零散信息：

```text
总结
合并
去重
```

形成：

```text
长期稳定知识
```

---

# Multi-Agent

## Sub-Agent

复杂任务可以拆给多个子 Agent：

```text
Main Agent
   │
   ├── Sub-Agent A
   ├── Sub-Agent B
   └── Sub-Agent C
          ↓
       各自完成
          ↓
       汇总结果
          ↓
      Main Agent
```

优势：

```text
任务并行
上下文隔离
专业分工
```

缺点：

```text
成本增加
协调复杂
结果可能冲突
```

---

## Role 分工

常见角色：

```text
Planner
Researcher
Coder
Reviewer
Verifier
Executor
```

重点：

```text
不同 Agent
负责不同职责
```

---

## Agent Communication

Multi-Agent 系统需要解决：

```text
Agent 之间怎么发现
怎么发消息
怎么共享状态
怎么分配任务
怎么汇总结果
```

---

## A2A

A2A：

```text
Agent ↔ Agent
```

主要解决：

```text
多个 Agent 之间
如何通信和协作
```

例如：

```text
任务委派
状态交换
结果返回
多 Agent 协作
```

一句话：

> **A2A 解决 Agent 与 Agent 怎么协作。**

---

## ANP

原笔记中 ANP 的定位：

```text
Agent Network
```

更偏向：

```text
Agent 发现
去中心化连接
大规模 Agent 网络
```

一句话：

> **ANP 更关注 Agent 之间如何形成更大规模的网络。**

---

## MCP / A2A / ANP

可以记：

```text
MCP
→ Agent ↔ Tool / Resource

A2A
→ Agent ↔ Agent

ANP
→ Agent ↔ Agent Network
```

---

# Agent Framework

## LangGraph

核心：

```text
Node
+
Edge
+
State
```

把 Agent 工作流建模成图：

```text
Node
→ 一个执行步骤

Edge
→ 流转关系

State
→ 全局任务状态
```

适合：

```text
分支
循环
ReAct
Reflection
复杂工作流
```

---

## AutoGen

核心：

```text
多个 Agent
通过对话协作
```

适合：

```text
Coder / Tester
多角色任务
自动协作
```

---

## AgentScope

原笔记中强调：

```text
标准化接口
生命周期管理
分布式通信
```

更偏：

```text
工程化多 Agent 平台
```

---

## CAMEL

核心：

```text
Role Playing
+
多 Agent 对话
```

强调：

```text
角色设定
Agent 间自然语言协作
```

---

# Agentic-RL

## 基本思想

传统 LLM 强化学习更多优化：

```text
一次回答质量
```

Agentic-RL 更关注：

```text
整个 Agent 行为过程
```

包括：

```text
观察
规划
工具调用
多步行动
环境反馈
长期奖励
```

---

## Agent 与环境交互

可以表示：

```text
Agent
 ↓
Observation
 ↓
Reasoning
 ↓
Action / Tool Call
 ↓
Environment
 ↓
Reward + New Observation
 ↓
继续
```

---

## Observation / Action / Reward

### Observation

```text
Agent 当前看到的信息
```

### Action

```text
Agent 执行的动作
```

### Reward

```text
对行动结果的评价
```

训练目标：

```text
最大化长期累计 Reward
```

---

## Tool-use RL

Tool-use RL：

```text
让模型学习
什么时候调用工具
调用哪个工具
参数怎么填
什么时候停止调用
```

重点不是：

```text
单纯生成文本
```

而是：

```text
学习行动策略
```

---

## Long-horizon RL

长时程任务：

```text
当前 Action
可能很久之后
才体现最终价值
```

因此难点：

```text
Credit Assignment
长期规划
探索
奖励设计
```

---

## Architecture Evolution

如果系统能够根据经验优化：

```text
工作流
路由策略
Agent 角色
反思机制
工具组合
```

这可以看作：

```text
Architecture Evolution
```

---

# 速记

```text
Architecture
│
├── Agent Loop
│   ├── Observation
│   ├── Reasoning
│   ├── Action
│   └── Feedback
│
├── Agent Pattern
│   ├── ReAct
│   ├── Plan-and-Solve
│   ├── Reflection
│   ├── Plan-and-Execute
│   └── Verifier
│
├── Workflow
│   ├── Planning
│   ├── Routing
│   ├── Retry
│   ├── Reflection
│   ├── Verification
│   └── Termination
│
├── Memory Architecture
│   └── MemoryManager
│
├── Multi-Agent
│   ├── Sub-Agent
│   ├── Role
│   ├── A2A
│   └── ANP
│
├── Framework
│   ├── LangGraph
│   ├── AutoGen
│   ├── AgentScope
│   └── CAMEL
│
└── Agentic-RL
    ├── Observation
    ├── Action
    ├── Reward
    ├── Tool-use RL
    └── Long-horizon RL
```

最核心区别：

```text
Model
→ 能力

Context
→ 当前信息

Tools
→ 外部能力

Architecture
→ 组织与调度
```

一句话：

> **Architecture 是 Agent 的“控制系统”，决定模型什么时候思考、什么时候调用工具、什么时候读取记忆、怎么验证、怎么循环，以及什么时候停止。**
