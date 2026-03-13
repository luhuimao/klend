# Kamino Finance klend — 深度技术分析

> 项目路径：`programs/klend`  
> 框架：**Anchor 0.29.0**  Solana SDK **~1.17.18**  
> 程序 ID：`KLend2g3cP87fffoy8q1mQqGKjrxjC8boSyAYavgmjD`（主网），`SLendK7ySfcEzyaFqy93gDnD3RtrpXJcnRwb6zFHJSh`（Staging）  
> 审计方：OtterSec、Offside Labs、Certora、Sec3

---

## 一、项目概述

klend 是 Kamino Finance 在 Solana 上部署的**超额抵押借贷协议**，与 Aave v2/v3 在设计理念上高度相似，但针对 Solana 的账户模型和 Slots 时间做了深度适配。其核心职责是：

- 管理多个**资产储备池（Reserve）**，接受存款并发行 cToken (collateral token)；
- 维护**用户借贷仓位（Obligation）**，追踪抵押物和债务；
- 通过预言机实时刷新价格，执行清算和自动去杠杆；
- 提供**闪电贷（Flash Loan）**、**提款队列（Withdraw Queue）**、**借款订单（Borrow Order）**等高级功能。

---

## 二、项目结构

```
programs/klend/src/
├── lib.rs                    # 程序入口，声明所有指令
├── handlers/                 # 46 个指令处理器（每个指令独立文件）
│   ├── handler_deposit_reserve_liquidity.rs
│   ├── handler_borrow_obligation_liquidity.rs
│   ├── handler_liquidate_obligation_and_redeem_reserve_collateral.rs
│   ├── handler_flash_borrow_reserve_liquidity.rs
│   ├── handler_withdraw_queued_liquidity.rs
│   └── ... (共 46 个)
├── state/                    # 核心账户数据结构
│   ├── reserve.rs            # Reserve / ReserveLiquidity / ReserveCollateral / ReserveConfig
│   ├── obligation.rs         # Obligation / ObligationCollateral / ObligationLiquidity
│   ├── lending_market.rs     # LendingMarket / ElevationGroup
│   ├── liquidation_operations.rs  # 清算计算逻辑
│   └── ...
├── lending_market/           # 业务逻辑核心
│   ├── lending_operations.rs # 借贷核心操作（3732行）
│   ├── lending_checks.rs     # 账户合法性校验
│   ├── config_items.rs       # 配置项解析与验证
│   └── ...
└── utils/                    # 工具库
    ├── fraction.rs           # 高精度定点数
    ├── borrow_rate_curve.rs  # 利率曲线
    ├── consts.rs             # 全局常量
    ├── seeds.rs              # PDA 种子
    └── prices/               # 预言机集成
```

---

## 三、核心数据结构

### 3.1 LendingMarket（市场全局账户）

```
大小：4,656 bytes，zero_copy + repr(C)
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `lending_market_owner` | Pubkey | 管理员 |
| `quote_currency` | [u8;32] | 计价货币（如 "USD"） |
| `referral_fee_bps` | u16 | 推荐人费率（基点） |
| `emergency_mode` | u8 | 紧急模式开关 |
| `autodeleverage_enabled` | u8 | 自动去杠杆开关 |
| `liquidation_max_debt_close_factor_pct` | u8 | 单次清算最大比例（默认 20%） |
| `global_allowed_borrow_value` | u64 | 全局最大借贷价值上限（默认 $45M） |
| `elevation_groups` | [ElevationGroup;32] | 32 个提升组配置 |
| `min_net_value_in_obligation_sf` | u128 | 仓位净值下限（$0.000001） |
| `borrow_order_creation_enabled` | u8 | 借款订单开关 |
| `withdraw_ticket_issuance_enabled` | u8 | 提款票据发行开关 |
| `immutable` | u8 | 市场不可变开关（防止配置修改） |

---

### 3.2 Reserve（资产储备池）

```
大小：8,616 bytes，zero_copy + repr(C)
```

**Reserve** 是一个完整的资产池，包含三个主要子结构：

#### ReserveLiquidity（流动性状态）

| 字段 | 类型 | 说明 |
|------|------|------|
| `mint_pubkey` | Pubkey | 底层代币 Mint |
| `supply_vault` | Pubkey | 流动性金库 Token Account |
| `fee_vault` | Pubkey | 手续费金库 |
| `total_available_amount` | u64 | 当前可用流动性（原始单位） |
| `borrowed_amount_sf` | u128 | 已借出总量（scaled fraction 格式） |
| `market_price_sf` | u128 | 预言机价格（sf 格式） |
| `cumulative_borrow_rate_bsf` | BigFractionBytes | 累积借款利率（用于利息计算） |
| `accumulated_protocol_fees_sf` | u128 | 协议累积手续费 |
| `accumulated_referrer_fees_sf` | u128 | 推荐人累积手续费 |
| `deposit_limit_crossed_timestamp` | u64 | 存款上限被跨越的时间戳 |
| `borrow_limit_crossed_timestamp` | u64 | 借款上限被跨越的时间戳 |

#### ReserveCollateral（抵押物状态）

| 字段 | 说明 |
|------|------|
| `mint_pubkey` | cToken (collateral token) Mint 地址 |
| `supply_vault` | cToken 金库 Token Account |
| `mint_total_supply` | cToken 流通总量 |

#### ReserveConfig（储备池配置）

| 字段 | 说明 |
|------|------|
| `status` | Active / Obsolete / Deprecated |
| `asset_tier` | Regular / IsolatedCollateral / IsolatedDebt |
| `loan_to_value_pct` | 最大 LTV（如 75%) |
| `liquidation_threshold_pct` | 清算阈值（如 80%） |
| `liquidation_bonus_bps` | 清算人奖励（基点） |
| `deposit_limit / borrow_limit` | 存借上限 |
| `borrow_rate_curve` | 利率曲线（分段线性，最多 11 个节点） |
| `fees` | 手续费配置（原始费率、闪电贷费率、推荐人费率） |
| `token_info` | 代币信息 + 预言机配置（Pyth/Switchboard/Scope 三合一） |
| `debt_withdrawal_cap / deposit_withdrawal_cap` | 提款速率限制（时间窗口内的提款上限） |
| `elevation_groups` | 该 Reserve 可参与的 ElevationGroup 列表 |
| `borrow_factor_pct` | 借款风险因子（>100% 则增加风险调整后债务值） |
| `utilization_limit_block_borrowing_above_pct` | 利用率上限，超过后禁止新借款 |

另有两个重要字段：
- `borrowed_amount_outside_elevation_group` — 普通模式下的借款量
- `borrowed_amounts_against_this_reserve_in_elevation_groups[32]` — 各提升组的借款量

**提款队列（WithdrawQueue）** 也内嵌在 Reserve 中，用于异步提款场景。

---

### 3.3 Obligation（用户借贷仓位）

```
大小：3,336 bytes，zero_copy + repr(C)
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `tag` | u64 | 仓位类型标签（0=普通，1/2=特殊种子） |
| `owner` | Pubkey | 仓位所有者 |
| `deposits` | [ObligationCollateral; 8] | 最多 8 个抵押头寸 |
| `borrows` | [ObligationLiquidity; 5] | 最多 5 个借款头寸 |
| `deposited_value_sf` | u128 | 抵押物总价值（USD） |
| `borrow_factor_adjusted_debt_value_sf` | u128 | 风险调整后债务值（用于 LTV 计算） |
| `borrowed_assets_market_value_sf` | u128 | 实际借款市价总值 |
| `allowed_borrow_value_sf` | u128 | 允许的最大借款价值 |
| `unhealthy_borrow_value_sf` | u128 | 触发清算的债务值阈值 |
| `elevation_group` | u8 | 当前所在提升组（0=普通模式） |
| `autodeleverage_target_ltv_pct` | u8 | 自动去杠杆目标 LTV |
| `obligation_orders` | [ObligationOrder; 2] | 最多 2 个自动订单（止损/止盈） |
| `borrow_order` | BorrowOrder | 借款订单配置 |

**ObligationCollateral** 记录：抵押 Reserve、存入量、市场价值、在提升组中承载的债务量。

**ObligationLiquidity** 记录：借款 Reserve、累积借款利率快照、存款时间戳（用于债期计算）、借出量、市场价值。

---

## 四、利率机制

### 4.1 利率曲线（BorrowRateCurve）

利率曲线是**分段线性**的，最多支持 11 个 `(utilization_rate, borrow_rate_bps)` 锚点。协议每隔一个 Slot 复利累积一次：

```
compound_interest(borrow_rate, slots_elapsed)
≈ borrow_rate * slots_elapsed / SLOTS_PER_YEAR
```

使用近似复利公式（`approximate_compounded_interest`），避免昂贵的指数运算。

### 4.2 利率分配

```
总利息
  └─ 协议费（protocol_take_rate_pct）
  └─ 推荐人费（referral_fee_bps）
  └─ host_fixed_interest_rate（固定主机利率）
  └─ 剩余归存款人
```

---

## 五、核心指令集（60+条）

### 5.1 市场与储备管理

| 指令 | 说明 |
|------|------|
| `init_lending_market` | 创建市场账户，设置报价货币 |
| `update_lending_market` | 以 `(mode, value[72bytes])` 方式配置市场参数 |
| `init_reserve` | 创建资产储备池（含 cToken Mint、流动性金库等） |
| `update_reserve_config` | 更新储备配置（支持跳过完整性验证） |
| `seed_deposit_on_init_reserve` | 管理员首次注入流动性（防 cToken ratio 操纵） |
| `init_farms_for_reserve` | 为储备关联 Kamino Farms 激励合约 |

### 5.2 存款操作

| 指令 | 说明 |
|------|------|
| `deposit_reserve_liquidity` | 存入底层代币，获得 cToken |
| `deposit_obligation_collateral` | 将 cToken 存入 Obligation 作为抵押 |
| `deposit_reserve_liquidity_and_obligation_collateral` | 原子化：存入+抵押（组合指令） |
| `deposit_and_withdraw` | 组合：存入一种资产同时提取另一种 |

### 5.3 借款操作

| 指令 | 说明 |
|------|------|
| `borrow_obligation_liquidity` | 从储备借款（先刷新储备和仓位） |
| `repay_obligation_liquidity` | 还款，利息在还款时结算 |
| `repay_and_withdraw_and_redeem` | 原子化：还款+提取抵押+赎回底层代币 |

### 5.4 提款操作

| 指令 | 说明 |
|------|------|
| `withdraw_obligation_collateral` | 从 Obligation 提取抵押 cToken |
| `redeem_reserve_collateral` | 销毁 cToken，赎回底层代币 |
| `withdraw_obligation_collateral_and_redeem_reserve_collateral` | 原子化提款+赎回 |
| `enqueue_to_withdraw` | 加入提款队列（异步提款） |
| `withdraw_queued_liquidity` | 执行队列中的提款请求 |

### 5.5 清算与风控

| 指令 | 说明 |
|------|------|
| `liquidate_obligation_and_redeem_reserve_collateral` | 清算不健康仓位 |
| `mark_obligation_for_deleveraging` | 标记仓位需要自动去杠杆 |
| `socialize_loss_v2` | 协议承担坏账损失（社会化损失） |
| `refresh_reserve` | 刷新储备利率和价格 |
| `refresh_obligation` | 刷新仓位健康度指标 |

### 5.6 闪电贷

| 指令 | 说明 |
|------|------|
| `flash_borrow_reserve_liquidity` | 闪电贷借出（只允许非 CPI 调用） |
| `flash_repay_reserve_liquidity` | 同笔交易内还款+手续费 |

### 5.7 高级功能

| 指令 | 说明 |
|------|------|
| `set_obligation_order` | 设置止损/止盈自动订单 |
| `set_borrow_order` | 设置借款订单（限价/限速借款） |
| `fill_borrow_order` | 执行他人的借款订单 |
| `request_elevation_group` | 切换到提升组（eMode） |
| `init_referrer_state_and_short_url` | 创建推荐人账户 |
| `withdraw_referrer_fees` | 提取推荐人手续费 |

---

## 六、ElevationGroup（提升组 / eMode）

类似 Aave v3 的 eMode，允许相关资产（如 wBTC/BTC 衍生品）享受更高 LTV。

```rust
pub struct ElevationGroup {
    pub id: u8,                         // 1-32（0 保留为无组模式）
    pub ltv_pct: u8,                    // 提升组内的 LTV 上限
    pub liquidation_threshold_pct: u8, // 清算阈值
    pub max_liquidation_bonus_bps: u16,
    pub allow_new_loans: u8,            // 是否允许新借款
    pub max_reserves_as_collateral: u8, // 最大抵押资产种类
    pub debt_reserve: Pubkey,           // 只允许借该 Reserve 的代币
}
```

**关键约束**：
- 提升组内只能有**一种债务 Reserve**（单一债务确定性高）
- 在提升组内，`borrow_factor = 1.0`（不放大风险权重）
- 可设置 `debt_reserve` 为某特定 Reserve，强制只能借该资产

---

## 七、清算机制

### 7.1 清算触发条件

```
borrow_factor_adjusted_debt_value > unhealthy_borrow_value
```

其中 `unhealthy_borrow_value = deposited_value * liquidation_threshold_pct`。

### 7.2 清算参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `liquidation_max_debt_close_factor_pct` | 20% | 单次最多清算债务比例 |
| `max_liquidatable_debt_market_value_at_once` | $500,000 | 单次清算上限 |
| `liquidation_close_value` | $2 | 触发强制全额清算的下限 |
| `insolvency_risk_unhealthy_ltv_pct` | 95% | 接近资不抵债的 LTV 阈值 |

### 7.3 清算优先规则

- **债务优先级**：清算必须优先还清 `borrow_factor` 最高（风险最大）的债务
- **抵押物优先级**：优先没收 `liquidation_ltv` 最低（最容易出手）的抵押物
- **价格触发清算**：可通过 `price_triggered_liquidation_disabled` 开关禁用

### 7.4 自动去杠杆（Autodeleverage）

当价格剧烈波动时，管理员可通过 `mark_obligation_for_deleveraging` 将仓位标记为需要去杠杆，设置目标 LTV 和 margin call 启动时间戳，协议自动执行仓位降杠杆操作。

---

## 八、闪电贷

```
flash_borrow_reserve_liquidity  →  [用户指令] → flash_repay_reserve_liquidity
```

- 通过指令索引验证（`borrow_instruction_index`）确保归还在同笔交易内
- **禁止 CPI 调用**（`FlashBorrowCpi` / `FlashRepayCpi` 错误）
- 费率：`flash_loan_fee_sf`，若值为 `u64::MAX` 则禁用闪电贷

---

## 九、借款订单系统（BorrowOrder）

一种创新功能，允许用户以"限价单"方式配置借款：

```rust
// 用户设置借款订单
set_borrow_order(order_config, min_expected_current_remaining_debt_amount)

// 任何人（做市商/机器人）可执行
fill_borrow_order(ctx)
```

**订单配置约束**：
- `max_borrow_rate`：最高接受利率
- `min_debt_term`：最短债务期限
- `fill_time_limit`：订单有效期

---

## 十、提款队列（Withdraw Queue）

一种延迟提款机制，适用于流动性不足时：

1. 用户调用 `enqueue_to_withdraw`，cToken 被锁入 WithdrawQueue
2. 当有新存款流入，执行者调用 `withdraw_queued_liquidity` 满足提款请求
3. 若票据无效（如 Reserve 状态变更），可通过 `recover_invalid_ticket_collateral` 恢复抵押物

---

## 十一、预言机集成

`ReserveConfig.token_info` 支持三种预言机，采用加权/验证策略：

| 预言机 | 说明 |
|--------|------|
| **Pyth** | 主流价格来源，含置信区间检查 |
| **Switchboard v2** | 备用价格来源 |
| **Scope** | Kamino 自研聚合预言机 |

每个价格源都有 `max_age_price_seconds` 和 TWAP 对比阈值（`PriceTooDivergentFromTwap`），防止价格操纵。

价格状态标志（`PriceStatusFlags`）：
- 借款/清算等高风险操作要求 `ALL_CHECKS`（所有预言机正常）
- 仅存款/提款（无债务时）只需 `NONE`（允许价格稍旧）

---

## 十二、安全机制

### 12.1 版本控制

每个 Reserve 存储 `version: u64`，任何操作前会校验 `version == PROGRAM_VERSION`，防止状态混淆。

### 12.2 保鲜检查（Staleness Check）

每次操作强制要求在**当前 Slot** 刷新 Reserve 和 Obligation（`last_update.is_stale(slot)` 检查）。

### 12.3 金库余额验证

每笔 Token 转账都会在转账**前后**比对金库余额，验证三个不变量：
- `ReserveTokenBalanceMismatch`：金库实际余额变化是否符合预期
- `ReserveVaultBalanceMismatch`：vault balance 变化是否正确
- `ReserveAccountingMismatch`：内部账务记录是否一致

### 12.4 CPI 白名单

通过 `CPI_WHITELISTED_ACCOUNTS` 允许指定合约进行 CPI 调用（如 Kamino Vault、Squads 多签、Meteora LP 等），其他 CPI 默认禁止。

### 12.5 紧急模式

设置 `emergency_mode = 1` 后，大部分用户操作（借款、存款、提款等）将被 `#[access_control(emergency_mode_disabled)]` 拒绝。

### 12.6 不可变市场

`immutable` 字段设为 1 后，`update_lending_market` 将返回 `OperationNotPermittedMarketImmutable`，配置被永久锁定。

### 12.7 最小净值保护

obligation 操作后会检查 `net_value >= min_net_value_in_obligation_sf`（默认 $0.000001），防止粉尘攻击。

---

## 十三、数值精度

| 类型 | 精度 | 用途 |
|------|------|------|
| `Fraction` | 128-bit 定点数（`fixed` crate） | 常规计算，利率、价格 |
| `BigFraction` | 256-bit（U256） | 累积借款利率（防溢出） |
| `sf` 后缀字段 | `to_bits()` 存储的 Fraction 原始格式 | 链上存储节省空间 |

利率累积采用大数乘法：
```
borrowed_amount_sf_new = borrowed_amount_sf * new_cumulative_rate / old_cumulative_rate
```

---

## 十四、已弃用接口（Version Migration）

自 `v1.8.0` 起，大量指令被标记为 `#[deprecated]`，替换为 `_v2` 后缀版本：

| 旧指令 | 新指令 | 变化 |
|--------|--------|------|
| `deposit_obligation_collateral` | `_v2` | 账户结构重组（嵌套账户） |
| `borrow_obligation_liquidity` | `_v2` | 同上 |
| `liquidate...` | `_v2` | 同上 |

v2 版本使用 `DepositAccounts`、`BorrowAccounts` 等**嵌套账户结构**，提升 IDL 可读性。

---

## 十五、与 Aave v3 对比

| 特性 | Kamino klend | Aave v3 |
|------|-------------|---------|
| eMode | ElevationGroup (32组) | eMode (同类资产) |
| 清算机制 | 优先级规则 + close factor | close factor |
| 闪电贷 | 指令内验证 | 同 Tx 内回调 |
| 利率曲线 | 分段线性（11节点） | 双斜率 |
| 预言机 | Pyth+Switchboard+Scope | Chainlink+带TWAP |
| 自动去杠杆 | ✅（链上 margin call） | ❌ |
| 借款订单 | ✅（创新特性） | ❌ |
| 提款队列 | ✅（异步提款） | ❌ |
| 推荐人体系 | ✅（PDA referrer状态） | ❌ |
