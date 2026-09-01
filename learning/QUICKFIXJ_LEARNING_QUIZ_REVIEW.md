# QuickFIX/J 学习阶段测验复盘

> 用途：复盘当前对 `QUICKFIXJ_INTRODUCTION_GUIDE.md` 主要内容的理解程度。
>
> 当前状态：已完成 9/10 题，第 10 题在本次测验中未正式作答。
>
> 评分方式：每题 10 分，共 100 分。

---

## 一、成绩总览

| 题号 | 考察主题 | 得分 |
|---|---|---:|
| 1 | QuickFIX/J 的定位 | 7.5 / 10 |
| 2 | Banzai、Executor、OrderMatch 的角色关系 | 8.5 / 10 |
| 3 | NewOrderSingle 的接收与分发链路 | 4.5 / 10 |
| 4 | ClOrdID、OrderID、ExecID | 2.5 / 10 |
| 5 | FieldMap 与 Group | 7.5 / 10 |
| 6 | Logon 与用户名密码认证 | 8.5 / 10 |
| 7 | Heartbeat 与 TestRequest | 9.0 / 10 |
| 8 | MsgSeqNum、MessageStore、FileStore | 8.0 / 10 |
| 9 | Dynamic Session 与 configureDynamicSessions | 6.5 / 10 |
| 10 | Session 可靠性主线总结 | 未作答 |
| **合计** | 仅统计已作答题目 | **62.5 / 90** |

### 换算结果

```text
已作答题目得分：62.5 / 90 ≈ 69.4 / 100
若按完整 100 分试卷暂计：62.5 / 100
```

### 阶段评价

当前学习状态可以概括为：

```text
整体架构已经建立，Session 可靠性相关概念掌握较好；
但订单标识体系、消息接收链路和动态 Session 的实现边界还需要进一步压实。
```

当前阶段评级：

```text
B-
```

---

# 二、逐题复盘

## 第 1 题：QuickFIX/J 的定位

### 题目

请说明：

```text
QuickFIX/J 到底是什么？
它解决的是哪一层问题？
它不是什么？
```

### 你的回答

> quickfixj是一个底层的协议、连接机制，它解决的是不同机构间信息通信规则不一致的问题，它不包含具体业务层的实现。

### 得分

```text
7.5 / 10
```

### 评价

答对的部分：

- 认识到 QuickFIX/J 位于较底层。
- 认识到它不包含具体业务层实现。
- 理解它用于解决交易双方之间的标准化通信问题。

需要修正的部分：

- QuickFIX/J 不只是“连接机制”，还包括 FIX 消息编解码、Session 管理、消息分发、序号管理、心跳、重连和重传等。
- QuickFIX/J 不是自己发明通信规则，而是实现 FIX 标准，使通信双方能够按照共同协议可靠通信。

### 标准回答

```text
QuickFIX/J 是一个 FIX 协议引擎，主要负责 FIX 会话管理、消息编解码、路由分发，以及心跳、断线重连、序号恢复和消息重传等可靠性机制。它解决的是交易双方基于 FIX 标准进行稳定通信的问题。QuickFIX/J 不是完整的 OMS、EMS 或交易系统，也不负责具体的策略、风控、订单管理和撮合业务。
```

### 记忆重点

```text
QuickFIX/J = FIX 协议引擎 + Session 引擎 + 消息引擎
不是完整的业务交易系统。
```

---

## 第 2 题：Banzai、Executor、OrderMatch 的角色关系

### 题目

请说明：

```text
Banzai、Executor、OrderMatch
分别更像真实交易系统里的哪一类角色？
它们三者之间是什么关系？
```

### 你的回答

> banzai是一个模仿买方机构的极简UI下单及订单管理软件，executor是一个极简的卖方接受买方下单并路由至交易所的模拟，ordermatch是一个交易所的极简搓合模拟；三者间在实际应用中的关系应该是banzai->executor->ordermtach，banzai下单给executor,executor发给ordermatch。

### 得分

```text
8.5 / 10
```

### 评价

答对的部分：

- `Banzai` 对应买方机构的下单和订单管理前端。
- `Executor` 对应卖方或券商执行端。
- `OrderMatch` 对应交易所或撮合场所。
- 真实生产架构中，订单方向通常是：

```text
买方 -> 卖方 / 券商 -> 交易所 / 撮合场所
```

需要区分的部分：

```text
真实生产架构中的应然关系
```

和：

```text
当前 QuickFIX/J example 已经实现的实际关系
```

当前 `Executor` example 通常收到订单后直接构造 `ExecutionReport`，并没有默认再通过 FIX 把订单转发给 `OrderMatch`。因此它更准确地说是卖方执行回报模拟器，而不是已经完成“卖方网关 -> 交易所”的完整实现。

### 标准回答

```text
Banzai 是模拟买方机构的极简下单和订单状态管理前端；Executor 是模拟卖方或券商执行端的示例，负责接收买方订单并产生执行回报；OrderMatch 是模拟交易所或撮合引擎的示例，负责订单簿和撮合。真实生产关系通常可以表示为 Banzai -> Executor -> OrderMatch，但当前 QuickFIX/J example 中 Executor 和 OrderMatch 默认没有直接串成完整链路，Executor 更多是一个独立的卖方执行模拟器。
```

### 记忆重点

```text
角色映射正确；需要始终区分“真实系统架构”和“当前 example 已实现的链路”。
```

---

## 第 3 题：NewOrderSingle 的接收与分发链路

### 题目

请按顺序写出：

```text
一条 NewOrderSingle 从网络进入 QuickFIX/J 后，
是怎么一步步走到 onMessage(NewOrderSingle, sessionID) 的？
```

### 你的回答

> NewOrderSingle会先被解析成FIX协议形式，被send到对应的session对手方，然后对方会根据这个NewOrderSingle对应的协议找到对应的onMessage去进行处理。

### 得分

```text
4.5 / 10
```

### 评价

你抓住了“根据消息类型找到对应的 `onMessage`”这个终点，但混合了发送链路和接收链路。

题目问的是：

```text
从网络进入接收方后，如何进入 onMessage
```

而不是：

```text
业务消息如何发送到对手方
```

另外，网络进来的内容本来就是原始 FIX 报文，不是“先被解析成 FIX 协议形式”。准确地说是：

```text
原始 FIX 报文 -> QuickFIX/J 强类型消息对象
```

### 标准回答

```text
原始 FIX 报文从 TCP / Apache MINA 进入 QuickFIX/J，由 Session 接收并解析。QuickFIX/J 根据 BeginString(8) 和 MsgType(35) 创建强类型的 NewOrderSingle 消息对象，然后进入 Application.fromApp(message, sessionID)。随后 MessageCracker.crack(message, sessionID) 根据消息类型进行分发，最终调用 onMessage(NewOrderSingle, sessionID) 进入业务处理。
```

### 标准链路

```text
TCP / Apache MINA
  -> Session 接收报文
  -> 解析 Tag=Value
  -> 根据 BeginString + MsgType 创建强类型消息
  -> Application.fromApp(message, sessionID)
  -> MessageCracker.crack(message, sessionID)
  -> onMessage(NewOrderSingle, sessionID)
```

### 记忆重点

```text
不要把发送链路和接收链路混在一起。
接收链路的关键节点是：Session -> fromApp -> crack -> onMessage。
```

---

## 第 4 题：ClOrdID、OrderID、ExecID

### 题目

请解释：

```text
ClOrdID(11)
OrderID(37)
ExecID(17)
```

并说明在 `Banzai -> Executor` 例子里它们大致是谁生成的。

### 你的回答

> OrderID是banzai自己生成的一个本地订单号；ClOrderID是一个由executor接到订单后生成的卖方订单号；EXECID是每次订单情况更新后的一个特定标识。

### 得分

```text
2.5 / 10
```

### 评价

`ExecID` 的理解基本正确，但 `OrderID` 和 `ClOrdID` 的归属混淆了。

正确关系是：

```text
Banzai 本地 Order.ID
  -> 通常作为 ClOrdID(11) 发给 Executor

Executor 生成 OrderID(37)
Executor 为每次回报事件生成 ExecID(17)
```

因此：

```text
Banzai 本地 Order.ID != OrderID(37)
ClOrdID(11) 不是 Executor 生成的卖方订单号
```

### 标准回答

```text
ClOrdID(11) 是买方生成的客户订单号，在 Banzai -> Executor 的例子里通常由 Banzai 的本地 Order.ID 充当；OrderID(37) 是卖方 / Executor 为这笔订单分配的执行端订单号；ExecID(17) 是卖方为每一次执行回报事件生成的唯一标识。ClOrdID 用于让买方关联自己的订单，OrderID 用于让卖方识别订单，ExecID 用于区分同一订单生命周期中的不同回报事件。
```

### 示例链路

```text
用户在 Banzai 下单
  -> Banzai 创建本地 Order.ID = C001
  -> NewOrderSingle.ClOrdID(11) = C001
  -> Executor 收到订单
  -> Executor 生成 OrderID(37) = O001
  -> Executor 生成 ExecID(17) = E001
  -> 回 ExecutionReport
```

### 记忆重点

```text
买方本地订单号 -> ClOrdID(11)
卖方订单号     -> OrderID(37)
每次回报事件号  -> ExecID(17)
```

---

## 第 5 题：FieldMap 与 Group

### 题目

请结合行情场景说明：

```text
FieldMap 里普通字段和 Group 的区别是什么？
NoMDEntries(268) 这种 repeating group 是拿来表示什么的？
```

### 你的回答

> FieldMap中的普通字段和Group的区别在于，普通字段只有1个value，Group里面会包含多个普通字段及各自对应的value；NoMDEntries（268）是用来表示有几条行情信息。

### 得分

```text
7.5 / 10
```

### 评价

你的方向是对的：

```text
普通字段 = 一个 Tag 对一个 Value
Group = 多个结构化字段组成的重复记录
```

需要进一步精确的是：

```text
Group 不只是“一组字段”，而是可以重复出现的“一条结构化记录”。
```

例如：

```text
NoMDEntries(268)=2
  Group 1: 269=0, 270=100.10, 271=1000
  Group 2: 269=1, 270=100.12, 271=1200
```

其中：

- Group 1 可以表示一条 Bid。
- Group 2 可以表示一条 Ask。
- `268=2` 表示后面有两条 `MDEntry` 记录。

### 标准回答

```text
FieldMap 中的普通字段表示单个 Tag->Value，例如 55=AAPL；Group 表示一组可以重复出现的结构化字段记录。在行情场景中，NoMDEntries(268) 表示后面有多少条 MDEntry 记录，每个 Group 可以包含 MDEntryType(269)、MDEntryPx(270)、MDEntrySize(271) 等字段。例如 268=2 可以表示一条 Bid 和一条 Ask。
```

### QuickFIX/J 内部模型

```text
FieldMap.fields
  -> 保存普通字段

FieldMap.groups
  -> 保存重复组列表

Message
  -> Header + Body + Trailer
```

### 记忆重点

```text
Group 的本质不是“多个值”，而是“多条重复的字段结构”。
```

---

## 第 6 题：Logon 与用户名密码认证

### 题目

请回答：

```text
FIX 会话建立为什么从 35=A 开始？
如果要在 Logon 里传用户名密码，一般在哪两个回调里处理？各自做什么？
```

### 你的回答

> FIX会话需要由initiator向acceptor发送logon请求，获取相应的回复后建立session;一般在initiator的toAdmin中放入用户名和密码，并在acceptor的fromAdmin中验证用户名和密码。

### 得分

```text
8.5 / 10
```

### 评价

答对了主要流程：

- Initiator 先发送 `Logon(35=A)`。
- Acceptor 校验后回复 `Logon(35=A)`。
- 用户名和密码可在 Initiator 的 `toAdmin(...)` 中加入。
- Acceptor 可在 `fromAdmin(...)` 中读取并验证。

还可以补充：

```text
35=A 不只是“登录请求”，而是 FIX Session 的握手消息。
```

它还用于：

- 确认双方身份。
- 协商 `HeartBtInt(108)` 等会话参数。
- 建立后续序号、心跳和重传管理的基础。
- 校验通过后触发 `onLogon(sessionID)`。

### 标准回答

```text
FIX 会话从 35=A(Logon) 开始，因为它是 FIX 会话层的握手消息，用来确认双方身份、建立会话并协商心跳等基础参数。通常由 initiator 先发送 Logon，acceptor 校验后回一个 Logon，双方随后进入 logged-on 状态。若需要传用户名密码，一般在 initiator 的 toAdmin(...) 中往 35=A 塞入 Username(553) 和 Password(554)，在 acceptor 的 fromAdmin(...) 中读取并校验，校验通过后才进入 onLogon(...)。
```

### 记忆重点

```text
toAdmin  = 发出管理消息前修改 Logon
fromAdmin = 收到管理消息后校验 Logon
onLogon   = 登录成功后的通知
```

---

## 第 7 题：Heartbeat 与 TestRequest

### 题目

请区分：

```text
Heartbeat(35=0)
TestRequest(35=1)
```

要求说明各自在什么情况下触发，以及目的有什么不同。

### 你的回答

> HeartBeat是在自己这一方在HeartBeatInt时间内没有发送信息则需要主动发一条给对方，TestRequest则是对方没有在约定时间内发送过信息就主动发给对方确定对方是否状况正常；Heartbeat是向对方确认自己情况正常，TestRequest是确认对方是否状况正常。

### 得分

```text
9 / 10
```

### 评价

核心区分已经准确：

```text
Heartbeat = 我告诉对方我还活着
TestRequest = 我主动确认对方是否还活着
```

需要更严谨的两个细节：

1. Heartbeat 不是无论如何固定发送，而是本方在 `HeartBtInt` 时间内没有发送任何消息时才需要发送。
2. TestRequest 的判断依据是没有收到对方任何消息，而不只是没有收到 `35=0`。正常业务消息同样可以证明链路活着。

### 标准回答

```text
Heartbeat(35=0) 是在本方在 HeartBtInt 时间内没有发送任何消息时，用来向对方表明本方连接仍然存活的保活消息。TestRequest(35=1) 是在本方在一段时间内没有收到对方任何消息时，主动发给对方的探活消息，用来确认对方是否仍然在线。前者是“我告诉你我还活着”，后者是“我来确认你是否还活着”。
```

### 核心公式

```text
Heartbeat：看“我多久没发消息”
TestRequest：看“我多久没收到对方消息”
```

### 典型流程

```text
长时间空闲
  -> 发 Heartbeat(35=0)

长时间未收到对方消息
  -> 发 TestRequest(35=1)

对方仍然没有响应
  -> 判定连接异常
  -> 断开并重连
```

---

## 第 8 题：MsgSeqNum、MessageStore、FileStore

### 题目

请解释这几个概念的关系：

```text
MsgSeqNum(34)
MessageStore
FileStore
```

并回答断线重连、ResendRequest、会话恢复为什么离不开它们。

### 你的回答

> MsgSeqNum(34)是用来对本方发送的信息进行计数的记录，MessageStore是一个大的概念，FileStore是其中的一个具体化，也就是具体文件以什么形式保存在哪里；断线重连后的信息恢复需要先确定MsgSeqNum是否连续，如果不连续则需要把缺失数据补全，这就需要从FileStorage里去获取。

### 得分

```text
8 / 10
```

### 评价

答对的核心：

- `MsgSeqNum(34)` 与消息序号连续性有关。
- `MessageStore` 是抽象概念，`FileStore` 是具体实现。
- 断线恢复发现序号缺口后，需要从存储中取历史消息。

需要补充和修正：

1. 一个 Session 的两个方向各自独立维护序号：

```text
A -> B 有一套序号
B -> A 也有一套序号
```

2. 恢复逻辑依赖的是 `MessageStore` 抽象，不一定特指 `FileStore`。如果底层选择 `FileStore`，才是从本地文件中恢复。
3. Store 不仅保存历史消息，也保存发送序号和接收序号。

### 标准回答

```text
MsgSeqNum(34) 是 FIX 会话中单方向消息的序号，每条发出的消息都会带上发送方当前的序号，因此一个 Session 的两个方向会各自独立维护序号。MessageStore 是保存会话状态和历史消息的抽象层，既保存收发序号，也保存可供重发的历史消息；FileStore 则是它的一种具体实现，把这些内容落到本地文件中。断线重连后，双方要先比较序号是否连续；如果发现缺口，就通过 ResendRequest 向对方索要缺失区间，对方再从 MessageStore 中取出历史消息重放。如果底层实现是 FileStore，这些恢复数据就是从文件中取得的。
```

### 关系图

```text
Session
  -> MessageStore
       ├─ 保存 next sender sequence
       ├─ 保存 next target sequence
       └─ 保存历史消息

FileStore
  -> MessageStore 的文件实现
  -> 将序号和消息落到本地文件
```

### 记忆重点

```text
恢复依赖 MessageStore；FileStore 只是 MessageStore 的文件版实现。
```

---

## 第 9 题：Dynamic Session 与 configureDynamicSessions

### 题目

请说明：

```text
banzai.cfg 和 banzai_dynamic.cfg 的核心区别是什么？
configureDynamicSessions(...) 在 Executor 里起到了什么作用？
```

重点说清：

```text
“一个端口接多个 Session”是怎么实现的。
```

### 你的回答

> banzai.cfg和banzai_dynamic.cfg的区别在于banzai用的是多session多端口的机制，但是banzai_dynamic.cfg则是一个template实现多session一端口的机制。

### 得分

```text
6.5 / 10
```

### 评价

你抓住了最核心的表面区别：

```text
banzai.cfg
  -> 多 Session、多端口

banzai_dynamic.cfg
  -> 多个 FIX 版本共用一个端口
```

需要修正两个边界：

1. `template` 和真正的动态 Session 创建机制不在 Banzai 侧实现，而是在 Executor 侧：

```text
executor_dynamic.cfg
+ AcceptorTemplate=Y
+ configureDynamicSessions(...)
+ DynamicAcceptorSessionProvider
```

2. `banzai_dynamic.cfg` 只是让客户端把多个版本的连接都发往同一个端口；真正根据 Logon 中的 `BeginString`、`SenderCompID` 和 `TargetCompID` 匹配模板并创建 Session 的，是 Executor 侧。

### 标准回答

```text
banzai.cfg 采用多版本对应多端口的静态 Session 接入方式，不同 BeginString 分别连接不同端口；banzai_dynamic.cfg 则让多个 FIX 版本共用同一个端口，因此客户端侧不再通过端口区分 Session。真正实现“多 Session 一端口”的关键在 Executor 侧：executor_dynamic.cfg 使用 AcceptorTemplate=Y 定义模板，configureDynamicSessions(...) 会收集这些模板并按监听地址分组，为对应端口安装 DynamicAcceptorSessionProvider。Executor 收到 Logon 后，根据 BeginString 和 SessionID 匹配模板，复制模板配置并动态创建实际 Session。
```

### `configureDynamicSessions(...)` 的核心流程

```text
遍历 SessionSettings 中的所有 Session
  -> 找出 AcceptorTemplate=Y 的模板
  -> 获取每个模板所属的监听地址(host:port)
  -> 按监听地址把模板归类
  -> 为每个监听入口安装 DynamicAcceptorSessionProvider
  -> 客户端 Logon 到达后
  -> 根据真实 SessionID 匹配模板
  -> 动态创建具体 Session
```

### “按监听地址分组”的含义

```text
Map<InetSocketAddress, List<TemplateMapping>>
```

也就是：

```text
0.0.0.0:9876
  -> FIX.4.0 模板
  -> FIX.4.1 模板
  -> FIX.4.2 模板

0.0.0.0:9999
  -> FIX.4.4 模板
  -> FIXT.1.1 模板
```

当前 `executor_dynamic.cfg` 的所有模板都继承 `SocketAcceptPort=9876`，因此实际只有一组：

```text
0.0.0.0:9876
  -> 多个 FIX 版本模板
```

### 记忆重点

```text
Banzai dynamic：多个客户端配置连接同一个端口。
Executor dynamic：服务端用模板和 Provider 在该端口动态创建具体 Session。
```

---

## 第 10 题：Session 可靠性主线总结

### 题目

请用不超过 8 行总结：

```text
你现在理解的 QuickFIX/J 主线三
（Session 可靠性：Logon、心跳、断线、重连、序号、重传）
```

### 你的回答

```text
本次测验中未正式作答。
```

### 得分

```text
未评分
```

### 标准参考回答（不超过 8 行）

```text
FIX Session 从 Initiator 发送 Logon(35=A) 开始，双方确认身份并协商 HeartBtInt。
会话空闲时，发送方在 HeartBtInt 内没有发送任何消息，就发送 Heartbeat(35=0)。
如果本方长时间没有收到对方任何消息，就发送 TestRequest(35=1) 探测对端。
网络断开后，Initiator 按 ReconnectInterval 尝试重连，双方重新 Logon。
QuickFIX/J 比较双方的 MsgSeqNum(34)，发现缺口后发送 ResendRequest(35=2)。
对端从 MessageStore 中重放历史消息，或用 SequenceReset(35=4) 做 Gap Fill。
FileStore 是 MessageStore 的文件实现，帮助进程重启和断线后保留序号及历史消息。
应用层仍需结合 ClOrdID、OrderID、ExecID 做幂等和订单状态恢复。
```

### 主线三压缩图

```text
Logon
  -> Heartbeat / TestRequest
  -> 网络断开
  -> Reconnect
  -> 重新 Logon
  -> 比较 MsgSeqNum
  -> ResendRequest
  -> MessageStore 重放 / SequenceReset Gap Fill
  -> Session 恢复
```

---

# 三、错题与重点收束

## 1. 订单标识层次

这是本次最需要优先补强的部分：

```text
Banzai Order.ID
  = Banzai 本地领域订单号

ClOrdID(11)
  = 买方发送给卖方的客户订单号
  = 当前 example 中通常使用 Banzai Order.ID

OrderID(37)
  = 卖方 / Executor 侧订单号

ExecID(17)
  = 每一次执行回报事件的唯一标识
```

## 2. NewOrderSingle 接收链路

```text
原始 FIX 报文
  -> Session
  -> 强类型 NewOrderSingle
  -> Application.fromApp(...)
  -> MessageCracker.crack(...)
  -> onMessage(NewOrderSingle, sessionID)
```

## 3. Dynamic Session 的实现边界

```text
banzai_dynamic.cfg
  = 客户端侧：多个版本连接同一端口

executor_dynamic.cfg
  = 服务端侧：定义 AcceptorTemplate

configureDynamicSessions(...)
  = 收集模板并安装动态 Provider

DynamicAcceptorSessionProvider
  = 根据收到的 Logon 动态创建具体 Session
```

## 4. ExecutionReport 的来源

`ExecutionReport(35=8)` 不一定都要等交易所返回：

```text
卖方内部校验 / 风控 / 接单 / 拒单
  -> 可以产生 ExecutionReport

交易所成交 / 部成 / 撤单确认
  -> 也可以由卖方转换成 ExecutionReport 回给买方
```

当前 `Executor` example 通常直接构造回报，并没有默认接入真正的外部交易所。

---

# 四、下一阶段复习建议

建议按下面顺序复习：

```text
第一优先级：订单标识与订单状态管理
  Order.ID -> ClOrdID -> OrderID / ExecID

第二优先级：NewOrderSingle 接收链路
  Session -> fromApp -> crack -> onMessage

第三优先级：Dynamic Session
  配置模板 -> 监听地址分组 -> Provider -> 动态创建 Session

第四优先级：主线三完整复述
  Logon -> 心跳 -> 探活 -> 重连 -> 序号 -> 重传 -> 恢复
```

复习目标可以设为：

```text
能够不用看文档，分别用 5~8 句话解释：
1. 一条订单如何从网络进入业务方法；
2. 三种订单 ID 如何关联；
3. 一个端口如何动态承载多个 FIX Session；
4. 断线后 Session 如何恢复一致状态。
```
