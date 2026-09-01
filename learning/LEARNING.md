# QuickFIX/J 学习路线与进度

> 学习分支：`learning/ordermatch`  
> 最后更新：2026-08-19  
> 当前阶段：阶段三（理解 Session 引擎，正在进行）

本文件是本地学习的恢复入口。每次继续前，先读 **当前进度** 与 **下一步**。

---

## 当前进度

已完成：

### 阶段一：跑通示例并沿消息链路阅读

- [x] 理解项目的三层运行时结构：`quickfixj-base`、`quickfixj-core`、`quickfixj-messages`。
- [x] 使用 Maven 3.9.16 成功构建 `ordermatch` 及其依赖链。
- [x] 运行 `OrderMatch`，作为 FIX 4.2 的 Acceptor，监听 `localhost:9876`。
- [x] 配置并运行 `Banzai`，作为 FIX 4.2 的 Initiator。
- [x] Banzai 与 OrderMatch 成功完成 FIX Logon。
- [x] 在 Banzai 提交订单，OrderMatch 成功收到订单。
- [x] 建立本地学习分支：`learning/ordermatch`。
- [x] 沿第一笔订单打通主链路：`Banzai -> NewOrderSingle -> OrderMatch -> ExecutionReport`。
- [x] 理解 `fromApp(Message, SessionID)` 为什么先接收到通用 `Message`。
- [x] 理解 `crack(message, sessionId)` 如何基于运行时类型分派到 `onMessage(NewOrderSingle, SessionID)`。
- [x] 理解 `8=FIX.4.2` 与 `35=D` 在解析阶段分别表示协议版本与消息类型。
- [x] 理解 Header 中 `49=BANZAI`、`56=EXEC` 的会话身份含义。
- [x] 确认 `ExecutionReport` 在 `ordermatch` 示例中的创建位置与发送 API。
- [x] 梳理从服务启动、Logon、下单、回报到 Logoff/停止的完整主流程。

### 阶段二：理解消息模型与字典（已完成）

- [x] 确认当前学习环境固定在 FIX 4.2：`Banzai` 默认加载 `banzai_ordermatch.cfg`，其 `BeginString=FIX.4.2`、`DataDictionary=FIX42.xml`。
- [x] 对照 `quickfixj-messages-fix42/pom.xml` 与 `quickfixj-messages-fix44/pom.xml`，理解各协议版本消息模块会分别打包自己的字典 XML 与强类型消息类。
- [x] 建立“协议版本 -> 字典文件 -> Java 强类型消息包”的对应关系：例如 FIX 4.2 对应 `FIX42.xml` 与 `quickfix.fix42.*`。
- [x] 阅读 `quickfixj-base/src/main/java/quickfix/FieldMap.java`，理解字段容器、Header/Body/Trailer 的公共存储结构。
- [x] 阅读 `quickfixj-base/src/main/java/quickfix/Message.java`，理解消息对象的整体结构与 parse/set/get 的入口。
- [x] 阅读 `quickfixj-base/src/main/java/quickfix/DataDictionary.java`，理解字段校验、必填校验、枚举值校验的规则来源。
- [x] 结合 `quickfixj-core/src/main/java/quickfix/MessageSessionUtils.java`，理解原始字符串如何被解析成强类型消息对象。
- [x] 结合 `quickfixj-core/src/main/java/quickfix/DefaultMessageFactory.java`，理解 `BeginString + MsgType` 如何决定具体消息类。

阶段二的核心产出：

```text
FieldMap 负责“字段怎么存”
Message 负责“消息怎么组织成 Header / Body / Trailer”
DataDictionary 负责“这样组织对不对、字段是否合法”
DefaultMessageFactory 负责“BeginString + MsgType -> 具体强类型 Message 类”
MessageSessionUtils / Message.fromString 负责“原始 FIX 字符串 -> 强类型消息对象”
```

当前本地配对：

```text
Banzai (Initiator)                     OrderMatch (Acceptor)
----------------------------------     ----------------------------------
BeginString=FIX.4.2                    BeginString=FIX.4.2
SenderCompID=BANZAI                    SenderCompID=EXEC
TargetCompID=EXEC                      TargetCompID=BANZAI
SocketConnectPort=9876                 SocketAcceptPort=9876
```

Banzai 的专用配置：

```text
quickfixj-examples/banzai/src/main/resources/
quickfix/examples/banzai/banzai_ordermatch.cfg
```

当前 `Banzai` 无启动参数时会加载该配置：

```text
quickfixj-examples/banzai/src/main/java/
quickfix/examples/banzai/Banzai.java
```

---

## 下一步：沿着第一笔订单阅读源码

目标：理解 Banzai 的 `NewOrderSingle` 如何从网络报文进入 OrderMatch 的强类型处理方法，再如何返回 `ExecutionReport`。

### 1. 准备运行环境

按顺序启动：

1. `quickfix.examples.ordermatch.Main`
2. `quickfix.examples.banzai.Banzai`

确认两侧都已 Logon：

```text
OrderMatch 控制台：Logon - FIX.4.2:EXEC->BANZAI
Banzai Session 下拉框：FIX.4.2:BANZAI->EXEC
```

### 2. 在 OrderMatch 设置断点

文件：

```text
quickfixj-examples/ordermatch/src/main/java/
quickfix/examples/ordermatch/Application.java
```

建议按顺序设置断点：

```java
// ① 所有业务消息的统一入口
crack(message, sessionId);

// ② 35=D 被 MessageCracker 分派到这里
public void onMessage(NewOrderSingle message, SessionID sessionID)

// ③ 示例业务层：订单进入撮合器
processOrder(order);

// ④ 创建 ExecutionReport
private void updateOrder(Order order, char status)

// ⑤ 发送回报
Session.sendToTarget(fixOrder, senderCompId, targetCompId);
```

### 3. 从 Banzai 发一笔订单

先使用最简单的 Market 订单：

```text
Symbol: AAPL
Side: Buy
Type: Market
Quantity: 100
Session: FIX.4.2:BANZAI->EXEC
```

进入断点后，检查：

```text
message 的运行时类型：quickfix.fix42.NewOrderSingle
message Header：35、49、56、34
ClOrdID (11)：客户端订单编号
Symbol (55)：AAPL
Side (54)：买卖方向
OrderQty (38)：100
sessionID：当前本地会话标识
```

### 4. 回答以下问题后再进入下一阶段

- `fromApp(Message, SessionID)` 为什么拿到的是通用 `Message`？
  - 因为 `Application` 接口把应用层消息的统一入口就定义成了 `fromApp(Message, SessionID)`，引擎在 `Session.fromCallback(...)` 中只区分这是 Admin 还是 App 消息，然后把它作为通用 `Message` 交给应用层。此时这样设计是为了先提供一个统一入口；真正的强类型分派是在后面的 `MessageCracker` 中完成。可对照：`quickfixj-core/src/main/java/quickfix/Application.java:124`、`quickfixj-core/src/main/java/quickfix/Session.java:1935`。
- `crack(message, sessionId)` 如何决定调用 `onMessage(NewOrderSingle, ...)`？
  - `MessageCracker` 在初始化时会反射扫描当前对象上所有符合签名的 `onMessage(某消息类型, SessionID)` 方法，并按“消息 Java 类 -> 调用器”建立映射。`crack(...)` 执行时直接用 `message.getClass()` 去这个映射里查找；如果传入对象的运行时类型正好是 `quickfix.fix42.NewOrderSingle`，就会命中并调用 `onMessage(NewOrderSingle, SessionID)`。可对照：`quickfixj-core/src/main/java/quickfix/MessageCracker.java:71`、`quickfixj-core/src/main/java/quickfix/MessageCracker.java:122`。
- `8=FIX.4.2` 与 `35=D` 分别表达什么？
  - `8=FIX.4.2` 是 BeginString，表示这条报文使用 FIX 4.2 协议版本；`35=D` 是 MsgType，表示这是一条 `NewOrderSingle` 新订单消息。引擎解析原始字符串时，会先从报文里取出 BeginString 和 MsgType，再调用消息工厂创建对应版本、对应类型的强类型消息对象。可对照：`quickfixj-core/src/main/java/quickfix/MessageSessionUtils.java:37`、`quickfixj-core/src/main/java/quickfix/MessageSessionUtils.java:71`、`quickfixj-core/src/main/java/quickfix/DefaultMessageFactory.java:157`。
- 消息 Header 的 `49=BANZAI`、`56=EXEC` 分别代表什么？
  - `49` 是 `SenderCompID`，表示发送方逻辑身份，所以 `49=BANZAI` 表示这条消息是 BANZAI 发出的；`56` 是 `TargetCompID`，表示接收方逻辑身份，所以 `56=EXEC` 表示目标是 EXEC。对 OrderMatch 收到的订单来说，就是“BANZAI 发给 EXEC”。这些值也会参与组成 `SessionID`。可对照：`quickfixj-base/src/main/java/quickfix/Message.java:568`、`quickfixj-examples/ordermatch/src/main/java/quickfix/examples/ordermatch/Application.java:85`。
- `ExecutionReport` 是在哪个方法创建、通过哪个 API 发送的？
  - 在 OrderMatch 示例里，正常订单状态回报是在 `updateOrder(Order order, char status)` 中创建 `ExecutionReport` 的；拒单场景则在 `rejectOrder(String senderCompId, String targetCompId, ...)` 中创建。创建完成后，都是通过 `Session.sendToTarget(...)` 发回对应会话。可对照：`quickfixj-examples/ordermatch/src/main/java/quickfix/examples/ordermatch/Application.java:117`、`quickfixj-examples/ordermatch/src/main/java/quickfix/examples/ordermatch/Application.java:164`、`quickfixj-core/src/main/java/quickfix/Session.java:723`、`quickfixj-core/src/main/java/quickfix/Session.java:762`。

完成这些问题后，继续阅读：

```text
quickfixj-core/src/main/java/quickfix/MessageCracker.java
quickfixj-base/src/main/java/quickfix/Message.java
quickfixj-base/src/main/java/quickfix/FieldMap.java
```

### 5. 从服务启动到结束的完整流程示意图

下面这张图把本地学习环境里最重要的一条主线串起来：

- Banzai：Initiator，负责主动连上 OrderMatch，并发送订单
- OrderMatch：Acceptor，负责监听端口、接收订单、返回回报
- QuickFIX/J Core：负责会话、解析、校验、分派、发送

```text
一、服务启动阶段

OrderMatch Main.main
  -> 读取 ordermatch.cfg
  -> new SessionSettings(inputStream)
  -> new Application()
  -> new FileStoreFactory(settings)
  -> new ScreenLogFactory(settings)
  -> new DefaultMessageFactory()
  -> new SocketAcceptor(...)
  -> acceptor.start()
  -> 创建 Session / 启动监听端口 9876

Banzai Banzai.main
  -> 读取 banzai_ordermatch.cfg
  -> new SessionSettings(inputStream)
  -> new BanzaiApplication(...)
  -> new FileStoreFactory(settings)
  -> new ScreenLogFactory(...)
  -> new DefaultMessageFactory()
  -> new SocketInitiator(...)
  -> banzai.logon()
  -> initiator.start()
  -> 创建 Session / 主动连接 localhost:9876
```

```text
二、FIX 会话建立（Logon）

Banzai(Session)
  -> Session.next()
  -> generateLogon()
  -> initializeHeader(...)
  -> sendRaw(Logon)
  -> 通过网络发给 OrderMatch

OrderMatch(IO)
  -> AbstractIoHandler.messageReceived()
  -> MessageSessionUtils.parse(...)
  -> Session.next(message)
  -> nextLogon(message)
  -> 状态机确认登录成功
  -> application.onLogon(sessionID)

OrderMatch(Session)
  -> 回发 Logon

Banzai(IO)
  -> AbstractIoHandler.messageReceived()
  -> MessageSessionUtils.parse(...)
  -> Session.next(message)
  -> nextLogon(message)
  -> 状态机确认登录成功
  -> application.onLogon(sessionID)
```

```text
三、用户在 Banzai 下单

用户点击 Submit
  -> OrderEntryPanel.SubmitListener.actionPerformed(...)
  -> application.send(order)
  -> send42(order)                   // 当前学习环境是 FIX 4.2
  -> new quickfix.fix42.NewOrderSingle(...)
  -> populateOrder(order, message)
  -> Session.sendToTarget(message, sessionID)
  -> Session.lookupSession(sessionID)
  -> session.send(message)
  -> initializeHeader(...)
  -> application.toApp(...)
  -> 持久化到 MessageStore
  -> responder.send(messageString)
  -> 通过网络发给 OrderMatch
```

```text
四、OrderMatch 收到订单并进入业务处理

OrderMatch(IO)
  -> AbstractIoHandler.messageReceived()
  -> sessionLog.onIncoming(raw FIX string)
  -> MessageSessionUtils.parse(session, raw)
       -> 读取 8=FIX.4.2
       -> 读取 35=D
       -> DefaultMessageFactory.create(beginString, msgType)
       -> 创建 quickfix.fix42.NewOrderSingle
       -> message.parse(...)
  -> Session.next(message)
  -> verify / DataDictionary 校验 / Session 状态检查
  -> application.fromApp(message, sessionID)

OrderMatch(Application)
  -> fromApp(Message, SessionID)
  -> crack(message, sessionID)
  -> onMessage(NewOrderSingle, SessionID)
  -> 读取 Header 和 Body 字段
  -> 构造业务 Order
  -> processOrder(order)
       -> acceptOrder(order) 或 rejectOrder(order)
       -> 如有撮合则 fillOrder(order)
  -> updateOrder(order, status) / rejectOrder(...)
  -> new ExecutionReport(...)
  -> Session.sendToTarget(fixOrder, senderCompId, targetCompId)
```

```text
五、ExecutionReport 返回到 Banzai

Banzai(IO)
  -> AbstractIoHandler.messageReceived()
  -> MessageSessionUtils.parse(...)
  -> Session.next(message)
  -> BanzaiApplication.fromApp(message, sessionID)
  -> SwingUtilities.invokeLater(new MessageProcessor(...))
  -> MessageProcessor.run()
       -> 如果 35=8，则 executionReport(message, sessionID)
       -> 更新 OrderTableModel
       -> 更新 ExecutionTableModel
       -> UI 显示 NEW / FILLED / REJECTED 等状态
```

```text
六、会话结束（Logoff）

Banzai 菜单 Logoff
  -> Banzai.logout()
  -> Session.lookupSession(sessionId).logout("user requested")
  -> Session.next() 后续调度中生成 Logout
  -> generateLogout(...)
  -> 发给 OrderMatch

OrderMatch(Session)
  -> nextLogout(message)
  -> 如本端尚未发 Logout，则 generateLogout(response)
  -> disconnect(...)
  -> application.onLogout(sessionID)

Banzai(Session)
  -> 收到 Logout response
  -> nextLogout(message)
  -> disconnect(...)
  -> application.onLogout(sessionID)
```

```text
七、服务停止

OrderMatch
  -> 控制台输入 #quit
  -> acceptor.stop()
  -> logoutAllSessions(...)
  -> stopAcceptingConnections()
  -> stopSessionTimer()
  -> unregisterSessions(...)
  -> 进程退出

Banzai
  -> 先 Logoff 可正常结束 FIX 会话
  -> 关闭窗口则结束 UI / JVM 进程
```

### 6. 读源码时建议对照的关键文件

```text
启动入口
- quickfixj-examples/ordermatch/.../Main.java
- quickfixj-examples/banzai/.../Banzai.java

应用层回调
- quickfixj-examples/ordermatch/.../Application.java
- quickfixj-examples/banzai/.../BanzaiApplication.java
- quickfixj-examples/banzai/.../ui/OrderEntryPanel.java

引擎核心
- quickfixj-core/src/main/java/quickfix/Session.java
- quickfixj-core/src/main/java/quickfix/MessageSessionUtils.java
- quickfixj-core/src/main/java/quickfix/DefaultMessageFactory.java
- quickfixj-core/src/main/java/quickfix/MessageCracker.java
- quickfixj-core/src/main/java/quickfix/mina/AbstractIoHandler.java
```

---

## 阶段二复盘：5 个问题与对应回答

### 问题 1：`FieldMap`、`Message`、`DataDictionary` 各自负责什么？

```text
FieldMap
  = 字段容器
  = 负责 Tag -> Value 与重复组（Group）的存储和读取

Message
  = 一条完整 FIX 消息对象
  = Header + Body + Trailer
  = 其中 Header、Body、Trailer 本质上都建立在 FieldMap 之上

DataDictionary
  = FIX 协议规则书 / 字典
  = 负责定义字段、消息类型、必填字段、枚举值、字段类型、重复组规则
```

一句话记忆：

```text
FieldMap 负责“装字段”
Message 负责“组织成一条消息”
DataDictionary 负责“定义这条消息是否合法”
```

### 问题 2：原始 FIX 字符串是如何变成强类型消息对象的？

```text
原始 FIX 字符串
  -> MessageSessionUtils.parse(...)
  -> 读取 8=BeginString 和 35=MsgType
  -> DefaultMessageFactory.create(beginString, msgType)
  -> 创建对应版本、对应类型的强类型消息对象
  -> Message.fromString(...)
  -> 用 FieldMap 把字段填入 Header / Body / Trailer
  -> 得到例如 quickfix.fix42.NewOrderSingle 这样的对象
```

一句话记忆：

```text
8 和 35 先决定“创建哪个 Message 类”
fromString 再决定“把具体字段填进去”
```

### 问题 3：`DataDictionary` 在解析和校验时具体起什么作用？

```text
解析阶段：
- 判断字段属于 Header / Body / Trailer 哪一部分
- 判断某字段是否是重复组起点
- 指导重复组如何展开

校验阶段：
- 检查 MsgType 是否存在
- 检查必填字段是否缺失
- 检查字段是否属于该消息类型
- 检查字段类型是否正确
- 检查枚举值是否合法
- 检查 NumInGroup 与实际组数量是否一致
```

一句话记忆：

```text
DataDictionary 既参与“怎么解析”，也参与“解析完是否合规”
```

### 问题 4：为什么说 `Message` 和 `FieldMap` 是“实例数据层”，`DataDictionary` 是“规则层”？

```text
Message / FieldMap
  存的是“这一条具体消息现在有哪些字段、字段值是什么”

DataDictionary
  存的是“这一类消息允许有哪些字段、字段类型是什么、哪些是必填”
```

类比：

```text
Message = 一张已经填好的表单
FieldMap = 表单里的字段容器
DataDictionary = 填表规则与模板说明书
```

### 问题 5：阶段二里 `DefaultMessageFactory`、`Message`、`DataDictionary` 之间是什么关系？

```text
DefaultMessageFactory
  负责：先创建“哪一个 Java 消息类”

Message
  负责：承载这条消息的 Header / Body / Trailer 数据

DataDictionary
  负责：告诉 Message 解析时怎么分区、怎么识别 Group、怎么做字段合法性校验
```

三者串联关系：

```text
BeginString + MsgType
  -> DefaultMessageFactory.create(...)
  -> 生成强类型 Message 对象
  -> Message.fromString(...)
  -> 借助 DataDictionary 完成解析与校验
```

---

## 下一步：进入阶段三（理解 Session 引擎）

目标：理解 Logon、心跳、序号、断线、重传和状态机，建立“会话引擎如何驱动整个 FIX 生命周期”的整体认知。

### 阶段三建议先看的一条主线

```text
Session.next()
  -> generateLogon()
  -> initializeHeader(...)
  -> sendRaw(...)
  -> 收到对端消息后再进入 nextLogon / nextLogout / nextQueued
```

### 阶段三建议优先断点

```text
quickfixj-core/src/main/java/quickfix/Session.java
- Session.next()
- generateLogon()
- initializeHeader(...)
- sendRaw(...)
- nextLogon(...)
- nextLogout(...)
```

### 阶段一：先学会使用（已完成）

目标：运行示例，理解 Application 回调、配置、订单收发。

```text
README
→ OrderMatch / Banzai 示例
→ Application
→ MessageCracker
→ SessionSettings
```

产出：可以解释 `Logon → NewOrderSingle → ExecutionReport` 的完整流程。

### 阶段二：理解消息模型与字典

目标：理解 FIX 字段、报文结构、强类型消息、重复组和校验。

阅读顺序：

```text
quickfixj-base/.../quickfix/FieldMap.java
quickfixj-base/.../quickfix/Message.java
quickfixj-base/.../quickfix/DataDictionary.java
quickfixj-core/.../quickfix/MessageSessionUtils.java
quickfixj-core/.../quickfix/DefaultMessageFactory.java
```

重点：

```text
Header / Body / Trailer
Tag=Value
DataDictionary 校验
MsgType 与协议版本
重复组（Group）
```

### 阶段三：理解 Session 引擎

目标：理解 Logon、心跳、序号、断线、重传和状态机。

阅读顺序：

```text
quickfixj-core/.../quickfix/DefaultSessionFactory.java
quickfixj-core/.../quickfix/SessionState.java
quickfixj-core/.../quickfix/Session.java
quickfixj-core/.../quickfix/mina/AbstractIoHandler.java
quickfixj-core/.../quickfix/mina/message/FIXMessageDecoder.java
quickfixj-core/.../quickfix/mina/SessionConnector.java
```

必须实验的场景：

```text
正常 Logon
Heartbeat / TestRequest
消息序号递增
高于预期的序号
ResendRequest
断线重连与 Store
```

### 阶段三：理解 Session 引擎（当前进行中）

- [ ] 阅读 `quickfixj-core/src/main/java/quickfix/Session.java` 的 `next()`，理解定时驱动入口。
- [ ] 追踪 `generateLogon()`、`initializeHeader(...)`、`sendRaw(...)` 的出站 Logon 流程。
- [ ] 追踪 `nextLogon(...)`、`nextLogout(...)` 的入站会话状态转换。
- [x] 结合真实日志观察 `MsgSeqNum(34)`、Heartbeat、TestRequest、Logout 和断线重连。
- [ ] 使用 Banzai + Executor 做多 FIX 版本 Session 联调，确认 `fix40` 到 `fix50` 的强类型分派。

阶段三的入口主线：

```text
Session.next()
  -> generateLogon()
  -> initializeHeader(...)
  -> sendRaw(...)
  -> 对端消息进入 nextLogon / nextLogout / nextQueued
```

### 阶段三重点：断线重连与会话恢复

断线重连不是简单地重新建立 TCP 连接，而是恢复一个已经存在的 FIX Session。恢复的核心是：

```text
TCP 连接断开
  -> Session 保留本地 MessageStore
  -> Banzai 按 ReconnectInterval 尝试重连
  -> 双方重新发送 Logon
  -> 使用 Store 中保存的发送/接收序号继续通信
  -> 发现序号缺口后通过 ResendRequest 请求补发
  -> 对方通过历史消息重发或 SequenceReset/GAP_FILL 跳过空缺
  -> 双方将会话推进到一致的下一个序号
```

本次 Banzai 与 Executor/OrderMatch 的恢复日志说明：

```text
Banzai -> EXEC：35=A, 34=54
  Banzai 不是从 1 开始，而是继续发送第 54 条消息。

EXEC -> Banzai：35=A, 34=55
  EXEC 也使用已保存的历史序号继续发送第 55 条消息。

Banzai：MsgSeqNum too high, expecting 52 but received 55
  Banzai 期望收到 EXEC 的 52，但实际先收到 55，说明 52~54 存在缺口。

Banzai -> EXEC：35=2, 34=55, 7=52, 16=0
  Banzai 发送 ResendRequest，请 EXEC 从 52 补到最新消息。

EXEC -> Banzai：35=8, 34=52, 43=Y
  EXEC 重发历史 ExecutionReport；43=Y 表示这是可能重复/恢复场景下的补发消息。

EXEC -> Banzai：35=4, 34=53, 43=Y, 36=57, 123=Y
  EXEC 用 SequenceReset/GAP_FILL 跳过 53~56，告诉 Banzai 下一条从 57 开始。
```

#### `MsgSeqNum(34)` 的含义

```text
34=MsgSeqNum
= 当前 FIX Session、当前发送方向上的消息序号

Banzai -> EXEC 有一条独立序列
EXEC -> Banzai 有另一条独立序列
```

在 QuickFIX/J 中，出站消息的序号由 `Session` 写入 Header：

```text
Session.initializeHeader(...)
  -> header.setInt(MsgSeqNum.FIELD, getExpectedSenderNum())
  -> 从 MessageStore 取得 next sender sequence number
```

接收时，`Session` 会把收到的 `34` 与本地期望的 `next target sequence number` 比较：

```text
收到序号 == 期望序号
  -> 正常处理并递增
收到序号 > 期望序号
  -> 发现消息缺口，缓存高序号消息并发送 ResendRequest
收到序号 < 期望序号
  -> 可能是重复消息或异常序号，按会话恢复规则处理
```

#### 断线重连的实际模拟方法

```text
实验一：保留双方 Store
1. 启动 Banzai 与 OrderMatch/Executor。
2. 正常 Logon，并发送几条订单或管理消息。
3. 停止一端进程，模拟连接断开。
4. 等待 Banzai 的 ReconnectInterval 到期并自动重连。
5. 重新启动对端。
6. 观察重连后的 Logon、34、35=2、35=4 和 43=Y。

实验二：制造序号不一致
1. 停止双方进程。
2. 只删除一端的 FileStore，保留另一端的 Store。
3. 重新启动双方。
4. 观察双方序号认知不一致时的 Logon、ResendRequest 和 SequenceReset。
```

对应配置与源码锚点：

```text
banzai_ordermatch.cfg
- FileStorePath=target/data/banzai_ordermatch
- ReconnectInterval=5

ordermatch.cfg
- FileStorePath=target/data/ordermatch

quickfixj-core/src/main/java/quickfix/Session.java
- initializeHeader(...)：写入 MsgSeqNum(34)
- generateLogon(...)：重连后的 Logon
- nextLogon(...)：处理 Logon 与序号协商
- nextResendRequest(...)：处理补发请求
- disconnect(...)：断开与重连状态切换
```

记忆结论：

```text
会话恢复 = 连接恢复 + Store 状态恢复 + 序号校准 + 缺失消息恢复
MsgSeqNum = FIX Header 中的 34 字段，不是全局序号，而是每个 Session、每个方向独立递增
```

---

### 阶段四：并发、存储和运维

目标：能做生产集成与问题排查。

```text
SocketInitiator / SocketAcceptor
ThreadedSocketInitiator / ThreadedSocketAcceptor
MessageStore / FileStore / JdbcStore
LogFactory
threading-model.md
threading-developer-guide.md
```

### 阶段五：定制协议与贡献

目标：支持自定义字段、私有字典、引擎改造。

```text
customising-quickfixj.md
quickfixj-messages/readme.md
quickfixj-dictgenerator
quickfixj-orchestration
acceptance tests
```

---

## 常用命令

使用本机 Maven 3.9.16 构建 OrderMatch 及依赖：

```powershell
& "C:\Users\liusi\.m2\wrapper\dists\apache-maven-3.9.16-bin\5grr65jo27hi51sujmtcldfovl\apache-maven-3.9.16\bin\mvn.cmd" `
  -pl quickfixj-examples/ordermatch -am clean package `
  -DskipTests -Dmaven.javadoc.skip=true `
  -PskipBundlePlugin,minimal-fix-latest
```

检查当前分支和改动：

```bash
git status --short --branch
git branch --show-current
```

---

## 关键概念速查

| 概念 | 含义 |
|---|---|
| Initiator | 主动建立 TCP/FIX 连接的一方；本学习环境中为 Banzai。 |
| Acceptor | 监听端口、接受连接的一方；本学习环境中为 OrderMatch。 |
| SessionID | FIX 会话逻辑身份，通常由 BeginString、SenderCompID、TargetCompID 组成。 |
| `35=A` | Logon。 |
| `35=D` | NewOrderSingle，新订单。 |
| `35=8` | ExecutionReport，订单状态/成交回报。 |
| `49` | SenderCompID，发送方身份。 |
| `56` | TargetCompID，接收方身份。 |
| `34` | MsgSeqNum，单方向递增的消息序号。 |
| `Application` | 业务回调接口。 |
| `MessageCracker` | 按协议版本与 MsgType 将通用 Message 分派到强类型处理方法。 |
| `Session.sendToTarget` | 将业务消息交给对应 FIX Session 发送。 |
