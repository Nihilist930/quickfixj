# QuickFIX/J 学习地图

> 类型：长期复习用结构图  
> 适用范围：QuickFIX/J 入门到理解消息链路、会话机制、业务接入边界  
> 最后更新：2026-08-17

本文件不是按问答整理，而是按“架构层次 + 关系图 + 调用链”组织，适合以后复习时快速建立整体脑图。

---

## 1. 总览地图

```text
                                  QuickFIX/J 学习地图
┌────────────────────────────────────────────────────────────────────────────┐
│ 目标：回答 5 个核心问题                                                    │
│                                                                            │
│ 1. QuickFIX/J 在交易系统里处于哪一层？                                     │
│ 2. 引擎如何把消息交给业务层？                                              │
│ 3. 通用 Message 如何进入强类型 onMessage(...)？                            │
│ 4. 一条消息属于哪个 Session？                                              │
│ 5. 放到买方/卖方系统里，行情、委托、回报分别在什么位置？                  │
└────────────────────────────────────────────────────────────────────────────┘

            ┌──────────────────┐
            │ 交易系统整体视角  │
            └────────┬─────────┘
                     │
                     ▼
            ┌──────────────────┐
            │ QuickFIX/J 的定位 │
            └────────┬─────────┘
                     │
                     ▼
            ┌──────────────────┐
            │ Application 边界  │
            └────────┬─────────┘
                     │
                     ▼
            ┌──────────────────┐
            │ MessageCracker    │
            │ 强类型分派        │
            └────────┬─────────┘
                     │
                     ▼
            ┌──────────────────┐
            │ SessionID 会话身份│
            └────────┬─────────┘
                     │
                     ▼
            ┌──────────────────┐
            │ 买方/卖方使用视角 │
            └──────────────────┘
```

---

## 2. QuickFIX/J 在交易系统中的位置

```text
完整交易系统 / 柜台 / OMS
├─ 行情接入
├─ 策略/信号
├─ 风控
├─ 订单管理
├─ 持仓/账户/清算
└─ 对外交易接入层
   └─ QuickFIX/J 常落在这里
      ├─ FIX 会话管理
      ├─ 报文解析与生成
      ├─ DataDictionary 校验
      ├─ 序号 / 重发 / 心跳 / Logon / Logout
      └─ 向业务层回调 Application
```

### 记忆结论

```text
QuickFIX/J = FIX 协议引擎 / 会话引擎 / 报文引擎
QuickFIX/J ≠ 完整交易系统 / 完整柜台 / OMS
```

---

## 3. 引擎和业务的分界线：Application

关键文件：

- `quickfixj-core/src/main/java/quickfix/Application.java`

```text
Application
├─ 生命周期回调
│  ├─ onCreate   : Session 对象创建
│  ├─ onLogon    : FIX 登录成功
│  └─ onLogout   : 会话离线
│
├─ 管理消息（Admin）
│  ├─ toAdmin    : 管理消息发出前
│  └─ fromAdmin  : 收到管理消息后
│
└─ 业务消息（App）
   ├─ toApp      : 业务消息发出前
   └─ fromApp    : 收到业务消息后（统一入口）
```

### 记忆结论

```text
引擎负责：连接、会话、解析、校验、收发
业务负责：收到消息后做什么处理

fromApp = 业务消息统一入口
toApp   = 业务消息发送前钩子
```

---

## 4. 通用 Message 如何进入强类型处理：MessageCracker

关键文件：

- `quickfixj-core/src/main/java/quickfix/MessageCracker.java`

### 4.1 初始化阶段

```text
Application extends MessageCracker
        │
        ▼
initialize(this)
        │
        ▼
扫描所有 public 方法
        │
        ▼
找到符合签名的方法：
onMessage(某消息类型, SessionID)
        │
        ▼
建立映射表：
消息 Java 类 -> Invoker
```

例如：

```text
quickfix.fix42.NewOrderSingle  -> onMessage(NewOrderSingle, SessionID)
quickfix.fix42.ExecutionReport -> onMessage(ExecutionReport, SessionID)
```

### 4.2 运行时阶段

```text
fromApp(Message msg, SessionID sid)
        │
        ▼
crack(msg, sid)
        │
        ▼
invokers.get(msg.getClass())
        │
        ▼
找到对应 Invoker
        │
        ▼
method.invoke(target, msg, sid)
        │
        ▼
onMessage(具体消息类型, SessionID)
```

### 4.3 OrderMatch：从 `crack(...)` 到 `onMessage(...)` 的完整流程

相关源码：

```text
业务处理器
- quickfixj-examples/ordermatch/src/main/java/
  quickfix/examples/ordermatch/Application.java

强类型分派器
- quickfixj-core/src/main/java/quickfix/MessageCracker.java

按 FIX 版本创建消息对象
- quickfixj-core/src/main/java/quickfix/DefaultMessageFactory.java
```

#### A. 启动阶段：扫描并登记处理器

```text
创建 OrderMatch Application
        │
        ▼
Application extends MessageCracker
        │
        ▼
MessageCracker 构造函数
initialize(this)
        │
        ▼
扫描 Application 的所有 public 方法
        │
        ▼
筛选处理器方法：

方法名为 onMessage，或标有 @Handler
      +
恰好两个参数
      +
参数一是 Message 的子类
      +
参数二是 SessionID
        │
        ▼
发现：
onMessage(NewOrderSingle, SessionID)
        │
        ▼
读取第一个参数的 Java 类型：
quickfix.fix42.NewOrderSingle.class
        │
        ▼
建立 invokers 映射表：

quickfix.fix42.NewOrderSingle.class
        │
        └──> Application.onMessage(NewOrderSingle, SessionID)

quickfix.fix42.OrderCancelRequest.class
        │
        └──> Application.onMessage(OrderCancelRequest, SessionID)

quickfix.fix42.MarketDataRequest.class
        │
        └──> Application.onMessage(MarketDataRequest, SessionID)
```

#### B. 收到订单阶段：原始 FIX 报文变成强类型对象

```text
Banzai 发送原始 FIX 报文

8=FIX.4.2 | 35=D | 49=BANZAI | 56=EXEC | 11=... | 55=AAPL | ...
        │           │
        │           └─ MsgType：NewOrderSingle
        │
        └─ BeginString：FIX 4.2
        │
        ▼
QuickFIX/J 解析器
        │
        ▼
DefaultMessageFactory
        │
        ▼
选择版本工厂：
quickfix.fix42.MessageFactory
        │
        ▼
根据 35=D 创建具体对象：
new quickfix.fix42.NewOrderSingle()
        │
        ▼
填充 Header / Body / Trailer 中的字段
        │
        ▼
Session.next(message)
        │
        ▼
OrderMatch.Application.fromApp(Message, SessionID)
```

> 此时变量的声明类型是 `Message`，但对象实际类型已是 `quickfix.fix42.NewOrderSingle`。

#### C. 分派阶段：`crack(...)` 查表并反射调用

```text
Application.fromApp(Message message, SessionID sessionId)
        │
        ▼
crack(message, sessionId)
        │
        ▼
message.getClass()
        │
        ▼
quickfix.fix42.NewOrderSingle.class
        │
        ▼
invokers.get(quickfix.fix42.NewOrderSingle.class)
        │
        ▼
找到启动时注册的 Invoker：

目标对象：OrderMatch Application 实例
目标方法：onMessage(NewOrderSingle, SessionID)
        │
        ▼
method.invoke(target, message, sessionId)
        │
        ▼
等价于概念上的直接调用：

this.onMessage((NewOrderSingle) message, sessionId)
        │
        ▼
OrderMatch.Application.onMessage(NewOrderSingle, SessionID)
        │
        ▼
读取 11 / 55 / 54 / 40 / 38 等订单字段
        │
        ▼
processOrder(order)
```

#### D. 没有匹配处理器时的兜底路径

```text
收到某种业务消息
        │
        ▼
message.getClass() 无法在 invokers 中找到处理器
        │
        ▼
MessageCracker.onMessage(Message, SessionID)
        │
        ▼
抛出 UnsupportedMessageType
        │
        ▼
QuickFIX/J 生成对应的业务拒绝处理
```

### 记忆结论

```text
35=D 的职责：
帮助解析器创建 quickfix.fix42.NewOrderSingle 对象

crack(...) 的职责：
按 message.getClass() 在 invokers 映射表中找处理器

onMessage(...) 的执行方式：
MessageCracker 通过反射 method.invoke(...) 调用

因此：
35=D -> 强类型 Message 对象 -> message.getClass() -> onMessage(具体类型,...)
```

---

## 5. SessionID：一条消息属于哪个逻辑会话

关键文件：

- `quickfixj-base/src/main/java/quickfix/SessionID.java`
- `quickfixj-base/src/main/java/quickfix/MessageUtils.java`

### 5.1 SessionID 的组成

```text
SessionID
= BeginString
+ SenderCompID
+ SenderSubID      (可选)
+ SenderLocationID (可选)
+ TargetCompID
+ TargetSubID      (可选)
+ TargetLocationID (可选)
+ SessionQualifier (可选)
```

最常见的简化形式：

```text
FIX.4.2:BANZAI->EXEC
```

### 5.2 在当前学习环境里的具体值

```text
Banzai 本地会话:
  FIX.4.2:BANZAI->EXEC

OrderMatch 本地会话:
  FIX.4.2:EXEC->BANZAI
```

### 5.3 SessionID 的两种来源

```text
A. 本地 Session 创建时
   来源：配置文件
   例：BeginString / SenderCompID / TargetCompID

B. 处理具体消息时
   来源：消息头
   例：8 / 49 / 56 / ...
```

### 5.4 一个容易混淆但最重要的点：方向

```text
报文头可能是：
49=BANZAI, 56=EXEC

从报文本身正向提取：
FIX.4.2:BANZAI->EXEC

但在 OrderMatch 本地匹配 Session 时，常常要看反向：
FIX.4.2:EXEC->BANZAI
```

### 记忆结论

```text
SessionID 不是网络连接号，而是 FIX 会话的逻辑身份证
```

---

## 6. 买方系统视角：行情 / 委托 / 回报 / fromApp / toApp

```text
买方系统
├─ 行情输入
│  ├─ 可能来自 FIX 行情 Session
│  └─ 也可能来自非 FIX 行情源
│
├─ 策略/信号计算
│
├─ 风控 / 约束检查
│
├─ 委托发送
│  └─ 通过交易 FIX Session 发给卖方/券商
│
└─ 回报处理
   └─ 从交易 FIX Session 收 ExecutionReport
```

### 映射到 QuickFIX/J

```text
行情 Session
  fromApp(MarketData...)
    -> 收行情
    -> 更新行情状态
    -> 触发策略判断

交易 Session
  toApp(NewOrderSingle...)
    -> 发委托前钩子

交易 Session
  fromApp(ExecutionReport...)
    -> 收订单回报
    -> 更新订单状态 / 成交 / 持仓
```

### 记忆结论

```text
fromApp = 收业务消息
toApp   = 发业务消息前

但 fromApp 不只收一种消息，可能是：
- 行情
- 回报
- 报价
- 其他业务消息

具体取决于这条 Session 的用途
```

---

## 7. 卖方 / 接收方视角：以 OrderMatch 为例

```text
买方 / Banzai
  toApp(NewOrderSingle)
        │
        ▼
   Session.sendToTarget(...)
        │
        ▼
      网络发送
        │
        ▼
卖方 / OrderMatch
  fromApp(Message, SessionID)
        │
        ▼
  crack(message, sessionID)
        │
        ▼
  onMessage(NewOrderSingle, SessionID)
        │
        ▼
  业务处理 / 生成 ExecutionReport
        │
        ▼
  Session.sendToTarget(...)
        │
        ▼
买方 fromApp(ExecutionReport, SessionID)
```

### 记忆结论

```text
买方视角：
  toApp   = 发单
  fromApp = 收回报

卖方视角：
  fromApp = 收订单
  toApp   = 发回报前
```

---

## 8. 一笔订单的端到端调用链

```text
Banzai / 买方侧
    │
    │ 构造 NewOrderSingle
    ▼
toApp(message, sessionID)
    │
    ▼
Session.sendToTarget(...)
    │
    ▼
QuickFIX/J Session / IO / 编码发送
    │
    ▼
网络
    │
    ▼
OrderMatch / 卖方侧收到原始 FIX 报文
    │
    ▼
MessageSessionUtils.parse(...)
    │
    ├─ 读取 8=FIX.4.2
    ├─ 读取 35=D
    └─ 创建 quickfix.fix42.NewOrderSingle
    │
    ▼
Session.next(message)
    │
    ▼
application.fromApp(message, sessionID)
    │
    ▼
crack(message, sessionID)
    │
    ▼
onMessage(NewOrderSingle, sessionID)
    │
    ▼
业务处理 / 生成 ExecutionReport
    │
    ▼
Session.sendToTarget(...)
    │
    ▼
买方侧 fromApp(ExecutionReport, sessionID)
    │
    ▼
更新订单状态 / 成交结果
```

---

## 9. 今天问题的串联主线

```text
交易系统是分层的
        │
        ▼
QuickFIX/J 负责 FIX 协议 / 会话 / 报文引擎
        │
        ▼
业务层通过 Application 接入
        │
        ▼
fromApp 是统一业务消息入口
        │
        ▼
MessageCracker 负责进入强类型 onMessage(...)
        │
        ▼
SessionID 标识“这条消息属于哪个会话”
        │
        ▼
放到买方 / 卖方系统中
  就能看清行情、委托、回报分别走哪条链路
```

---

## 10. OrderMatch 与 Executor：两个 Acceptor 示例的定位对照

```text
                              Banzai（Initiator）
                                      │
                  ┌───────────────────┴───────────────────┐
                  │                                       │
                  ▼                                       ▼
        OrderMatch（单版本撮合示例）             Executor（多版本执行示例）
```

### 10.1 连接与协议能力

```text
OrderMatch
├─ 角色：Acceptor
├─ 默认协议：FIX.4.2
├─ 监听端口：9876
├─ 本地 Session：FIX.4.2:EXEC->BANZAI
└─ 配对学习配置：Banzai FIX.4.2 连接 9876

Executor
├─ 角色：Acceptor
├─ 默认协议：FIX.4.0、4.1、4.2、4.3、4.4、FIXT.1.1/FIX.5.0
├─ 监听端口：9876 到 9881（每版本一个端口）
├─ 本地身份：EXEC->BANZAI
└─ 可直接与默认 banzai.cfg 的多 Session 配对
```

```text
Banzai 默认多版本连接

FIX.4.0:BANZAI->EXEC  ── 9876 ──> Executor FIX.4.0:EXEC->BANZAI
FIX.4.1:BANZAI->EXEC  ── 9877 ──> Executor FIX.4.1:EXEC->BANZAI
FIX.4.2:BANZAI->EXEC  ── 9878 ──> Executor FIX.4.2:EXEC->BANZAI
FIX.4.3:BANZAI->EXEC  ── 9879 ──> Executor FIX.4.3:EXEC->BANZAI
FIX.4.4:BANZAI->EXEC  ── 9880 ──> Executor FIX.4.4:EXEC->BANZAI
FIXT.1.1:BANZAI->EXEC ── 9881 ──> Executor FIXT.1.1:EXEC->BANZAI
```

> `OrderMatch` 与 `Executor` 都会占用 `9876`；两者不能同时使用默认端口运行。

### 10.2 业务行为对照

```text
OrderMatch：模拟订单簿与撮合

Banzai 发 Buy Limit AAPL
        │
        ▼
OrderMatch 将订单放入内存订单簿
        │
        ▼
先回 ExecutionReport：New
        │
        ▼
等待可匹配的反向订单
        │
        ▼
价格/方向匹配
        │
        ▼
双方订单获得 Filled 或 Partially Filled 回报
```

```text
Executor：模拟执行服务

Banzai 发 NewOrderSingle
        │
        ▼
Executor 校验订单类型与市场价格配置
        │
        ▼
先回 ExecutionReport：New
        │
        ▼
订单达到可执行条件？
   ┌────┴────┐
   │ 是      │ 否
   ▼         ▼
回 Filled   保持 New
```

```text
核心差异

OrderMatch：成交依赖另一笔相反方向、可匹配的订单
Executor  ：成交依赖模拟价格 / AlwaysFillLimitOrders 等执行条件
```

### 10.3 源码结构与学习价值

```text
OrderMatch Application
├─ FIX.4.2 的 onMessage(NewOrderSingle)
├─ OrderMatcher：维护内存订单簿
├─ Order：领域订单对象
└─ 重点：单一协议版本下的订单接收、撮合、撤单、市场数据请求

Executor Application
├─ onMessage(fix40.NewOrderSingle)
├─ onMessage(fix41.NewOrderSingle)
├─ onMessage(fix42.NewOrderSingle)
├─ onMessage(fix43.NewOrderSingle)
├─ onMessage(fix44.NewOrderSingle)
├─ onMessage(fix50.NewOrderSingle)
├─ validateOrder：订单类型与市场价格校验
└─ 重点：同一业务语义在多 FIX 版本中如何被强类型分派和生成回报
```

### 记忆结论

```text
学习 OrderMatch：
理解一条 FIX.4.2 订单如何进入业务逻辑、订单簿和撮合流程

学习 Executor：
理解多协议版本如何由 MessageFactory + MessageCracker
分派到不同的 onMessage(fixXX.NewOrderSingle, ...) 方法

选择建议：
单版本端到端消息链路 -> OrderMatch
多版本差异、订单校验与模拟执行 -> Executor
```

### 10.4 重要环节：断线重连、MsgSeqNum 与会话恢复

```text
┌────────────────────────────────────────────────────────────────────────────┐
│                 FIX Session 断线重连与会话恢复图                            │
└────────────────────────────────────────────────────────────────────────────┘

 Banzai / Initiator                                      EXEC / Acceptor
 ─────────────────                                      ─────────────────
        │                                                      │
        │ ① 正常运行：双方各自维护一条 MsgSeqNum 序列            │
        │                                                      │
        │  发送方向：Banzai -> EXEC                            │
        │  接收方向：EXEC -> Banzai                            │
        │                                                      │
        │ ② TCP 连接断开                                       │
        │◄────────────── FileStore 保留会话状态 ──────────────►│
        │                                                      │
        │ ③ Banzai 按 ReconnectInterval 自动重连               │
        │                                                      │
        │── 35=A, 34=54 : Logon ──────────────────────────────►│
        │◄─ 35=A, 34=55 : Logon ───────────────────────────────│
        │                                                      │
        │ ④ Banzai 期望 52，却先收到 55                          │
        │ 发现缺口：52~54                                      │
        │                                                      │
        │── 35=2, 34=55, 7=52, 16=0 ──────────────────────────►│
        │   ResendRequest：请从 52 补到最新                        │
        │                                                      │
        │◄─ 35=8, 34=52, 43=Y ─────────────────────────────────│
        │   重发历史 ExecutionReport                             │
        │                                                      │
        │◄─ 35=4, 34=53, 43=Y, 36=57, 123=Y ────────────────────│
        │   SequenceReset / GAP_FILL：下一条从 57 开始            │
        │                                                      │
        │ ⑤ 双方补齐或跳过缺口，推进到一致的下一个序号              │
        │                                                      │
        └────────────── 会话恢复完成，继续业务消息 ──────────────┘
```

#### `MsgSeqNum(34)` 的双向序列

```text
Banzai -> EXEC                                      EXEC -> Banzai
独立序列：34=1,2,3,...,54,55                        独立序列：34=1,2,3,...,55,56

每个方向都有自己的：
- next sender sequence number：下一条要发送的序号
- next target sequence number：下一条期望接收的序号
```

```text
每条 FIX 报文 Header

8=FIX.4.2 | 9=... | 35=... | 34=<MsgSeqNum> | 49=... | 56=...
                                      ▲
                                      └─ 当前方向的消息序号
```

#### 真实日志与恢复动作对应表

| 日志片段 | 含义 | 恢复动作 |
|---|---|---|
| `35=A, 34=54` | Banzai 重连后的 Logon 序号 | 证明 Store 中的发送序号没有重置 |
| `35=A, 34=55` | EXEC 重连后的 Logon 序号 | 对端也在继续旧会话序列 |
| `expecting 52 but received 55` | 收到高于期望值的消息 | 发现 52~54 缺口，先缓存 55 |
| `35=2, 7=52, 16=0` | ResendRequest | 请求从 52 补到最新 |
| `35=8, 34=52, 43=Y` | 补发历史业务消息 | `43=Y` 表示可能重复/恢复补发 |
| `35=4, 34=53, 36=57, 123=Y` | SequenceReset/GAP_FILL | 跳过 53~56，下一条从 57 开始 |

#### 会话恢复的源码锚点

```text
quickfixj-core/src/main/java/quickfix/Session.java

initializeHeader(...)
  -> header.setInt(MsgSeqNum.FIELD, getExpectedSenderNum())
  -> 将 MessageStore 中的发送序号写入 34

generateLogon(...)
  -> 重连后生成并发送 Logon

nextLogon(...)
  -> 接收 Logon，比较双方会话状态与序号

nextResendRequest(...)
  -> 处理对方的 35=2 ResendRequest

SequenceReset / GAP_FILL
  -> 使用 35=4、36=NewSeqNo、123=Y 跳过无需重发的序号区间

disconnect(...)
  -> 断开连接并进入后续重连状态
```

#### 断线重连学习实验

```text
实验 1：正常恢复
启动双方 -> 发几条消息 -> 停止一端 -> 等待自动重连 -> 恢复对端
观察：Logon、34 是否续上、ResendRequest、SequenceReset、43=Y

实验 2：制造序号不一致
停止双方 -> 只删除一端 FileStore -> 重启双方
观察：双方如何通过序号检查与恢复消息重新建立一致状态
```

> 这是从“能建立 FIX 连接”进入“生产环境高可用会话管理”的关键环节：断线重连的本质不是重新连接 TCP，而是恢复 Session 状态、校准双向序号并恢复缺失消息。

---


### 10.1 五分钟复习

```text
1. 看第 1 节总览地图
2. 看第 3 节 Application
3. 看第 4 节 MessageCracker
4. 看第 5 节 SessionID
5. 看第 8 节端到端调用链
```

### 10.2 二十分钟复习

```text
1. 第 2 节：QuickFIX/J 在交易系统中的位置
2. 第 3 节：Application 的 7 个回调
3. 第 4 节：crack / invoke 的分派机制
4. 第 5 节：SessionID 的来源与方向
5. 第 6、7 节：买方 / 卖方视角对照
6. 第 8 节：一笔订单完整链路
```

### 10.3 继续深入时的源码锚点

```text
Application 边界
- quickfixj-core/src/main/java/quickfix/Application.java

强类型分派
- quickfixj-core/src/main/java/quickfix/MessageCracker.java

消息与会话身份
- quickfixj-base/src/main/java/quickfix/SessionID.java
- quickfixj-base/src/main/java/quickfix/MessageUtils.java

Session 创建与发送
- quickfixj-core/src/main/java/quickfix/DefaultSessionFactory.java
- quickfixj-core/src/main/java/quickfix/Session.java
```

---

## 11. 当前阶段与下一步

```text
当前已经形成的稳定认知：
- QuickFIX/J 的系统定位
- Application 的边界意义
- fromApp / toApp 的职责
- MessageCracker 的分派机制
- SessionID 的组成与方向
- 买方 / 卖方视角下的消息流

下一步最自然的深入方向：
- FieldMap
- Message
- DataDictionary
- MessageSessionUtils
- DefaultMessageFactory
```

---

## 12. 一页压缩版脑图

```text
QuickFIX/J
├─ 是 FIX 协议引擎，不是完整交易系统
├─ Application 是引擎和业务的边界
│  └─ fromApp 是统一业务入口
├─ MessageCracker 负责强类型分派
│  └─ Message -> onMessage(具体类型,...)
├─ SessionID 标识逻辑会话
│  └─ BeginString + Sender + Target + ...
└─ 在交易系统里
   ├─ 买方行情 Session: fromApp 收行情
   ├─ 买方交易 Session: toApp 发单
   ├─ 买方交易 Session: fromApp 收回报
   └─ 卖方交易 Session: fromApp 收订单并回报
```
