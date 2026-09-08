# 生产级 FIX 核心问题与回答

本文整理五个生产环境中常见的 FIX 订单与会话问题。每个问题只保留两部分：架构图和精简回答。

---

## 1. 断线重连后 MsgSeqNum 相差几十万，应该如何处理？

### 架构图

```text
FIX Session 重连
      │
      ▼
比较双方 MsgSeqNum(34)
      │
      ├── 缺口较小
      │      ├─ 订单/成交/撤单结果：原样重传
      │      └─ 心跳/无需恢复消息：SequenceReset + GapFill
      │
      └── 缺口几十万
             │
             ▼
        进入恢复模式
             │
             ├─ 暂停新订单、改单、撤单
             ├─ ResendRequest(35=2) 分段请求
             ├─ 关键业务消息：重传或业务对账
             ├─ 行情：GapFill 后重新获取 Snapshot
             └─ 其余非业务区间：SequenceReset(35=4, 123=Y)
                              │
                              ▼
                     MsgSeqNum 对齐
                              │
                              ▼
                       恢复正常交易
```

### 精简回答

1. 不能无条件逐条重传几十万条消息。
2. 订单确认、成交回报和撤单结果：原样重传，或完成业务对账确认后恢复。
3. `Heartbeat`、`TestRequest` 和可恢复的行情增量：使用 `GapFill`，不逐条重传。
4. 行情：跳过历史增量，重新获取最新快照。
5. 使用 `ResendRequest(35=2)` 请求缺失区间：

   ```text
   7=BeginSeqNo
   16=EndSeqNo
   ```

6. 使用 `SequenceReset(35=4)` 跳过无需恢复的消息：

   ```text
   123=Y              // GapFillFlag
    36=NewSeqNo
   ```

7. 恢复期间暂停业务发送；序号恢复不等于订单状态恢复，还要完成订单和成交对账。

---

## 2. 一笔订单和对应撤单同时发出，先收到撤单 ExecutionReport，应该如何处理？

### 架构图

```text
买方本地订单 C001：NEW
        │
        ├── NewOrderSingle(35=D)
        │   ClOrdID(11)=C001
        │
        └── OrderCancelRequest(35=F)
            ClOrdID(11)=C002
            OrigClOrdID(41)=C001
                    │
                    ▼
             撤单回报先到达
                    │
          ┌─────────┼─────────┐
          │         │         │
          ▼         ▼         ▼
   Pending Cancel  Canceled  CancelReject
   150=6, 39=6     150=4     35=9
          │         │         │
          ▼         ▼         ▼
   等待最终结果   撤单终态   订单继续按成交结果处理
                    │
                    ▼
          迟到的 New 不得覆盖终态
```

### 精简回答

1. 使用 `ClOrdID(11)=C002` 标识当前撤单请求。
2. 使用 `OrigClOrdID(41)=C001` 指向被撤原订单。
3. 使用 `OrderID(37)` 关联卖方订单。
4. 收到 `ExecType=Pending Cancel(6)`：进入待撤状态，不立即标记为 `Canceled`。
5. 收到 `ExecType=Canceled(4)`：撤单成功，订单进入终态。
6. 收到 `OrderCancelReject(35=9)`：撤单失败，不能当作撤单成功。
7. 如果交易所先成交，撤单通常返回 `OrderCancelReject`。
8. 如果订单已经进入 `Canceled`，迟到的 `New` 回报不能把订单恢复为活动状态。
9. 最终状态必须由订单状态机和交易所实际处理顺序决定，不能简单按回报到达顺序覆盖。

---

## 3. 如何把一个公司的订单与资金账号关联起来？

### 架构图

```text
FIX Session 身份
49=BUY001                       // 买方公司/机构
56=SELL001                      // 卖方
        │
        ▼
NewOrderSingle(35=D)
11=C001                         // 买方订单号
 1=FUND-A-TRADING               // 订单资金/交易账号
        │
        ▼
卖方账户主数据
        │
        ├─ BUY001
        │    ├─ 公司：Buy Corp
        │    ├─ 资金账号：FUND-A-TRADING
        │    └─ 风控/额度/结算配置
        │
        ▼
订单与成交记录
SessionID + ClOrdID + Account
        │
        ▼
ExecutionReport(35=8)
11=C001
 1=FUND-A-TRADING
```

### 精简回答

1. `SenderCompID(49)` 表示 FIX Session 层的公司或机构身份。
2. `Account(1)` 表示本笔订单使用的资金或交易账号。
3. 账户主数据通过 `Account(1)` 关联公司、基金、策略、风控和结算信息。
4. 例如：

   ```text
   49=BUY001
   11=C001
    1=FUND-A-TRADING
   ```

5. 卖方通过 `BUY001 + FUND-A-TRADING` 查找具体公司和账户。
6. 一个公司管理多个基金或子账户时，每笔订单都应明确填写 `Account(1)`。
7. 需要表达基金、执行方、清算方等更细粒度参与者时，使用 `Parties` 重复组：

   ```text
   453=NoPartyIDs
   448=PartyID
   447=PartyIDSource
   452=PartyRole
   ```

8. FIX 不会自动判断账号属于哪个公司，双方必须通过 Session 身份、账号约定和账户主数据完成关联。

---

## 4. 一笔下单涉及哪些主要要素，以及对应的 Tag/Value？

### 架构图

```text
订单请求
  │
  ├─ FIX Header：会话与消息身份
  │    8 / 9 / 35 / 34 / 49 / 52 / 56 / 10
  │
  ├─ NewOrderSingle(35=D)：订单意图
  │    11 / 1 / 55 / 54 / 38 / 40 / 44 / 99 / 59 / 60
  │
  ├─ 卖方/交易所身份映射
  │    ClOrdID(11) -> OrderID(37)
  │
  ├─ 执行回报 ExecutionReport(35=8)
  │    17 / 37 / 11 / 150 / 39 / 32 / 31 / 14 / 151 / 6
  │
  └─ 撤单/改单
       OrderCancelRequest(35=F)：11 / 41 / 37
       OrderCancelReplaceRequest(35=G)：11 / 41 / 37 + 修改后的订单字段
```

### 精简回答

1. 说明：以下是常用的 FIX 4.2 订单要素；是否必填以 `FIX42.xml`、对手方协议和订单类型为准。
2. 会话 Header：

   | Tag | 字段 | 示例 Value | 含义 |
   |---:|---|---|---|
   | 8 | BeginString | `FIX.4.2` | FIX 版本 |
   | 9 | BodyLength | `...` | Body 长度，由引擎计算 |
   | 35 | MsgType | `D` | `NewOrderSingle` |
   | 34 | MsgSeqNum | `101` | 当前方向消息序号 |
   | 49 | SenderCompID | `BUY001` | 发送方机构身份 |
   | 52 | SendingTime | `20260908-09:30:00.000` | 发送时间 |
   | 56 | TargetCompID | `SELL001` | 接收方机构身份 |
   | 10 | CheckSum | `...` | 报文校验和，由引擎计算 |

3. `NewOrderSingle(35=D)`：

   | Tag | 字段 | 示例 Value | 含义 |
   |---:|---|---|---|
   | 11 | ClOrdID | `C001` | 当前发单方客户订单号 |
   | 1 | Account | `FUND-A-TRADING` | 资金/交易账号 |
   | 21 | HandlInst | `1` | 处理指示 |
   | 55 | Symbol | `600000` | 证券/合约 |
   | 54 | Side | `1` | `1=Buy`、`2=Sell` |
   | 38 | OrderQty | `100` | 订单数量 |
   | 40 | OrdType | `2` | `1=Market`、`2=Limit` |
   | 44 | Price | `10.50` | 限价，限价单通常需要 |
   | 99 | StopPx | `10.00` | 止损价，止损单使用 |
   | 59 | TimeInForce | `0` | `0=Day`、`1=GTC`、`3=IOC` |
   | 60 | TransactTime | `20260908-09:30:00.000` | 交易时间 |
   | 114 | LocateReqd | `N` | 卖空定位需求，按协议使用 |
   | 453 | NoPartyIDs | `...` | 参与方重复组数量 |
   | 448 | PartyID | `FUND-A` | 参与方标识，位于 Parties 组内 |
   | 447 | PartyIDSource | `D` | 参与方标识来源 |
   | 452 | PartyRole | `...` | 参与方角色 |

4. `ExecutionReport(35=8)`：

   | Tag | 字段 | 示例 Value | 含义 |
   |---:|---|---|---|
   | 37 | OrderID | `S1001` | 接收方分配的订单号 |
   | 17 | ExecID | `E001` | 本次执行事件编号 |
   | 11 | ClOrdID | `C001` | 关联买方订单 |
   | 41 | OrigClOrdID | `C001` | 关联原订单版本，改单/撤单场景使用 |
   | 20 | ExecTransType | `0` | FIX 4.2 执行交易类型 |
   | 150 | ExecType | `0/1/2/4/6` | New/Partial/Fill/Canceled/Pending Cancel |
   | 39 | OrdStatus | `0/1/2/4/6/8` | New/Partial/Filled/Canceled/Pending Cancel/Rejected |
   | 32 | LastShares | `100` | FIX 4.2 本次成交数量 |
   | 31 | LastPx | `10.50` | 本次成交价格 |
   | 14 | CumQty | `100` | 累计成交数量 |
   | 151 | LeavesQty | `0` | 剩余数量 |
   | 6 | AvgPx | `10.50` | 平均成交价格 |
   | 1 | Account | `FUND-A-TRADING` | 关联资金/交易账号 |
   | 58 | Text | `...` | 补充说明 |

5. 撤单与改单：

   | MsgType | 消息 | 关键 Tag |
   |---|---|---|
   | `F` | OrderCancelRequest | `11=ClOrdID`、`41=OrigClOrdID`、`37=OrderID`、`55=Symbol`、`54=Side` |
   | `G` | OrderCancelReplaceRequest | `11`、`41`、`37`，以及新的 `38`、`40`、`44`、`59` |
   | `9` | OrderCancelReject | `11`、`41`、`37`、`39`、`434`、`102` |

6. 核心关联：

   ```text
   Account(1)       = 资金账号
   ClOrdID(11)      = 当前请求标识
   OrigClOrdID(41)  = 原订单版本
   OrderID(37)      = 接收方订单标识
   ExecID(17)       = 执行事件标识
   OrdStatus(39)    = 当前订单状态
   ExecType(150)    = 本次状态事件
   ```

---

## 5. 灾备切换时，如何保存每一条消息并避免丢失或重复处理？

### 架构图

```text
                         同步复制
              ┌────────────────────────┐
              │                        │
              ▼                        ▼
┌──────────────────────┐      ┌──────────────────────┐
│ Active 主交易中心     │      │ Standby 灾备中心      │
│                      │      │                      │
│ QuickFIX/J Session   │      │ QuickFIX/J Session   │
│ MessageStore         │─────>│ MessageStore Replica │
│ FIX Message Journal  │─────>│ Journal Replica      │
│ Order/Execution DB   │─────>│ Business DB Replica  │
│ 唯一发送权           │      │ 无发送权              │
└──────────┬───────────┘      └──────────┬───────────┘
           │                             │
           └────── fencing / 租约 ───────┘
                         │
                         ▼
                  主节点故障后接管
                         │
       加载序号、原始报文、订单状态、处理状态
                         │
       Logon / Resend / 业务对账 / 恢复交易
```

### 精简回答

1. 分开保存三类数据：
   - `MessageStore`：保存 `MsgSeqNum` 和 FIX 历史报文，用于 Session 重传；
   - `FIX Message Journal`：保存每条入站、出站原始 FIX 报文，用于审计和重放；
   - 业务数据库：保存订单、成交、持仓、资金和状态变更。
2. 入站消息处理：收到报文后写入 `Journal / Inbox`，同步复制到灾备，完成业务处理，更新订单与成交，最后标记 `PROCESSED`。
3. 出站消息处理：生成订单意图，持久化原始报文和 `ClOrdID`，同步复制后发送网络，最后标记 `SENT`。
4. 主备切换：
   - fencing 旧主节点；
   - 灾备节点获得唯一 Session 发送权；
   - 恢复 `MessageStore` 和业务数据库；
   - 处理 `RECEIVED`、`PROCESSING`、`UNKNOWN` 消息；
   - 执行 FIX Logon、Resend 和业务对账；
   - 恢复新订单发送。
5. 使用以下唯一键和幂等机制避免重复：

   ```text
   (SessionID, MsgSeqNum)
   (SessionID, ExecID)
   (OrderID, EventID)
   ```

6. 如果要求订单和成交消息 `RPO=0`，必须确认消息已同步持久化到灾备副本后，才将其视为可靠。
7. 网络发送后主机立即宕机会出现 `UNKNOWN` 状态，此时必须查询交易所或对手方，不能直接生成新的 `ClOrdID` 重发。
8. 最终原则：`MessageStore` 恢复 FIX 会话，`Journal` 恢复原始消息事实，业务数据库恢复订单状态，同步复制保证灾备数据最新，幂等与对账避免重复和遗漏。

---

## 6. 卖方已向买方返回 NEW，但到交易所的 Session 断线，如何避免重复下单？

### 架构图

```text
卖方发送交易所订单
        │
        ▼
交易所方向 Session 断线
        │
        ▼
订单状态 = UNKNOWN
        │
        ├─ 1. 查询确认已存在
        │      └─ 记录 OrderID(37)，禁止重发
        │
        ├─ 2. 查询确认未存在
        │      └─ 使用原 ClOrdID 补发
        │
        └─ 3. 无法确认
               └─ 暂停重发，继续对账
```

### 精简回答

1. 买方收到卖方 `OrdStatus=NEW`，只说明卖方已接单，不代表交易所已接收。
2. 断线后，卖方将交易所方向订单标记为 `UNKNOWN`，并保留原 `ClOrdID=R2001`。
3. 重连后分类处理：
   - 交易所已存在：记录 `OrderID(37)`，禁止重复下单；
   - 明确未存在：使用原 `ClOrdID=R2001` 补发，不生成新的 `ClOrdID`；
   - 无法确认：保持 `UNKNOWN`，暂停重发并继续对账。
4. 核心原则：`ClOrdID + 状态查询 + 幂等控制`，不能把本地发送成功当成交易所已接收。
