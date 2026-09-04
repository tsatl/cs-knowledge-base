# 计算机网络基础知识结构速记

> 本篇在原笔记基础上重新整理，并补齐网络层、HTTP/3、现代 TLS 等基础内容。

学习主线：

```text
网络模型
   ↓
网络层
   ↓
传输层
   ↓
应用层
   ↓
网络场景
   ↓
网络安全
```

---

# 网络模型

## OSI 七层模型

OSI：

```text
Application
Presentation
Session
Transport
Network
Data Link
Physical
```

从上到下：

```text
应用层
表示层
会话层
传输层
网络层
数据链路层
物理层
```

---

## 各层作用

```text
应用层
→ 为应用程序提供网络服务

表示层
→ 数据格式转换、编码、压缩、加密

会话层
→ 建立、管理、终止通信会话

传输层
→ 端到端数据传输

网络层
→ IP 寻址、路由、分组转发

数据链路层
→ Frame、MAC 寻址、差错检测

物理层
→ Bit、电信号 / 光信号 / 无线信号
```

重点：

```text
数据链路层
→ Frame

物理层
→ Bit
```

---

## TCP/IP 四层模型

```text
应用层
传输层
网络层
网络接口层
```

常见协议：

```text
应用层
→ HTTP / HTTPS / DNS / SMTP / FTP

传输层
→ TCP / UDP

网络层
→ IP / ICMP

网络接口层
→ Ethernet / Wi-Fi / ARP 等
```

---

## OSI 与 TCP/IP 对应

```text
OSI                         TCP/IP

应用层 ┐
表示层 ├──────────────→ 应用层
会话层 ┘

传输层 ───────────────→ 传输层

网络层 ───────────────→ 网络层

数据链路层 ┐
          ├──────────→ 网络接口层
物理层    ┘
```

---

## 数据封装

发送：

```text
Application Data
      ↓
TCP / UDP Segment
      ↓
IP Packet
      ↓
Ethernet Frame
      ↓
Bit
```

接收端反向：

```text
Bit
 ↓
Frame
 ↓
Packet
 ↓
Segment
 ↓
Application Data
```

一句话：

> **发送端逐层加头，接收端逐层拆头。**

---

# 网络层

## IP

IP：

```text
Internet Protocol
```

主要负责：

```text
逻辑寻址
路由选择
分组转发
```

IP 提供的是：

```text
Best Effort
尽力而为
```

协议本身：

```text
不保证一定到达
不保证按序
不保证不重复
```

可靠性通常交给：

```text
TCP
或上层协议
```

---

## IPv4

IPv4 地址：

```text
32 Bit
```

例如：

```text
192.168.1.10
```

由：

```text
网络部分
+
主机部分
```

组成。

---

## IPv6

IPv6：

```text
128 Bit
```

主要解决：

```text
IPv4 地址空间不足
```

还带来：

```text
更大的地址空间
更简化的基础首部
更好的自动配置能力
```

---

## 子网掩码

用于判断：

```text
IP 的哪些位属于网络号
哪些位属于主机号
```

例如：

```text
IP:
192.168.1.100

Mask:
255.255.255.0
```

等价：

```text
192.168.1.100/24
```

---

## CIDR

CIDR：

```text
Classless Inter-Domain Routing
无类别域间路由
```

形式：

```text
IP / Prefix Length
```

例如：

```text
192.168.1.0/24
```

表示：

```text
前 24 位
→ 网络前缀

后 8 位
→ 主机部分
```

---

## 是否同一网段

判断：

```text
IP & Subnet Mask
```

如果两个 IP：

```text
网络号相同
→ 同一网段

网络号不同
→ 需要经过路由器 / 网关
```

---

## ARP

ARP：

```text
Address Resolution Protocol
```

作用：

```text
IPv4 地址
→ MAC 地址
```

---

## ARP 使用场景

如果目标：

```text
在本地网段
```

需要：

```text
ARP 查询目标主机 MAC
```

如果目标：

```text
不在本地网段
```

不会直接查询远程服务器 MAC。

而是：

```text
目标 IP 不同网段
   ↓
查路由表
   ↓
找到下一跳 / 默认网关
   ↓
ARP 获取网关 MAC
   ↓
Frame 发给网关
```

重点：

> **跨网段通信时，客户端需要的是下一跳网关 MAC，不是远程服务器 MAC。**

---

## MAC 地址

MAC：

```text
Media Access Control Address
```

作用范围主要是：

```text
本地链路
```

IP：

```text
用于跨网络逻辑寻址
```

MAC：

```text
用于当前链路帧传输
```

---

## ICMP

ICMP：

```text
Internet Control Message Protocol
```

主要：

```text
网络状态和错误控制信息
```

常见：

```text
Echo Request
Echo Reply
Destination Unreachable
Time Exceeded
```

---

## Ping

Ping 主要基于：

```text
ICMP Echo Request / Reply
```

所以：

```text
Ping 不通
≠
服务器一定不可访问
```

例如：

```text
Firewall
禁止 ICMP

但 TCP 443
允许
```

就可能：

```text
Ping 不通
HTTPS 正常
```

---

## 路由

主机发送数据前：

```text
查看 Routing Table
```

判断：

```text
目标是否本地
应该走哪个网卡
下一跳是谁
```

---

## 默认网关

如果没有更具体路由：

```text
使用 Default Route
```

例如：

```text
0.0.0.0/0
→ Default Gateway
```

---

## NAT

NAT：

```text
Network Address Translation
```

常见作用：

```text
私网 IP
↔
公网 IP
```

典型家庭网络：

```text
PC 192.168.1.10
   ↓
Router NAT
   ↓
Public IP
   ↓
Internet
```

---

## 为什么需要 NAT

主要：

```text
缓解 IPv4 地址不足
隐藏内部地址结构
多个内网设备共享公网地址
```

但 NAT：

```text
不是防火墙本身
```

不能简单认为：

```text
有 NAT
→ 就绝对安全
```

---

# 传输层

## TCP

TCP：

```text
Transmission Control Protocol
```

特点：

```text
面向连接
可靠字节流
全双工
有序
支持流量控制
支持拥塞控制
```

---

## TCP 重要头部字段

### Sequence Number

序列号：

```text
标识数据在字节流中的位置
```

建立连接时：

```text
双方选择初始序列号 ISN
```

之后：

```text
Seq
按发送的数据字节数推进
```

作用：

```text
排序
去重
可靠确认
```

---

### Acknowledgment Number

确认号：

```text
表示下一次期望收到的字节序号
```

例如：

```text
ACK = 101
```

可以理解：

```text
100 及之前的数据
我已经连续收到
接下来希望 101
```

---

### Flags

常见：

```text
SYN
ACK
FIN
RST
PSH
URG
```

重点：

```text
SYN
→ 建立连接

ACK
→ 确认字段有效

FIN
→ 正常关闭一个方向

RST
→ 异常重置连接
```

---

# TCP 三次握手

假设：

```text
Client ISN = x
Server ISN = y
```

---

## 第一次

Client：

```text
SYN = 1
Seq = x
```

发送：

```text
Client
   ───── SYN(x) ────→
Server
```

Client：

```text
SYN_SENT
```

---

## 第二次

Server：

```text
SYN = 1
ACK = 1
Seq = y
Ack = x + 1
```

发送：

```text
Client
←── SYN(y) + ACK(x+1) ──
Server
```

Server：

```text
SYN_RCVD
```

---

## 第三次

Client：

```text
ACK = 1
Ack = y + 1
```

发送：

```text
Client
   ───── ACK(y+1) ────→
Server
```

之后：

```text
双方
→ ESTABLISHED
```

第三次 ACK：

```text
可以携带应用数据
```

---

## 为什么三次握手

主要：

```text
确认双方收发能力
同步双方初始序列号
防止历史连接请求造成错误连接
避免无意义资源占用
```

可以理解：

```text
1：
Client 告诉 Server
“我想连你，我的 ISN 是 x”

2：
Server 告诉 Client
“收到 x，我的 ISN 是 y”

3：
Client 告诉 Server
“y 我也收到了”
```

---

# TCP 为什么可靠

TCP 可靠性主要来自：

```text
连接管理
序列号
ACK
Checksum
超时重传
快速重传
滑动窗口
流量控制
拥塞控制
```

---

## 序列号

解决：

```text
乱序
重复
```

---

## ACK

接收方：

```text
确认已经连续收到的数据
```

发送方根据：

```text
ACK
```

知道哪些数据已经成功到达。

---

## Checksum

用于：

```text
检测报文在传输过程中
是否发生比特错误
```

---

## 超时重传

发送数据后：

```text
迟迟没有收到 ACK
```

超过：

```text
RTO
```

则：

```text
重新发送
```

---

## 快速重传

如果连续收到：

```text
3 个 Duplicate ACK
```

经典 TCP 会认为：

```text
中间某个 Segment 可能丢失
```

不等 RTO：

```text
立即重传
```

---

# 流量控制

流量控制解决：

> **不要让发送方发得太快，把接收方缓冲区打爆。**

接收方通过：

```text
rwnd
Receive Window
```

告诉发送方：

```text
自己还能接收多少数据
```

---

## rwnd

```text
接收窗口
→ 接收方维护 / 通告
```

---

## 滑动窗口

发送方：

```text
不必每发一个 Segment
就停下来等 ACK
```

而是可以：

```text
在窗口范围内
连续发送多个 Segment
```

提高吞吐率。

---

# 拥塞控制

拥塞控制解决：

> **不要让发送方把整个网络打爆。**

发送方维护：

```text
cwnd
Congestion Window
```

真正发送窗口：

```text
swnd = min(rwnd, cwnd)
```

其中：

```text
rwnd
→ 接收方能力

cwnd
→ 网络承载能力估计
```

---

## 经典 TCP Reno

下面的：

```text
Slow Start
Congestion Avoidance
Fast Retransmit
Fast Recovery
```

属于经典 Reno 思路。

现代操作系统还可能使用：

```text
CUBIC
BBR
```

等其他拥塞控制算法。

---

## Slow Start

条件：

```text
cwnd < ssthresh
```

经典理解：

```text
一个 RTT
cwnd 大约翻倍
```

属于：

```text
指数增长
```

---

## Congestion Avoidance

当：

```text
cwnd >= ssthresh
```

进入：

```text
Congestion Avoidance
```

经典 Reno 中：

```text
一个 RTT
cwnd 大约增加 1 MSS
```

属于：

```text
线性增长
```

---

## Timeout

超时：

```text
通常说明拥塞可能比较严重
```

经典 Reno：

```text
ssthresh
≈ 原窗口一半

cwnd
→ 重新减小

再进入慢启动
```

具体实现可能随系统和算法不同。

---

## 3 Duplicate ACK

连续：

```text
3 个 Duplicate ACK
```

说明：

```text
有数据丢失
但后续数据仍能到达
```

于是：

```text
Fast Retransmit
```

随后进入：

```text
Fast Recovery
```

---

## 经典流程

```text
             Slow Start
             指数增长
                ↓
        cwnd >= ssthresh
                ↓
       Congestion Avoidance
             线性增长
          ↙             ↘
      Timeout       3 Duplicate ACK
         ↓               ↓
    大幅降低窗口       Fast Retransmit
         ↓               ↓
    Slow Start       Fast Recovery
                         ↓
                 Congestion Avoidance
```

---

# TCP 四次挥手

TCP：

```text
全双工
```

两个方向需要分别关闭。

假设 Client 主动关闭。

---

## 第一次

Client：

```text
FIN
```

告诉 Server：

```text
我没有数据要继续发了
```

Client：

```text
FIN_WAIT_1
```

---

## 第二次

Server：

```text
ACK
```

Server：

```text
CLOSE_WAIT
```

Client：

```text
FIN_WAIT_2
```

注意：

```text
Client 不再发送数据
≠
Server 不能继续发送数据
```

---

## 第三次

Server 业务数据发送完：

```text
FIN
```

Server：

```text
LAST_ACK
```

---

## 第四次

Client：

```text
ACK
```

然后：

```text
Client
→ TIME_WAIT

Server
→ 收到 ACK 后 CLOSE
```

Client 等：

```text
2MSL
```

之后：

```text
CLOSE
```

---

# TIME_WAIT

哪一方：

```text
主动完成关闭
```

通常哪一方进入：

```text
TIME_WAIT
```

---

## 为什么等待 2MSL

两个主要原因：

```text
1. 确保最后 ACK 可重传

2. 让网络中的旧报文
   有足够时间消失
```

---

## 服务端大量 TIME_WAIT

表示：

```text
Server 在大量连接中
承担 Active Close
```

常见：

```text
服务端主动关闭短连接
HTTP Keep-Alive 超时
应用主动 close
代理 / 上游短连接
```

不要简单认为：

```text
服务器出现 TIME_WAIT
→ 一定是异常
```

关键：

```text
数量
增长趋势
连接模型
端口资源
```

---

# TCP 粘包与拆包

TCP 提供：

```text
Byte Stream
字节流
```

它：

```text
没有应用层消息边界
```

所谓：

```text
粘包 / 拆包
```

本质上是：

```text
应用层协议
如何划分 Message Boundary
```

---

## 解决方式

### 固定长度

```text
每个消息固定 N Bytes
```

---

### 分隔符

例如：

```text
\r\n
```

或其他特殊字符作为消息边界。

---

### Length + Body

最常见：

```text
Header
├── length
└── ...

Body
└── payload
```

接收：

```text
先读 Header
 ↓
得到 Length
 ↓
再读取 Length 个字节
```

---

# TCP Keepalive

TCP Keepalive：

```text
内核级死连接探测机制
```

如果连接长时间没有活动：

```text
发送 Probe
```

若多次无响应：

```text
认为连接失效
```

---

## 对端正常

```text
Probe
 ↓
ACK
 ↓
连接继续
```

---

## 对端重启

如果对端系统已经：

```text
丢失原连接状态
```

收到 Probe 后可能：

```text
返回 RST
```

于是本端发现：

```text
连接失效
```

---

## 主机断电 / 网络中断

Probe：

```text
连续没有响应
```

达到阈值后：

```text
内核报告连接失效
```

---

## TCP Keepalive vs HTTP Keep-Alive

完全不同：

```text
TCP Keepalive
→ 探测死连接

HTTP Keep-Alive
→ 多个 HTTP 请求
  复用同一个连接
```

---

# UDP

UDP：

```text
User Datagram Protocol
```

特点：

```text
无连接
面向 Datagram
协议头简单
不保证可靠
不保证顺序
不负责重传
```

---

## TCP vs UDP

| 对比 | TCP | UDP |
| --- | --- | --- |
| 连接 | 面向连接 | 无连接 |
| 数据形式 | Byte Stream | Datagram |
| 可靠性 | 提供可靠机制 | 协议本身不保证 |
| 顺序 | 保证有序交付 | 不保证 |
| 重传 | 有 | 无 |
| 流控 | 有 | 无 |
| 拥塞控制 | 有 | UDP 本身无 |
| 协议开销 | 较高 | 较低 |

不要简单记：

```text
TCP 慢
UDP 快
```

更准确：

```text
UDP 控制机制更少
协议开销更低
```

业务整体性能：

```text
取决于具体协议设计
```

例如：

```text
QUIC
→ 基于 UDP
→ 自己实现可靠传输、拥塞控制等
```

---

# 应用层

## DNS

DNS：

```text
Domain Name System
域名系统
```

作用：

```text
Domain Name
→ IP Address
```

例如：

```text
www.example.com
   ↓
93.x.x.x
```

---

## DNS 特点

```text
分层命名
分布式数据库
缓存
```

传统 DNS：

```text
UDP 53
为主
```

某些情况也会使用：

```text
TCP 53
```

另外还有：

```text
DoT
DoH
```

用于加密 DNS 查询。

---

## DNS 层级

```text
Root
 ↓
Top-Level Domain
 ↓
Authoritative DNS
 ↓
Host Record
```

例如：

```text
www.example.com
```

从右到左：

```text
.
└── com
    └── example
        └── www
```

---

## Root DNS

传统上有：

```text
13 个逻辑 Root Server 标识
A ~ M
```

注意：

```text
不是全球只有 13 台物理服务器
```

实际通过：

```text
Anycast
```

部署大量实例。

---

## DNS 查询基本流程

```text
Browser / OS
   ↓
Local DNS Resolver
   ↓
Root DNS
   ↓
TLD DNS
   ↓
Authoritative DNS
   ↓
得到 IP
```

实际过程中：

```text
Browser Cache
OS Cache
Local DNS Cache
```

都可能直接命中。

---

## 递归查询

递归：

```text
请求方只问一个服务器
服务器负责把最终结果查回来
```

最典型：

```text
Client
→ Local DNS Resolver
```

客户端：

```text
希望得到最终答案
```

---

## 迭代查询

迭代：

```text
当前 DNS
不知道最终答案
```

但告诉查询者：

```text
下一步应该找谁
```

典型：

```text
Local DNS
→ Root

Root
→ “去问 .com TLD”

Local DNS
→ .com TLD

TLD
→ “去问 example.com Authoritative”

Local DNS
→ Authoritative
```

---

## DNS 查询关系

可以记：

```text
Client → Local DNS
→ 通常递归

Local DNS → Root / TLD / Authoritative
→ 通常迭代
```

---

## DNS 缓存

DNS Resolver 会缓存：

```text
Domain → IP
```

一段时间。

时间由：

```text
TTL
```

影响。

作用：

```text
减少 DNS 查询延迟
降低上级 DNS 压力
```

缺点：

```text
记录变更后
缓存可能暂时还是旧值
```

---

# HTTP

HTTP：

```text
HyperText Transfer Protocol
```

应用层协议。

传统：

```text
HTTP/1.1 / HTTP/2
→ 通常基于 TCP
```

---

## HTTP 请求报文

```text
Request Line
Headers
空行
Body
```

---

### Request Line

例如：

```http
GET /users/1 HTTP/1.1
```

包括：

```text
Method
Request Target
HTTP Version
```

---

### Headers

例如：

```text
Host
User-Agent
Content-Type
Authorization
Cookie
```

---

### Body

常用于：

```text
POST
PUT
PATCH
```

携带：

```text
JSON
Form
Binary Data
```

GET 语义通常不依赖 Request Body。

---

## HTTP 响应报文

```text
Status Line
Headers
空行
Body
```

---

### Status Line

例如：

```http
HTTP/1.1 200 OK
```

---

# HTTP Method

常见：

```text
GET
POST
PUT
PATCH
DELETE
HEAD
OPTIONS
```

---

## GET

```text
获取资源
```

---

## POST

常用于：

```text
创建资源
提交数据
触发操作
```

---

## PUT

通常：

```text
整体更新 / 替换资源
```

---

## PATCH

通常：

```text
部分更新资源
```

---

## DELETE

```text
删除资源
```

---

## HEAD

类似 GET：

```text
只获取 Response Headers
不返回完整 Body
```

---

# HTTP 状态码

## 1xx

```text
Informational
信息性响应
```

---

## 2xx

成功。

常见：

```text
200 OK
201 Created
204 No Content
```

---

## 3xx

重定向 / 缓存相关。

```text
301 Moved Permanently
302 Found
304 Not Modified
```

---

## 4xx

客户端请求相关问题。

```text
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
405 Method Not Allowed
```

---

## 401 vs 403

```text
401
→ 通常表示缺少 / 无效认证凭证

403
→ 已识别请求者
  但没有权限访问
```

---

## 5xx

服务器 / 上游处理问题。

```text
500 Internal Server Error
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
```

---

## 502 vs 504

```text
502
→ 网关从上游
  收到无效响应

504
→ 网关等待上游
  超时
```

---

# HTTP 持久连接

## 非持久连接

典型：

```text
一个请求
   ↓
一个 TCP Connection
   ↓
响应完成
   ↓
关闭
```

连接建立开销较大。

---

## HTTP Keep-Alive

多个 HTTP Request / Response：

```text
复用一个 TCP Connection
```

HTTP/1.1：

```text
默认支持持久连接
```

作用：

```text
减少 TCP 建连次数
降低延迟
```

---

# HTTP 版本

## HTTP/1.0

经典：

```text
默认非持久连接
```

---

## HTTP/1.1

主要：

```text
默认持久连接
Host Header
Chunked Transfer
缓存能力增强
```

多个请求仍然可能受到：

```text
HTTP/1.x 队头阻塞
```

---

## HTTP/2

核心：

```text
Binary Framing
Header Compression
Stream Multiplexing
```

---

### Binary Framing

HTTP 报文：

```text
拆成二进制 Frame
```

---

### Header Compression

使用：

```text
HPACK
```

减少重复 Header 开销。

---

### Multiplexing

一个 TCP Connection：

```text
同时承载多个 Stream
```

解决 HTTP/1.x 应用层一个请求阻塞后续请求的问题。

但 HTTP/2：

```text
仍然基于一个 TCP
```

如果 TCP 层发生丢包：

```text
可能影响多个 Stream
```

---

## HTTP/3

HTTP/3：

```text
HTTP over QUIC
```

QUIC：

```text
基于 UDP
```

典型：

```text
UDP 443
```

特点：

```text
更快连接建立
Stream 独立
减少 TCP 层队头阻塞影响
内置 TLS 1.3
```

---

# HTTP 为什么不安全

纯 HTTP：

```text
明文传输
```

风险：

```text
窃听
篡改
冒充
```

---

# HTTPS

HTTPS：

```text
HTTP
+
TLS
```

解决：

```text
Confidentiality
机密性

Integrity
完整性

Authentication
身份认证
```

传统：

```text
HTTP/1.1 / HTTP/2 over TLS
→ TCP 443
```

HTTP/3：

```text
HTTP/3 over QUIC
→ UDP 443
```

---

# TLS 基本思想

TLS 结合：

```text
Certificate
非对称密码 / 密钥交换
对称加密
Hash / AEAD
```

---

## 为什么不用非对称加密传全部数据

非对称加密：

```text
计算成本高
```

对称加密：

```text
速度快
适合大量业务数据
```

因此：

```text
身份认证 / 密钥协商
→ 非对称密码等机制

业务数据
→ 对称加密
```

---

# 数字证书

服务器提供：

```text
Certificate
```

证书中包含：

```text
域名
公钥
颁发者
有效期
签名
```

客户端验证：

```text
证书链是否可信
域名是否匹配
是否过期
签名是否正确
```

目的：

```text
证明公钥确实属于目标服务器
```

---

# TLS 1.3 简化握手

现代 TLS 1.3 可以简单理解：

```text
ClientHello
→ Supported Versions
→ Cipher Suites
→ Key Share
        ↓
ServerHello
→ 选择参数
→ Server Key Share
        ↓
Certificate
CertificateVerify
Finished
        ↓
双方通过 (EC)DHE
推导 Session Keys
        ↓
对称加密通信
```

重点：

```text
现代 TLS
通常不是简单的
“客户端生成 Premaster Secret
再用服务器 RSA 公钥加密”
```

那是典型旧版 TLS/RSA Key Exchange 理解。

---

# Cookie、Session、Token、JWT

## Cookie

Cookie：

```text
浏览器保存的一小段数据
```

服务器可以通过：

```http
Set-Cookie
```

让浏览器保存。

后续符合条件的请求：

```text
Browser
→ 自动携带 Cookie
```

Cookie 可用于：

```text
Session ID
偏好设置
认证凭证
追踪信息
```

---

## Cookie 常见属性

```text
Expires / Max-Age
Domain
Path
Secure
HttpOnly
SameSite
```

---

### Secure

```text
只通过 HTTPS 发送
```

---

### HttpOnly

```text
JavaScript 无法通过 document.cookie
直接读取
```

降低：

```text
XSS 直接窃取 Cookie
```

的风险。

---

### SameSite

控制：

```text
跨站请求
是否自动携带 Cookie
```

有助于：

```text
降低 CSRF 风险
```

---

# Session

Session：

```text
服务器端保存会话状态
```

常见：

```text
Server
→ 创建 Session

Server
→ 生成 Session ID

Browser
→ 保存 Session ID

后续请求
→ 携带 Session ID

Server
→ 找到对应 Session
```

Session ID 常放：

```text
Cookie
```

但：

```text
Session ≠ Cookie
```

Cookie 只是：

```text
携带 Session ID
的一种常见方式
```

---

## 禁用 Cookie 后 Session

Session 本身：

```text
仍然可以存在
```

关键是：

```text
客户端如何把 Session ID
带回服务器
```

替代方案：

```text
URL Rewriting
隐藏表单字段
自定义 Header
```

但：

```text
安全性
使用体验
```

通常不如 Cookie 方案自然。

---

# Token

Token：

```text
身份 / 授权凭证
的泛称
```

不要简单理解：

```text
Token = 无状态
```

Token 可以是：

```text
Stateful Token
→ Opaque Token
→ Server 查 DB / Redis

Stateless Token
→ JWT 是常见实现之一
```

---

# JWT

JWT：

```text
JSON Web Token
```

典型结构：

```text
Header
.
Payload
.
Signature
```

---

## Header

通常：

```text
Token Type
Signature Algorithm
```

---

## Payload

保存：

```text
Claims
```

例如：

```text
sub
exp
iat
roles
```

注意：

```text
Payload
通常只是 Base64URL 编码
不是加密
```

所以：

```text
不要存密码等秘密信息
```

---

## Signature

作用：

```text
验证 Token
有没有被篡改

验证签发者
是否掌握正确密钥
```

签名保证：

```text
Integrity
Authenticity
```

不直接保证：

```text
Confidentiality
```

---

## JWT 工作流程

```text
User Login
   ↓
Server 验证身份
   ↓
签发 JWT
   ↓
Client 保存
   ↓
后续请求携带 JWT
   ↓
Server 验证签名 / Claims
   ↓
授权访问
```

---

## JWT 并不天然比 Session 更安全

安全性取决于：

```text
Token 存哪里
是否 HTTPS
有效期
密钥管理
签名算法
XSS
CSRF
撤销机制
```

例如：

```text
JWT 放 Cookie
→ 仍需要考虑 CSRF

JWT 放 localStorage
→ 更需要考虑 XSS
```

---

# Cookie / Session / Token / JWT 关系

可以记：

```text
Cookie
→ 客户端怎么保存 / 携带数据

Session
→ 服务端状态放哪里

Token
→ 客户端拿什么证明身份

JWT
→ Token 的一种具体格式
```

重点：

```text
Cookie
≠ Session
≠ Token
```

而且：

```text
JWT
可以放 Cookie
```

---

# Token 存储方式

## localStorage

特点：

```text
容量大
刷新页面后仍存在
不会自动随每个 HTTP 请求发送
```

风险：

```text
JavaScript 可读取
→ XSS 风险高
```

---

## sessionStorage

特点：

```text
Tab / Window 会话级
```

一般：

```text
刷新页面
→ 仍存在

关闭对应 Tab
→ 被清除
```

同样：

```text
JavaScript 可读取
→ 有 XSS 风险
```

---

## Cookie

优点：

```text
可设置 HttpOnly
可设置 Secure
可设置 SameSite
```

缺点：

```text
容量较小
符合条件的请求自动携带
需要考虑 CSRF
```

---

# JWT 撤销与刷新

JWT 自包含：

```text
签发后
通常不能像 Server Session
那样直接删除就失效
```

常见：

```text
短 Access Token
+
Refresh Token
```

---

## Access Token

```text
生命周期较短
```

用于：

```text
访问 API
```

---

## Refresh Token

```text
生命周期较长
```

用于：

```text
获取新的 Access Token
```

---

## 撤销方式

可以：

```text
Blacklist / Revocation List
Token Version
Refresh Token Rotation
修改密码后使旧 Token 失效
缩短 Access Token TTL
```

如果维护 Blacklist：

```text
系统就引入了额外状态
```

这也是安全性和无状态之间的权衡。

---

# 网络场景

## 浏览器输入 URL 后发生什么

可以按：

```text
URL
 ↓
Cache
 ↓
DNS
 ↓
Route / ARP
 ↓
TCP / QUIC
 ↓
TLS
 ↓
HTTP
 ↓
Server
 ↓
Response
 ↓
Browser Render
```

理解。

---

## 1. 解析 URL

浏览器解析：

```text
Scheme
Host
Port
Path
Query
```

例如：

```text
https://www.example.com/users?id=1
```

---

## 2. 缓存判断

浏览器可能检查：

```text
HTTP Cache
Service Worker
DNS Cache
HSTS
```

如果已有可用缓存：

```text
可能不需要完整网络请求
```

---

## 3. DNS

```text
Domain
→ IP
```

可能经过：

```text
Browser Cache
OS Cache
Local DNS
Root / TLD / Authoritative
```

---

## 4. 路由与 MAC

得到目标 IP 后：

```text
查 Routing Table
```

如果不同网段：

```text
找到 Default Gateway
   ↓
ARP 获取 Gateway MAC
```

注意：

```text
不是直接获取远程 Web Server MAC
```

---

## 5. 建立传输层连接

HTTP/1.1 / HTTP/2：

```text
TCP Three-Way Handshake
```

HTTP/3：

```text
QUIC
```

---

## 6. TLS

HTTPS：

```text
进行 TLS 握手
验证 Certificate
协商 / 推导 Session Keys
```

---

## 7. HTTP Request

例如：

```http
GET / HTTP/1.1
Host: www.example.com
```

---

## 8. Server 处理

服务器可能：

```text
Load Balancer
 ↓
Nginx / Gateway
 ↓
Application
 ↓
Redis / Database / RPC
```

---

## 9. HTTP Response

返回：

```text
Status Code
Headers
Body
```

---

## 10. 浏览器渲染

浏览器解析：

```text
HTML
CSS
JavaScript
Images
Fonts
```

并：

```text
构建 DOM
构建 CSSOM
Layout
Paint
Composite
```

---

# 网页访问很慢如何排查

不要直接猜：

```text
数据库慢
```

按层排查。

---

## Client

先确认：

```text
是不是只有这个网站慢
其他网站是否正常
CPU / Browser 是否异常
```

---

## DNS

检查：

```text
DNS 是否解析成功
DNS 耗时是否异常
```

---

## Network

关注：

```text
RTT
Packet Loss
Route
Bandwidth
```

---

## TCP / QUIC

查看：

```text
Connection Time
Retransmission
连接是否频繁建立
```

---

## TLS

检查：

```text
TLS Handshake Time
Certificate 问题
```

---

## Server

重点：

```text
TTFB
Time To First Byte
```

如果 TTFB 很长：

```text
Server / Application
可能处理慢
```

---

## Application

继续排：

```text
Thread Pool
DB
Redis
RPC
Disk IO
CPU
GC
```

---

## Frontend

如果：

```text
HTTP 200 很快
但页面仍然很慢
```

检查：

```text
大资源
大量 Request
JavaScript 执行
DOM Rendering
第三方资源
```

浏览器 DevTools 常见时间：

```text
DNS
Connect
SSL
TTFB
Content Download
```

---

# 两台服务器如何判断连接正常

不要只依赖：

```text
Ping
```

可以分层：

```text
ICMP
TCP
Application
```

---

## ICMP

```bash
ping host
```

只能说明：

```text
ICMP 是否可达
```

---

## TCP

例如检查：

```text
目标 IP:Port
```

是否能完成：

```text
TCP Handshake
```

---

## Application

即使 TCP 成功：

```text
应用也可能异常
```

最终还要检查：

```text
HTTP Status
RPC Response
业务 Health Check
```

---

# Ping 不通但 HTTP 正常

完全可能。

因为：

```text
Ping
→ ICMP

HTTP/1.1 / HTTP/2
→ TCP
```

Firewall 可能：

```text
Drop ICMP
Allow TCP 80 / 443
```

于是：

```text
Ping Fail
HTTP Success
```

---

# 网络安全

> 本节只做基础速记。

## DDoS

DDoS：

```text
Distributed Denial of Service
```

通过大量：

```text
流量
连接
请求
```

耗尽：

```text
Bandwidth
Connection
CPU
Application Resource
```

---

## 常见类型

```text
Volumetric
→ UDP / DNS / NTP Amplification

Protocol
→ SYN Flood

Application
→ HTTP Flood
```

---

## 基础防护

```text
Rate Limit
CDN / Anycast
Traffic Scrubbing
Load Balancing
Firewall / WAF
SYN Cookie
Capacity Expansion
```

---

# SQL Injection

原因：

```text
用户输入
被直接拼进 SQL 结构
```

例如：

```text
"SELECT ... WHERE name = '" + input + "'"
```

---

## 防护

第一优先：

```text
Parameterized Query
PreparedStatement
```

例如 MyBatis：

```text
#{value}
```

其次：

```text
输入验证
白名单
最小权限
```

不要把：

```text
过滤几个特殊字符
```

作为核心防线。

---

# CSRF

CSRF：

```text
Cross-Site Request Forgery
```

核心：

```text
浏览器可能自动携带
目标站 Cookie
```

攻击者诱导用户：

```text
向目标站发起恶意请求
```

---

## 防护

```text
SameSite Cookie
CSRF Token
Origin / Referer 校验
重要操作二次认证
```

---

# XSS

XSS：

```text
Cross-Site Scripting
```

攻击者让：

```text
恶意 JavaScript
在受害者浏览器执行
```

---

## 类型

```text
Stored XSS
Reflected XSS
DOM XSS
```

---

## 防护

```text
上下文相关输出编码
HTML Sanitization
模板默认 Escaping
避免危险 DOM API
CSP
HttpOnly Cookie
```

注意：

```text
HttpOnly
只能降低 Cookie 被 JS 直接读取的风险
不能阻止 XSS 本身
```

---

# DNS Hijacking

DNS Hijacking：

```text
DNS 查询 / 响应
被攻击者篡改
```

导致：

```text
正常域名
→ 恶意 IP
```

降低风险：

```text
可信 DNS
DoH / DoT
DNSSEC
HTTPS Certificate Validation
```

---

# 速记

## 网络模型

```text
应用层
→ HTTP / DNS

传输层
→ TCP / UDP

网络层
→ IP / ICMP

链路层
→ MAC / Frame

物理层
→ Bit
```

---

## 数据封装

```text
Data
 ↓
Segment
 ↓
Packet
 ↓
Frame
 ↓
Bit
```

---

## ARP

```text
IPv4
→ MAC
```

跨网段：

```text
不是找远程服务器 MAC
而是找下一跳网关 MAC
```

---

## Ping

```text
Ping
→ ICMP

HTTP
→ TCP / QUIC
```

所以：

```text
Ping 不通
HTTP 可能正常
```

---

## TCP

```text
面向连接
可靠字节流
```

可靠：

```text
Seq
ACK
Checksum
Retransmission
Sliding Window
Flow Control
Congestion Control
```

---

## 三次握手

```text
Client
→ SYN

Server
→ SYN + ACK

Client
→ ACK
```

目的：

```text
确认双方能力
同步 ISN
避免历史连接
```

---

## 四次挥手

```text
FIN
 ↓
ACK
 ↓
FIN
 ↓
ACK
```

主动关闭方：

```text
通常进入 TIME_WAIT
```

---

## TIME_WAIT

```text
2MSL
```

目的：

```text
最后 ACK 可重传
+
旧报文消失
```

---

## 流量 vs 拥塞

```text
rwnd
→ 接收方
→ 别把接收方打爆

cwnd
→ 发送方
→ 别把网络打爆

swnd
= min(rwnd, cwnd)
```

---

## TCP 粘包

```text
TCP
→ Byte Stream
→ 没有消息边界
```

应用层解决：

```text
固定长度
Delimiter
Length + Body
```

---

## UDP

```text
无连接
Datagram
协议本身不保证可靠
```

不要死记：

```text
UDP 一定比 TCP 快
```

---

## DNS

```text
Domain
 ↓
Local DNS
 ↓
Root
 ↓
TLD
 ↓
Authoritative
 ↓
IP
```

记：

```text
Client → Local DNS
→ 通常递归

Local DNS → 各级 DNS
→ 通常迭代
```

---

## HTTP

```text
Request
→ Request Line + Headers + Body

Response
→ Status Line + Headers + Body
```

---

## HTTP 版本

```text
HTTP/1.0
→ 默认非持久

HTTP/1.1
→ 默认 Keep-Alive

HTTP/2
→ Binary + HPACK + Multiplexing

HTTP/3
→ QUIC / UDP + TLS 1.3
```

---

## HTTPS

```text
HTTPS
= HTTP + TLS
```

TLS：

```text
Certificate
→ 身份认证

Key Exchange
→ 建立共享密钥

Symmetric Encryption
→ 传业务数据
```

---

## Cookie / Session / Token / JWT

```text
Cookie
→ 客户端怎么保存 / 携带

Session
→ 服务端状态放哪里

Token
→ 客户端拿什么证明身份

JWT
→ Token 的一种格式
```

---

## Web 安全

```text
SQL Injection
→ 参数化查询

CSRF
→ SameSite + CSRF Token

XSS
→ 输出编码 + CSP

DDoS
→ 限流 + 清洗 + 高可用

DNS Hijacking
→ 可信 DNS + 加密 DNS + HTTPS
```

---

# 一句话总结

```text
OSI / TCP-IP
→ 网络分层

IP / ARP / Route
→ 数据怎么找到下一跳

TCP / UDP
→ 主机之间怎么传数据

DNS
→ 域名怎么找到 IP

HTTP
→ 应用怎么交换请求和响应

TLS
→ 数据怎么安全传输

Cookie / Session / Token
→ Web 怎么维持身份和状态

Network Troubleshooting
→ 网络慢了怎么定位

Web Security
→ 常见攻击怎么基础防护
```
