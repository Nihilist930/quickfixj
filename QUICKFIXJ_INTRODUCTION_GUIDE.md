# QuickFIX/J 介绍纲领：从业务流程到会话可靠性

> **目标**：用一笔订单说明业务如何接入，用消息转换说明 FIX 协议如何落地，用会话恢复说明生产环境如何保证通信可靠。  
> **适用场景**：学习 QuickFIX/J、向团队介绍 FIX 接入架构、准备 30～45 分钟的技术分享。

---

## 1. 核心结论

用三个方向介绍 QuickFIX/J 的总体思路是正确的：

```text
1. NewOrderSingle 的完整下单流程
2. 领域 Order、FIX Message 与原始 FIX 报文之间的转换
3. 断线、重连、序号恢复与消息重传
```

它们对应 QuickFIX/J 的三层核心能力：

```text
业务接入层
    ↓
消息与协议层
    ↓
Session 会话可靠性层
```

但是，在进入这三条主线前，需要先说明 **QuickFIX/J 的定位与边界**；第三条也不应只称为“断线重连”或“高可用”，而应提升为：

```text
Session 可靠性：心跳、断线、重连、序号恢复与消息重传
```

---

## 2. 第零部分：QuickFIX/J 的定位与边界

### 2.1 QuickFIX/J 所在的位置

```text
交易员 UI / 策略 / API
          │
          ▼
OMS / EMS / 风控 / 订单状态管理
          │
          ▼
Application：业务与 FIX 引擎的边界
          │
          ▼
QuickFIX/J
├─ Session：会话、序号、重连、心跳、重传
├─ Message：FIX 消息对象与字段模型
├─ DataDictionary：协议合法性校验规则
├─ MessageStore：消息与序号持久化
└─ MINA / TCP：网络通信
          │
          ▼
券商 / 交易所 / ECN / 内部撮合服务
```

### 2.2 必须明确的边界

```text
QuickFIX/J 是 Java 的 FIX 协议与会话引擎。

QuickFIX/J 不是：
- 完整 OMS
- 完整 EMS
- 风控系统
- 撮合引擎
- 行情系统
- 系统级高可用平台
```

以示例为例：

```text
Banzai
  = 带 UI 的示例买方交易终端

OrderMatch
  = 带内存订单簿的示例撮合服务

Executor
  = 多版本订单执行服务示例

QuickFIX/J
  = 支撑上述示例进行 FIX 消息收发、解析、校验和会话管理的引擎
```

---

## 3. 主线一：一笔 NewOrderSingle 的端到端业务流程

### 3.1 这一部分要回答的问题

```text
- 业务代码如何使用 QuickFIX/J？
- 一笔订单从哪里产生、如何被发送、如何被接收？
- ExecutionReport 如何回到买方系统？
- QuickFIX/J 与业务实现的边界在哪里？
```

### 3.2 完整业务链路

```text
Banzai（买方 / Initiator）
─────────────────────────────────────────────────────────────
OrderEntryPanel
  │ 用户填写 Symbol、Quantity、Side、Type 等
  ▼
Banzai Order
  │ 本地领域订单模型
  ▼
BanzaiApplication.send(order)
  │ 根据 SessionID 的 BeginString 选择 FIX 版本
  ▼
send42(order)
  │
  ▼
NewOrderSingle (35=D)
  │
  ▼
Session.sendToTarget(...)
  │
  ▼
QuickFIX/J Session
  ├─ 补充 Header
  ├─ 分配 MsgSeqNum(34)
  ├─ 持久化消息与序号
  ├─ 计算 BodyLength(9)
  └─ 计算 CheckSum(10)
  │
  ▼
TCP / Apache MINA
  │
  ▼
OrderMatch（卖方 / Acceptor）
─────────────────────────────────────────────────────────────
TCP / Apache MINA
  │
  ▼
Session.next(...)
  │
  ▼
Application.fromApp(message, sessionID)
  │
  ▼
MessageCracker.crack(message, sessionID)
  │
  ▼
onMessage(NewOrderSingle, sessionID)
  │
  ▼
OrderMatch Order
  │ 卖方 / 撮合服务自己的领域订单模型
  ▼
OrderMatcher / Market
  │
  ▼
ExecutionReport (35=8)
  │
  ▼
Session.sendToTarget(...)
  │
  ▼
BanzaiApplication.fromApp(...)
  │
  ▼
订单状态、成交数量、均价等更新到 Banzai
```

### 3.3 三层职责划分

```text
业务层
├─ Banzai Order
├─ UI、OMS、风控、策略
├─ OrderMatch 的撮合逻辑
└─ 订单状态机

QuickFIX/J 引擎层
├─ Application
├─ Session
├─ MessageCracker
├─ MessageStore
├─ SocketInitiator
└─ SocketAcceptor

FIX 协议层
├─ NewOrderSingle (35=D)
├─ ExecutionReport (35=8)
├─ Header / Body / Trailer
├─ Tag=Value
└─ DataDictionary
```

### 3.4 对外介绍时的关键结论

> 一笔订单的业务含义由 OMS、EMS、风控、撮合等业务系统决定；QuickFIX/J 负责把该业务意图可靠地表达为 FIX 消息，并在对手方之间安全、正确地传递。

---

## 4. 主线二：领域对象、FIX Message 与原始报文的转换

### 4.1 更严谨的表达

不要笼统地说：

```text
Order 变成 FIX 协议，再变成 Order。
```

建议使用下面的表达：

```text
买方领域 Order
  → FIX 强类型消息 NewOrderSingle
  → 原始 FIX Tag=Value 报文
  → 接收方强类型消息 NewOrderSingle
  → 卖方领域 Order
```

### 4.2 对象与报文的完整转换链路

```text
Banzai / 买方
─────────────────────────────────────────────────────────────
领域对象
quickfix.examples.banzai.Order
  │
  │ BanzaiApplication.send42(order)
  ▼
协议对象
quickfix.fix42.NewOrderSingle
  │
  │ Session 编码并添加会话字段
  ▼
原始 FIX 报文
8=FIX.4.2|9=...|35=D|49=BANZAI|56=EXEC|34=2|
11=ORD-001|55=AAPL|54=1|38=100|40=2|44=123.45|10=...|
  │
  ▼
TCP 字节流

OrderMatch / 卖方
─────────────────────────────────────────────────────────────
TCP 字节流
  │
  ▼
原始 FIX 报文
  │
  │ 根据 BeginString(8) 与 MsgType(35) 选择消息类型
  ▼
协议对象
quickfix.fix42.NewOrderSingle
  │
  │ Application.onMessage(...)
  ▼
领域对象
quickfix.examples.ordermatch.Order
  │
  ▼
撮合 / 执行 / 风控等卖方业务逻辑
```

> 上图中的 `|` 仅用于展示；实际 FIX 字段分隔符是 SOH 字符（`\001`）。

### 4.3 三种“订单”不是同一个对象

```text
Banzai Order
  └─ Banzai 示例的本地业务对象，服务于 UI 和订单状态维护。

NewOrderSingle
  └─ FIX 规范定义的协议消息，MsgType(35)=D。

OrderMatch Order
  └─ OrderMatch 示例的内部领域对象，服务于订单簿与撮合。
```

因此，QuickFIX/J 不会自动把任意业务类映射为 FIX 消息。正确理解是：

```text
业务代码负责：
领域对象 ↔ FIX Message 的映射。

QuickFIX/J 负责：
FIX Message ↔ 原始 FIX 报文 ↔ 网络传输的处理。
```

### 4.4 QuickFIX/J 消息模型的核心对象

```text
FieldMap
  ├─ 保存 Tag -> Value 字段
  └─ 保存重复组（Group）

Message
  ├─ Header
  ├─ Body
  └─ Trailer

DataDictionary
  ├─ 字段类型
  ├─ 必填字段
  ├─ 枚举值
  ├─ 消息定义
  └─ 重复组定义

DefaultMessageFactory
  └─ 根据 BeginString(8) 与 MsgType(35) 创建正确的强类型消息。
```

### 4.5 Header、Body、Trailer

```text
Header：会话和路由信息
8=FIX.4.2        BeginString
35=D             MsgType：NewOrderSingle
49=BANZAI        SenderCompID
56=EXEC          TargetCompID
34=2             MsgSeqNum

Body：业务字段
11=ORD-001       ClOrdID
55=AAPL          Symbol
54=1             Side（Buy）
38=100           OrderQty
40=2             OrdType（Limit）
44=123.45        Price

Trailer：完整性校验
10=...           CheckSum
```

### 4.6 对外介绍时的关键结论

> 领域 `Order` 表达“业务想做什么”，`NewOrderSingle` 表达“FIX 协议如何表达这笔订单”，原始 Tag=Value 报文表达“它如何在线路上传输”。三者职责不同，不能混为一谈。

---

## 5. 主线三：Session 可靠性——心跳、重连、序号与重传

### 5.1 为什么不只讲“断线重连”

仅有 Socket 重连不足以保证 FIX 会话正确：

```text
连接恢复了
≠
双方确认没有丢消息、没有错序、没有重复处理消息
```

FIX Session 的关键问题是：

```text
网络中断后，双方如何恢复到一致的消息序列状态？
```

因此，第三条主线的准确名称应为：

```text
Session 可靠性：心跳、断线、重连、序号恢复与消息重传
```

### 5.2 正常会话中的可靠性机制

```text
每个逻辑 Session
├─ 每个方向独立维护 MsgSeqNum(34)
├─ MessageStore 持久化消息和序号
├─ Heartbeat(35=0) 保持活性
└─ TestRequest(35=1) 探测疑似失联的对端
```

### 5.3 断线恢复流程

```text
网络连接中断
  │
  ▼
Initiator 按 ReconnectInterval 重连
  │
  ▼
双方重新建立 TCP 连接
  │
  ▼
重新发送 Logon (35=A)
  │
  ▼
双方检查接收序号与期望序号
  │
  ├─ 序号连续
  │    │
  │    ▼
  │  正常恢复业务消息处理
  │
  └─ 发现序号缺口
       │
       ▼
     ResendRequest (35=2)
       │
       ▼
     对端从 MessageStore 查找历史消息
       │
       ├─ 可以重放
       │    │
       │    ▼
       │  重发业务消息
       │  PossDupFlag(43)=Y
       │
       └─ 不应或无法逐条重放
            │
            ▼
          SequenceReset (35=4)
            │
            ▼
          填补 Gap，推进期望序号
            │
            ▼
          双方序号重新一致，Session 恢复
```

### 5.4 关键概念表

| 概念 | 含义 | 要解决的问题 |
|---|---|---|
| `MsgSeqNum(34)` | 单方向消息序列号 | 如何发现乱序或缺失消息？ |
| `MessageStore` | 保存消息和收发序号 | 重连后如何知道历史状态？ |
| `FileStore` | 基于文件的 Store 实现 | 学习和本地演示中如何持久化状态？ |
| `Heartbeat(35=0)` | 定期心跳 | 空闲时如何确认连接存活？ |
| `TestRequest(35=1)` | 主动健康检查 | 怀疑对端失联时如何探测？ |
| `ResendRequest(35=2)` | 请求缺失序号范围 | 发现缺号后如何补消息？ |
| `PossDupFlag(43)` | 重发标识 | 收到重发业务消息时如何识别？ |
| `SequenceReset(35=4)` | 序号重置或 Gap Fill | 不逐条重发时如何推进序号？ |
| `ResetSeqNumFlag(141)` | Logon 时重置序号标志 | 在双方明确同意时如何从头开始？ |

### 5.5 应用层仍需具备幂等能力

即使 FIX 有序号和 `PossDupFlag(43)`，业务系统也不能假设“同一业务结果绝不会再次出现”。

```text
应用层通常需要根据：
- ClOrdID
- OrigClOrdID
- ExecID
- OrderID

识别重复请求、重复执行回报或重发消息，避免重复记账、重复成交、重复更新状态。
```

### 5.6 Session 可靠性与系统级高可用的边界

```text
QuickFIX/J 可以提供：
- 会话级自动重连
- 收发序号管理
- 心跳与失联探测
- 消息重传
- 本地消息和序号持久化

QuickFIX/J 不会自动提供：
- 多实例双活
- 主备切换编排
- 分布式一致性
- 多机共享的 Session 状态
- 容灾与网络冗余
- 监控、告警、容量治理
```

因此：

```text
QuickFIX/J 的 Session 可靠性
≠
完整交易系统的高可用架构
```

完整生产方案通常还需要：

```text
进程监控 / 容器编排
主备或集群部署
可恢复或共享的持久化 Store
网络冗余
监控、告警、审计
业务幂等与重复回报处理
```

### 5.7 对外介绍时的关键结论

> QuickFIX/J 不只是“把消息发到 Socket 上”。它通过 Session、序号、Store、心跳和重传机制，使断线后的通信可以恢复到双方可验证的一致状态。

---

## 6. 推荐的分享结构

适合 30～45 分钟技术分享的结构：

```text
第 0 部分：QuickFIX/J 的定位与边界                 3～5 分钟
第 1 部分：一笔 NewOrderSingle 的端到端业务流程     10～15 分钟
第 2 部分：领域对象、FIX Message、原始报文的转换    10～15 分钟
第 3 部分：Session 可靠性与恢复机制                 10～15 分钟
第 4 部分：总结与源码、实验入口                     3～5 分钟
```

### 6.1 每部分建议包含的内容

| 部分 | 架构图 | 示例 | 建议源码入口 | 可演示实验 |
|---|---|---|---|---|
| 定位与边界 | 完整交易系统与 QuickFIX/J 的层次图 | Banzai、OrderMatch、Executor 的角色 | `quickfix.Application` | 展示 Initiator / Acceptor 配置 |
| 下单流程 | Banzai → OrderMatch → ExecutionReport | `35=D`、`35=8` | `BanzaiApplication.send42`、OrderMatch `onMessage` | Banzai 登录、下单、收到回报 |
| 消息转换 | Order → Message → FIX 文本 → Message → Order | Header / Body / Trailer | `Message`、`FieldMap`、`DataDictionary`、`MessageCracker` | 打印一条真实 FIX 报文 |
| Session 可靠性 | Disconnect → Logon → Resend → Recover | `35=A`、`35=2`、`35=4` | `Session.next()`、`MessageStore` | 停止并重启一端，观察重连与序号 |

---

## 7. 最终收束图

```text
                    QuickFIX/J 的三层理解框架

业务层
Banzai Order / OrderMatch Order / OMS / EMS / 风控 / 撮合
                              │
                              ▼
消息与协议层
NewOrderSingle / ExecutionReport / Message / DataDictionary / MessageCracker
                              │
                              ▼
Session 可靠性层
Session / MsgSeqNum / Store / Heartbeat / Resend / Reconnect
                              │
                              ▼
网络层
SocketInitiator / SocketAcceptor / Apache MINA / TCP
```

---

## 8. 一句话版本

```text
用一笔订单说明“业务怎么接入”，
用消息转换说明“协议怎么落地”，
用 Session 恢复说明“通信为什么可靠”。
```

最终结论：

> 以“下单业务链路、消息转换机制、Session 可靠性”作为三条主线是一个高质量的 QuickFIX/J 介绍框架。补上 QuickFIX/J 的边界，并把“断线重连”扩展为“序号、存储与重传驱动的会话恢复”，这套材料就能从学习笔记提升为完整的架构介绍。
