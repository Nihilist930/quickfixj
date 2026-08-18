# 基于 Banzai 的买方实战路线图

> 类型：长期演进路线图  
> 适用范围：从 QuickFIX/J 示例走向更贴近海外机构买方的本地实战项目  
> 基础起点：`Banzai + OrderMatch` 已跑通，`Banzai + Executor` 的多版本联调作为阶段三实验待确认

本文件不是源码阅读清单，而是按“买方系统能力层次 + 可落地小项目 + 对应示例映射”组织的长期路线图。

---

## 1. 总览图：从示例到买方小型 OMS/EMS

```text
                    基于 Banzai 的买方实战路线图

┌──────────────────────────────────────────────────────────────────────┐
│ 第一层：把 Banzai 看成“买方交易前端 / 订单录入台”                   │
└──────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 第二层：补齐买方最小闭环                                              │
│ 行情 -> 下单 -> 回报 -> 订单状态                                      │
└──────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 第三层：补齐真实买方业务能力                                          │
│ 多账户 / 风控 / 路由 / 分配 / 审计                                    │
└──────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 第四层：补齐会话与运维能力                                            │
│ 多 Session / 断线重连 / Drop Copy / 序号恢复 / 文件留痕              │
└──────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│ 目标：把示例演进成一个“可演示、可断点、可复习”的小型买方 OMS/EMS     │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. 当前三个 example 在买方实战里的角色映射

```text
Banzai
= 买方前端 / 交易员下单台
= 负责订单录入、发单、接收 ExecutionReport、更新订单状态

Executor
= 券商执行服务 / 卖方网关的简化版
= 负责接单、校验、按规则成交或回报

OrderMatch
= 交易场所 / 撮合引擎的简化版
= 负责订单簿、撮合、市场数据返回
```

```text
学习时最重要的定位：

Banzai 不是“完整买方系统”
但它非常适合继续扩造成：

Banzai
├─ 订单录入台
├─ 账户/组合下单台
├─ 简化 OMS
├─ 简化 EMS
└─ FIX 行情 + FIX 交易双会话前端
```

---

## 3. 路线图主线：优先做哪 5 个项目

```text
P1. 行情面板 + 发单联动
P2. 多账户 / Block Order / Allocation
P3. 买方预交易风控
P4. 多券商路由 / 多 Session 管理
P5. 交易后分配 / 审计 / Drop Copy
```

这 5 个项目不是并列独立的，而是一条自然演进链：

```text
Banzai 原始示例
  -> P1 先看到“价格驱动下单”
  -> P2 再进入“机构账户结构”
  -> P3 增加“真实买方风控前置”
  -> P4 增加“跨券商执行路由”
  -> P5 补齐“交易后处理与留痕”
```

---

## 4. 第一优先级项目：FIX 行情面板 + 发单联动

### 4.1 目标

把当前 Banzai 从“纯手工下单 UI”扩成“先看行情，再下单”的最小买方终端。

```text
MarketDataRequest
  -> OrderMatch 返回 MarketDataSnapshotFullRefresh
  -> Banzai 更新价格面板
  -> 用户基于价格发 NewOrderSingle
  -> 收到 ExecutionReport 后更新状态
```

### 4.2 为什么最适合先做

```text
优点
├─ 改动范围可控
├─ 直接补上“行情 -> 交易”的闭环
├─ 能把 fromApp 的用途扩展到不止 ExecutionReport
└─ 很适合打断点观察两类业务消息：行情与回报
```

### 4.3 对应源码映射

```text
Banzai 侧
- quickfixj-examples/banzai/src/main/java/quickfix/examples/banzai/BanzaiApplication.java
- quickfixj-examples/banzai/src/main/java/quickfix/examples/banzai/ui/OrderEntryPanel.java

OrderMatch 侧
- quickfixj-examples/ordermatch/src/main/java/quickfix/examples/ordermatch/Application.java
  已有 onMessage(MarketDataRequest, SessionID)
```

### 4.4 做完后你会真正理解什么

```text
一个买方前端通常至少会有两类业务消息入口：
- 行情类 fromApp
- 回报类 fromApp

也会有两类用户动作出口：
- 请求行情
- 发送订单
```

### 4.5 最小完成标志

```text
- Banzai 可以发 MarketDataRequest
- OrderMatch 可以返回 MarketDataSnapshotFullRefresh
- Banzai 界面能显示某个 Symbol 的 Bid / Ask / Last（哪怕是简化版）
- 用户可基于界面价格继续发单
```

### 4.6 从一次快照扩展到持续订阅

当前最小实现是一次性快照：

```text
Banzai
  │
  │ MarketDataRequest（35=V，263=0 Snapshot）
  ▼
OrderMatch.onMessage(MarketDataRequest, ...)
  │
  ▼
MarketDataSnapshotFullRefresh（35=W）
  │
  ▼
Banzai.fromApp(...)
  │
  ▼
控制台打印一次行情
```

持续订阅使用 `SubscriptionRequestType(263)` 表示订阅语义：

```text
263=0  -> Snapshot：只返回一次快照
263=1  -> Snapshot + Updates：返回初始快照并持续更新
263=2  -> Disable previous subscription：取消指定订阅
```

持续订阅后的目标链路：

```text
Banzai 登录成功
  │
  ▼
发送 MarketDataRequest
  ├─ MDReqID（262）
  ├─ SubscriptionRequestType=1（263）
  ├─ NoMDEntryTypes（267）
  └─ NoRelatedSym（146）
  │
  ▼
OrderMatch.onMessage(MarketDataRequest, ...)
  │
  ├─ 校验订阅请求
  ├─ 保存 SessionID + MDReqID + Symbol + EntryType
  └─ 立即返回初始 MarketDataSnapshotFullRefresh（35=W）
       │
       ▼
Banzai.fromApp(...)
  │
  ▼
打印初始行情

之后由 OrderMatch 的行情发布任务驱动：

定时器 / 行情事件
  │
  ▼
读取当前价格
  │
  ▼
遍历有效订阅
  │
  ▼
向每个 Session 发送 MarketDataSnapshotFullRefresh（35=W）
  │
  ▼
Banzai.fromApp(...)
  │
  ▼
持续打印价格更新
```

### 4.7 第一版订阅更新的推荐实现边界

第一版先采用“重复发送完整快照”，不立即引入增量行情：

```text
第一版：
初始快照 -> 35=W
后续变化 -> 继续发送 35=W

后续进阶：
初始快照 -> 35=W
价格变化 -> 35=X MarketDataIncrementalRefresh
```

```text
第一版只支持：
├─ 一个 FIX Session
├─ 一个或少量 Symbol
├─ BID / OFFER / TRADE 中的一个或几个 EntryType
├─ 定时生成模拟价格，验证协议链路
└─ Banzai 控制台打印，不修改 UI
```

```text
不建议第一版直接做：
├─ 真实外部行情接入
├─ 复杂增量行情序列
├─ 多市场深度簿
└─ UI 行情表格
```

### 4.8 OrderMatch 侧的订阅生命周期

```text
MarketDataRequest（263=1）
  │
  ▼
SubscriptionRegistry
  └─ SessionID
      └─ MDReqID
          ├─ Symbol
          ├─ EntryType
          └─ Active
```

订阅更新：

```text
行情定时器 / 订单簿变化
  │
  ▼
MarketDataPublisher
  │
  ▼
查找有效订阅
  │
  ▼
构造 MarketDataSnapshotFullRefresh
  │
  ▼
Session.send(message)
```

取消订阅：

```text
Banzai
  │
  │ MarketDataRequest（263=2，携带原 MDReqID）
  ▼
OrderMatch.onMessage(MarketDataRequest, ...)
  │
  ▼
根据 SessionID + MDReqID 删除订阅
  │
  ▼
后续行情发布不再向该订阅发送
```

会话断开时清理：

```text
OrderMatch.onLogout(SessionID)
  │
  ▼
删除该 SessionID 下的全部订阅
  │
  ▼
避免定时任务向已断开的 Session 持续发送
```

### 4.9 行情价格来源的演进

```text
第一阶段：模拟行情
  123.45 -> 123.46 -> 123.44 -> 123.47
       │
       ▼
验证订阅建立、周期发送和 Banzai 接收
```

```text
第二阶段：OrderMatch 订单簿驱动

订单进入 OrderMatcher
  │
  ▼
Market.bidOrders / askOrders 变化
  │
  ▼
计算 Best Bid / Best Ask / Last Trade
  │
  ▼
生成行情事件
  │
  ▼
通知订阅者
```

```text
第三阶段：独立行情提供者

MarketDataProvider
  │
  ▼
接收外部行情
  │
  ▼
MarketDataEvent
  │
  ▼
订阅分发器
  │
  ▼
FIX Session
```

### 4.10 P1 订阅更新的完成标志

```text
- Banzai 发出 263=1 的 MarketDataRequest
- OrderMatch 保存 SessionID / MDReqID / Symbol 订阅关系
- OrderMatch 立即返回初始快照
- OrderMatch 定时推送后续行情
- Banzai 控制台能连续打印同一 Symbol 的价格更新
- Banzai 发出 263=2 后不再收到该订阅更新
- Logout 或断线后 OrderMatch 清理对应订阅
```

---

## 5. 第二优先级项目：多账户 / Block Order / Allocation

### 5.1 目标

把 Banzai 从“单账户单笔订单录入器”扩成“机构买方最小账户/组合下单台”。

```text
交易员下一个 block order
  -> 选择 Fund / Account / Desk
  -> 定义分配规则
  -> 先以一笔主单发出
  -> 回报回来后再做 allocation 展示或拆分记录
```

### 5.2 买方实战对应关系

```text
海外机构买方常见对象
├─ Fund / Portfolio
├─ SubAccount
├─ Trader / Desk
├─ Strategy
└─ Allocation Rule
```

### 5.3 在当前示例中的演进方式

```text
现有 Banzai Order
  -> 增加账户维度字段
  -> UI 增加 Fund / Account / Desk 选择
  -> 本地维护 Block Order -> Child Allocation 关系
  -> ExecutionReport 回来后归集到主单与分配视图
```

### 5.4 重点不是 FIX 字段本身，而是本地业务模型

```text
买方真实难点通常不在“会不会发 D 消息”
而在：
- 订单属于哪个账户
- 一笔主单如何分配给多个账户
- 回报回来后如何回写到 allocation 结构
```

### 5.5 最小完成标志

```text
- Banzai UI 能选账户
- 本地订单表能看到 account / fund / desk 维度
- 一笔主单能挂多个 allocation 记录
- 回报回来后能映射回对应 block order
```

---

## 6. 第三优先级项目：买方预交易风控

### 6.1 目标

在 Banzai 真正发出订单前，插入一个本地“风控前置层”。

```text
用户点击 Submit
  -> 本地风险检查
     ├─ 数量限制
     ├─ 名义金额限制
     ├─ 价格偏离检查
     ├─ 黑白名单检查
     └─ 市场状态检查
  -> 通过才发送 NewOrderSingle
  -> 不通过则本地拒单
```

### 6.2 为什么这是买方实战核心

```text
真实买方系统里
“发 FIX” 之前几乎一定先过风控 / 合规 / 限额控制

所以：
Banzai 如果加上风险前置
就会从“示例下单器”明显变成“真实交易前端雏形”
```

### 6.3 推荐的结构图

```text
OrderEntryPanel
  -> BanzaiApplication.send(order)
     -> PreTradeRiskEngine.check(order)
         -> pass  -> send42(order)
         -> reject -> 本地标记 REJECTED / 显示原因
```

### 6.4 最小完成标志

```text
- 至少实现 3 条风控规则
- UI 能展示拒绝原因
- 被本地拒绝的订单不会进入 Session.sendToTarget(...)
- 可对比“本地拒绝”和“对端拒绝”的区别
```

### 6.5 学习价值

```text
这一步会让你清楚区分：
- 买方本地拒单
- 卖方 / 交易所返回的拒单
- FIX 引擎层错误

这三者在真实系统里属于完全不同的层次
```

---

## 7. 第四优先级项目：多券商路由 / 多 Session 执行

### 7.1 目标

把 Banzai 从“单会话发单”扩成“按规则选择不同执行通道”的简化 EMS。

```text
订单进入路由层
  -> Router 按规则选择 Session
     ├─ Broker A
     ├─ Broker B
     └─ Venue C
  -> 发送到不同 FIX Session
  -> 回报统一归并回一个订单视图
```

### 7.2 为什么这是买方机构很关键的能力

```text
真实海外机构买方通常不会只有一个执行目的地
常见场景：
- 多券商竞争执行
- 不同市场走不同通道
- DMA / Algo / Care Order 分流
```

### 7.3 在当前示例中的可落地做法

```text
简单版
- 启动多个卖方会话
- Banzai 中增加 Broker / Route 选择
- 每个路由对应一个 SessionID

进阶版
- 不手选，由 Router 自动判断：
  - symbol 所属市场
  - 订单大小
  - 订单类型
  - 是否 urgent
```

### 7.4 推荐演进结构

```text
Banzai UI
  -> Order
  -> OrderRouter
       -> session A
       -> session B
       -> session C
  -> Session.sendToTarget(message, chosenSessionID)
```

### 7.5 最小完成标志

```text
- 本地存在至少 2 个可选执行 Session
- 同一 Banzai 可以把订单发往不同会话
- 回报能聚合回统一订单表
- 能看清“业务订单 ID”与“会话 / 通道”之间的关系
```

---

## 8. 第五优先级项目：交易后分配 / 审计 / Drop Copy

### 8.1 目标

把“下单成功并收到回报”继续延伸到买方 post-trade 视角。

```text
ExecutionReport
  -> 回写本地订单状态
  -> 更新成交记录
  -> 生成 allocation 结果
  -> 记录审计日志
  -> 可选发送/接收 drop copy
```

### 8.2 这部分为什么重要

```text
真实买方系统不止关心“发出去了没”
还关心：
- 成交了多少
- 分给了哪些账户
- 谁在什么时候做了什么动作
- 原始 FIX 报文是否可追溯
```

### 8.3 可落地的小版本

```text
本地先不做真正清算
只做：
- ExecutionReport 历史表
- Allocation 结果表
- 原始 FIX 报文留痕
- 用户操作审计表
```

### 8.4 最小完成标志

```text
- 可以查看某笔订单的完整状态变化历史
- 可以查看该订单关联的所有成交回报
- 可以看到本地分配结果
- 可以追溯原始发送/接收 FIX 文本
```

---

## 9. 每个项目和当前 examples 的映射关系

```text
P1 行情面板 + 发单联动
├─ Banzai：新增行情请求按钮、行情显示区、行情消息处理
└─ OrderMatch：复用已有 MarketDataRequest -> Snapshot 逻辑

P2 多账户 / Allocation
├─ Banzai：扩 UI 与本地订单模型
└─ Executor / OrderMatch：可先不改，只把回报映射回本地账户结构

P3 预交易风控
├─ Banzai：send(...) 前插入风险检查
└─ 对端示例基本无需修改

P4 多券商路由
├─ Banzai：增加 Router 与多 Session 选择
├─ Executor：可作为一个执行目的地
└─ OrderMatch：可作为另一个目的地 / Venue

P5 交易后分配 / 审计
├─ Banzai：扩展 ExecutionReport 落地与视图
└─ 可选增加单独的 drop copy 接收端
```

---

## 10. 推荐的实际执行顺序

```text
阶段 A：先补最小闭环
1. P1 行情面板 + 发单联动
2. P3 预交易风控

阶段 B：再补买方组织结构
3. P2 多账户 / Allocation

阶段 C：再补执行层能力
4. P4 多券商路由 / 多 Session

阶段 D：最后补 post-trade
5. P5 审计 / 分配 / Drop Copy
```

推荐原因：

```text
先做 P1：能最快把“行情、下单、回报”串成完整前台体验
先做 P3：能最快把 Banzai 从示例下单器提升为买方前置系统
P2 放中间：开始进入真实买方账户结构
P4 再推进：此时你已经理解 Session 与业务订单的区别
P5 最后补：更适合在前面主链稳定后加入
```

---

## 11. 每一阶段最值得打的断点

### P1 行情面板

```text
Banzai
- BanzaiApplication.fromApp(...)
- 新增的 MarketDataSnapshotFullRefresh 处理入口
- UI 更新行情表的位置

OrderMatch
- onMessage(MarketDataRequest, SessionID)
- MarketDataSnapshotFullRefresh 创建与发送位置
```

### P2 多账户 / Allocation

```text
Banzai
- OrderEntryPanel.actionPerformed(...)
- application.send(order)
- allocation 结构创建位置
- ExecutionReport 回写主单 / 子分配位置
```

### P3 预交易风控

```text
Banzai
- application.send(order)
- risk check 入口
- send42(order) 是否被调用
```

### P4 多券商路由

```text
Banzai
- router.chooseSession(order)
- Session.sendToTarget(message, chosenSessionID)
- 回报归并位置
```

### P5 审计 / 交易后处理

```text
Banzai
- executionReport(...)
- 历史状态落地位置
- 原始 FIX 文本记录位置
```

---

## 12. 一张压缩版脑图

```text
基于 Banzai 的买方实战演进
├─ 起点：Banzai = 下单前端
├─ 第一跳：补行情
│  └─ MarketDataRequest / Snapshot
├─ 第二跳：补风控
│  └─ 发单前检查
├─ 第三跳：补账户结构
│  └─ Fund / Account / Allocation
├─ 第四跳：补执行路由
│  └─ 多 Broker / 多 Session
└─ 第五跳：补交易后处理
   └─ 审计 / Drop Copy / Allocation 回写
```

---

## 13. 当前结论

```text
如果目标是：
“在 QuickFIX/J example 的基础上，做一个更贴近海外机构买方实战的小项目”

最合适的主线不是继续把 OrderMatch 做得更像交易所，
而是把 Banzai 逐步做成一个小型买方 OMS/EMS。
```

```text
一句话路线：
Banzai 下单器
  -> Banzai 行情+交易终端
  -> Banzai 带风控的买方前端
  -> Banzai 多账户 OMS
  -> Banzai 多通道路由 EMS
  -> Banzai 带审计与 post-trade 的小型买方平台
```
