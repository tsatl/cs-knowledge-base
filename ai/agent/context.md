# C — Context

> **Context 决定模型这一次“能看到什么、知道什么、按什么要求行动”。**

Context 不只是 Prompt，还包括：

```text
System Prompt
User Prompt
Few-shot 示例
对话历史
Memory
RAG 检索结果
Tool Result
当前任务状态
```

核心主线：

```text
Prompt Engineering
      ↓
Context Engineering
      ↓
Memory / RAG
      ↓
长时程上下文管理
```

---

# Prompt Engineering

## Prompt 基础

编写 Prompt 的关键：

```text
明确目标
清晰表达
提供上下文
约束输出
```

常见技巧：

```text
明确角色和任务
提供背景
提供示例
限制输出格式
指定风格
逐步引导
```

---

## RDAEFS

可以用 RDAEFS 组织 Prompt：

| 部分 | 含义 |
| --- | --- |
| Role | 设定模型角色 |
| Directive | 告诉模型做什么 |
| Additional Info | 背景与限制 |
| Exemplars | Few-shot 示例 |
| Format | 输出格式 |
| Style | 语气与风格 |

可以理解：

```text
Role
→ 你是谁

Directive
→ 你要做什么

Additional Info
→ 已知条件是什么

Exemplars
→ 参考示例是什么

Format
→ 怎么输出

Style
→ 用什么风格输出
```

---

## Zero-shot 与 Few-shot

### Zero-shot

不给示例：

```text
任务说明
→ 模型直接完成
```

适合：

```text
任务简单
模型已经熟悉
```

### Few-shot

提供少量示例：

```text
示例 1
示例 2
示例 3
   ↓
模型模仿模式完成新任务
```

适合：

```text
格式严格
分类边界复杂
希望模型模仿特定风格
```

---

## 输出约束

常见约束：

```text
JSON
Markdown
表格
固定字段
固定长度
固定语气
```

核心：

> **Prompt 不仅告诉模型“做什么”，还要告诉模型“怎么做”和“怎么输出”。**

---

# Context Engineering

## Context 是什么

Context 可以理解为：

```text
一次 LLM 调用中
模型实际能够看到的全部 token
```

可能包括：

```text
系统指令
用户问题
历史消息
Memory
RAG 文档
Tool Result
当前任务状态
```

---

## Prompt Engineering vs Context Engineering

Prompt Engineering：

```text
重点：
指令怎么写
```

Context Engineering：

```text
重点：
这一次到底应该给模型看什么
```

关系：

```text
Prompt
⊂
Context
```

所以：

> **Prompt Engineering 关注“写好指令”，Context Engineering 关注“组织好整个上下文”。**

---

## Context Engineering 的目标

核心目标：

> **用尽可能少、但高信号密度的 token，提高得到目标结果的概率。**

不是：

```text
上下文越多越好
```

而是：

```text
相关
准确
新鲜
高价值
足够
```

---

## 上下文腐蚀

当上下文不断变长：

```text
Token 增加
   ↓
无关信息增加
   ↓
重要信息被稀释
   ↓
模型更难准确利用关键内容
```

因此：

```text
Context Window 大
≠
应该把所有内容都塞进去
```

---

## JIT Context

JIT：

```text
Just-In-Time Context
即时上下文
```

核心思想：

```text
不要提前加载所有内容
```

而是只维护轻量引用：

```text
文件路径
URL
数据库查询入口
索引
资源 ID
```

真正需要时：

```text
Agent
 ↓
调用工具
 ↓
动态加载
 ↓
加入当前 Context
```

优点：

```text
减少无关 Token
提高上下文新鲜度
降低上下文污染
```

---

## Progressive Disclosure

Progressive Disclosure：

```text
渐进式披露
```

核心：

```text
先看少量信息
   ↓
根据结果判断下一步
   ↓
再读取更具体内容
   ↓
逐层构建理解
```

例如：

```text
目录
 ↓
文件名
 ↓
相关文件
 ↓
相关章节
 ↓
具体内容
```

而不是：

```text
一次性加载整个项目
```

---

## 混合策略

比较实用的方法：

```text
预加载少量高价值 Context
+
按需检索其余信息
```

例如：

```text
提前加载：
README
项目规则
核心约束

运行时加载：
具体代码
日志
文档
数据库结果
```

---

# Memory

## Memory 是什么

Agent Memory 可以理解为：

```text
当前 Context 之外
可持久化保存
并在未来重新取回的信息
```

Memory 本身最终还是要：

```text
被检索
   ↓
重新放入 Context
   ↓
供模型使用
```

因此在 MCTA 中：

```text
Memory 内容
→ Context

Memory 的调度策略
→ Architecture
```

---

## 人类记忆层次

原笔记中将人类记忆分为：

```text
感觉记忆
工作记忆
长期记忆
```

长期记忆进一步包括：

```text
程序性记忆

陈述性记忆
├── 语义记忆
└── 情景记忆
```

---

## Agent 常见记忆类型

### Working Memory

工作记忆：

```text
当前任务正在使用的信息
```

特点：

```text
容量小
时效短
变化快
```

类似：

```text
当前 Context
任务草稿
临时状态
```

---

### Episodic Memory

情景记忆：

```text
记住发生过什么
```

例如：

```text
上次执行了什么
某次任务结果
某次会话发生了什么
```

通常与：

```text
时间
事件
会话
```

有关。

---

### Semantic Memory

语义记忆：

```text
记住事实和知识
```

例如：

```text
项目规则
用户偏好
领域知识
长期总结
```

---

### Perceptual Memory

感知记忆：

```text
记住看到 / 听到的内容
```

适用于：

```text
图像
音频
视频
多模态输入
```

---

## Memory 操作

常见操作：

```text
add
search
forget
consolidate
```

### add

```text
写入新记忆
```

可以附带：

```text
时间
来源
会话
重要性
标签
```

### search

```text
根据当前任务
找到相关记忆
```

通常考虑：

```text
语义相关性
时间
重要性
```

### forget

```text
删除低价值
过期
重复
无用记忆
```

### consolidate

```text
把多条零散记忆
总结 / 合并
形成更稳定的长期记忆
```

---

# RAG

## RAG 是什么

RAG：

```text
Retrieval-Augmented Generation
检索增强生成
```

核心：

> **先从外部知识库检索，再把结果放进 Context，让模型基于这些内容生成回答。**

---

## RAG 基本流程

### 数据准备

```text
外部文档
   ↓
数据提取
   ↓
文本切分 Chunking
   ↓
Embedding
   ↓
向量 / 检索索引
```

### 应用阶段

```text
用户问题
   ↓
Query
   ↓
检索相关文档
   ↓
筛选 / 排序
   ↓
注入 Prompt / Context
   ↓
LLM 生成
```

---

## Chunking

Chunking：

```text
把长文档切成较小文本块
```

目标：

```text
块不能太大
→ 否则检索不精确

块不能太小
→ 否则上下文不完整
```

核心是平衡：

```text
语义完整性
+
检索粒度
```

---

## Embedding

Embedding：

```text
文本
 ↓
向量
```

用于：

```text
语义相似度检索
```

基本思想：

```text
语义越接近
→ 向量通常越接近
```

---

## Retrieval

检索阶段负责：

```text
从知识库
找出最相关内容
```

常见思路：

```text
向量检索
关键词检索
混合检索
重排序
```

---

## Context Injection

检索结果最终需要：

```text
加入当前 Prompt / Context
```

让模型：

```text
基于检索内容
而不是只依赖参数知识
```

---

## MQE

MQE：

```text
Multi-Query Expansion
```

可以理解：

```text
一个问题
 ↓
生成多种问法
 ↓
分别检索
 ↓
合并结果
```

主要解决：

```text
用户 Query
与
文档表述不一致
```

目标：

```text
提高召回率
```

---

## HyDE

HyDE：

```text
Hypothetical Document Embeddings
```

核心：

```text
用户问题
   ↓
先生成一个假设答案 / 文档
   ↓
使用生成内容进行检索
```

目的：

```text
缩小问题表达
与文档表达之间的语义差距
```

---

## 扩展检索

可以组合：

```text
MQE
+
HyDE
+
重排序
```

目标：

```text
高召回
+
高精度
```

---

# 长时程 Context

## Compaction

Compaction：

```text
上下文越来越长
   ↓
总结已有内容
   ↓
保留关键状态
   ↓
用压缩后的 Context 继续
```

可以理解：

```text
“总结重开”
```

优点：

```text
节省 Token
支持长任务
```

风险：

```text
总结可能丢失细节
```

---

## Structured Notes

Structured Notes：

```text
把重要状态写到外部笔记
```

例如：

```text
Markdown
+
YAML / Metadata
```

适合记录：

```text
任务状态
关键结论
待办事项
重要决策
长期约束
```

优点：

```text
不持续占用 Context
需要时再检索
```

---

## GSSC

GSSC：

```text
Gather
  ↓
Select
  ↓
Structure
  ↓
Compress
```

### Gather

收集：

```text
Memory
RAG
History
Tool Result
当前状态
```

### Select

筛选：

```text
相关性
新鲜度
重要性
```

### Structure

组织：

```text
按任务需要
形成清晰 Context
```

### Compress

控制长度：

```text
摘要
裁剪
去重
合并
```

一句话：

> **先收集，再筛选，再组织，最后压缩。**

---

# 速记

```text
Context
│
├── Prompt
│   ├── Role
│   ├── Directive
│   ├── Additional Info
│   ├── Exemplars
│   ├── Format
│   └── Style
│
├── Context Engineering
│   ├── JIT
│   ├── Progressive Disclosure
│   └── Context Corrosion
│
├── Memory
│   ├── Working
│   ├── Episodic
│   ├── Semantic
│   └── Perceptual
│
├── RAG
│   ├── Chunking
│   ├── Embedding
│   ├── Retrieval
│   ├── MQE
│   └── HyDE
│
└── Long-horizon Context
    ├── Compaction
    ├── Structured Notes
    └── GSSC
```

一句话：

> **Context Engineering 的核心不是“塞更多内容”，而是让模型在当前时刻看到最相关、最准确、最有价值的信息。**
