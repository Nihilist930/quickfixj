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

### 记忆结论

```text
不是按 35=D 直接找方法名
而是：

初始化时：按 handler 第一个参数类型建表
运行时  ：按 message.getClass() 查表
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

## 10. 复习路线

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
