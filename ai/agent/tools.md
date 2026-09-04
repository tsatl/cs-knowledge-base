# T — Tools

> **Tools 扩展模型的外部行动能力。**

LLM 本身只能：

```text
接收输入
进行推理
生成输出
```

Tool 可以让模型进一步：

```text
搜索网页
读取文件
执行代码
访问数据库
调用 API
操作终端
使用外部系统
```

核心关系：

```text
LLM
 ↓
选择 Tool
 ↓
调用 Tool
 ↓
获得 Observation / Result
 ↓
继续推理
```

---

# Tool 基础

## Tool Calling

Tool Calling 可以理解为：

```text
模型不直接完成某件事
而是决定调用某个外部能力
```

例如：

```text
用户：北京现在天气怎么样？

LLM
 ↓
判断需要实时天气
 ↓
调用 Weather Tool
 ↓
获得天气结果
 ↓
生成回答
```

---

## Function Calling

Function Calling 常用于把工具描述成结构化函数：

```text
函数名
参数
参数类型
功能描述
```

例如：

```text
get_weather(city)
```

模型负责：

```text
决定是否调用
选择哪个函数
生成参数
```

真正执行：

```text
由外部系统完成
```

---

## Tool Definition

一个工具通常需要描述：

```text
Name
Description
Input Schema
Output
```

高质量 Tool Definition 要：

```text
名称清楚
功能单一
参数明确
边界明确
返回结果稳定
```

---

## Tool Selection

模型可能同时拥有：

```text
Search
Browser
Code
Files
Database
Terminal
API
```

因此需要根据任务选择工具。

例如：

```text
实时信息
→ Search / Browser

精确计算
→ Calculator / Code

用户文件
→ Files

业务数据
→ Database / API
```

注意：

```text
Tool 有什么能力
→ Tools

什么时候调用哪个 Tool
→ Architecture
```

---

# 常见工具

## Search / Browser

用于：

```text
实时信息
外部知识
网页查询
资料检索
```

优势：

```text
弥补模型知识时效性
```

---

## Code / Python

用于：

```text
数学计算
数据分析
代码执行
文件生成
```

适合：

```text
模型自己算不可靠
需要确定性结果
需要程序处理
```

---

## Terminal

Terminal Tool：

```text
允许 Agent 执行受控命令
访问文件系统
运行程序
读取日志
```

常见安全机制：

```text
命令白名单
工作目录限制
沙箱
超时
输出大小限制
权限限制
```

---

## Files

用于：

```text
读取文件
搜索文档
保存结果
管理知识文件
```

典型：

```text
PDF
Markdown
Word
代码
日志
配置文件
```

---

## Database

用于：

```text
读取结构化业务数据
查询记录
写入数据
```

通常通过：

```text
SQL
数据库 API
ORM
```

访问。

---

## API

API Tool 可以连接：

```text
天气
地图
支付
邮件
GitHub
企业系统
第三方服务
```

核心：

```text
LLM
→ 负责决策

API
→ 负责真正执行
```

---

# Memory Tool

Memory 本身属于 Context，但：

```text
读写 Memory 的接口
```

属于 Tool。

常见能力：

```text
add
search
forget
consolidate
```

---

## add

```text
把信息写入 Memory
```

---

## search

```text
根据当前任务查询相关 Memory
```

---

## forget

```text
删除低价值、过期或错误记忆
```

---

## consolidate

```text
合并零散记忆
形成长期总结
```

需要区分：

```text
Memory 内容
→ Context

MemoryTool
→ Tools

何时调用 MemoryTool
→ Architecture
```

---

# MCP

## MCP 是什么

MCP：

```text
Model Context Protocol
```

可以简单理解为：

> **让 Agent 以统一方式连接外部工具和资源。**

核心关系：

```text
Agent / LLM Host
      ↓
   MCP Client
      ↓
   MCP Server
      ↓
Tools / Resources / Prompts
```

---

## Client / Server

### MCP Client

位于 Agent 侧：

```text
发现 Server 能力
发起调用
接收结果
```

### MCP Server

对外暴露能力：

```text
Tools
Resources
Prompts
```

---

## Tools

MCP Tool：

```text
可执行能力
```

例如：

```text
查数据库
创建 Issue
运行命令
调用 API
```

---

## Resources

Resource：

```text
可读取的数据 / 内容
```

例如：

```text
文件
数据库记录
文档
配置
```

---

## Prompts

Prompt：

```text
Server 提供的可复用提示模板
```

用于：

```text
标准化特定任务的上下文或指令
```

---

## MCP 的作用

MCP 主要解决：

```text
不同 Agent
如何用统一方式
接入不同外部系统
```

可以记：

> **MCP 解决 Agent ↔ Tool / Resource。**

---

# Skills

## Skill 是什么

Skill 可以理解为：

```text
可复用的高级能力包
```

它可能包含：

```text
Prompt
Tool
Workflow
规则
模板
知识
```

---

## Tool vs Skill

Tool：

```text
原子能力
```

例如：

```text
搜索
读文件
执行代码
发邮件
```

Skill：

```text
更高层、可复用的任务能力
```

例如：

```text
论文检索 Skill
报告生成 Skill
代码审查 Skill
```

可以记：

```text
Tool
→ “能做什么”

Skill
→ “把多个能力组织成什么可复用任务能力”
```

---

## Skill 与 Architecture

Skill 属于 Tools 侧：

```text
有哪些 Skill
→ Tools
```

而：

```text
什么时候选择 Skill
多个 Skill 怎么组合
失败后如何重试
怎么路由
```

属于：

```text
Architecture
```

---

## Tool / Skill Evolution

Agent 系统可以逐步扩展：

```text
发现新 Tool
创建新 Tool
组合已有 Tool
封装为 Skill
复用 Skill
```

这可以理解为：

```text
Tool Evolution
```

---

# 速记

```text
Tools
│
├── Tool Calling
├── Function Calling
│
├── 常见 Tool
│   ├── Search
│   ├── Browser
│   ├── Code
│   ├── Terminal
│   ├── Files
│   ├── Database
│   └── API
│
├── Memory Tool
│   ├── add
│   ├── search
│   ├── forget
│   └── consolidate
│
├── MCP
│   ├── Client
│   ├── Server
│   ├── Tools
│   ├── Resources
│   └── Prompts
│
└── Skills
    ├── Tool vs Skill
    ├── Skill 组合
    └── Skill Evolution
```

核心关系：

```text
Model
→ 会思考

Tools
→ 能行动

Architecture
→ 决定什么时候行动、怎么行动
```

一句话：

> **Tool 是 Agent 的“手和脚”，让模型从只会生成文本，扩展到能够真正访问和操作外部世界。**
