# QuickFIX/J 学习路线与进度

> 学习分支：`learning/ordermatch`  
> 最后更新：2026-08-16  
> 当前阶段：阶段一（跑通示例并沿消息链路阅读）

本文件是本地学习的恢复入口。每次继续前，先读 **当前进度** 与 **下一步**。

---

## 当前进度

已完成：

- [x] 理解项目的三层运行时结构：`quickfixj-base`、`quickfixj-core`、`quickfixj-messages`。
- [x] 使用 Maven 3.9.16 成功构建 `ordermatch` 及其依赖链。
- [x] 运行 `OrderMatch`，作为 FIX 4.2 的 Acceptor，监听 `localhost:9876`。
- [x] 配置并运行 `Banzai`，作为 FIX 4.2 的 Initiator。
- [x] Banzai 与 OrderMatch 成功完成 FIX Logon。
- [x] 在 Banzai 提交订单，OrderMatch 成功收到订单。
- [x] 建立本地学习分支：`learning/ordermatch`。

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
- `crack(message, sessionId)` 如何决定调用 `onMessage(NewOrderSingle, ...)`？
- `8=FIX.4.2` 与 `35=D` 分别表达什么？
- 消息 Header 的 `49=BANZAI`、`56=EXEC` 分别代表什么？
- `ExecutionReport` 是在哪个方法创建、通过哪个 API 发送的？

完成这些问题后，继续阅读：

```text
quickfixj-core/src/main/java/quickfix/MessageCracker.java
quickfixj-base/src/main/java/quickfix/Message.java
quickfixj-base/src/main/java/quickfix/FieldMap.java
```

---

## 全项目学习框架

### 阶段一：先学会使用（当前）

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
