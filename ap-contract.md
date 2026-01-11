# AlianaProtocol 智能合约技术规格说明书 (Technical Specification)

本文档为 `AlianaProtocol.sol` 智能合约的详细技术参考手册，旨在为前端开发人员、合约审计人员及维护人员提供详尽的协议逻辑说明。

---

## 1. 协议概览 (Protocol Overview)

AlianaProtocol 是一个基于 BSC 网络的去中心化理财协议，采用 "4% 本金返还 + X% 净利润" 的双轨收益模型，结合多层级动态奖励与自动化风控机制。

*   **核心资产**: USDT (BEP20)
*   **合约地址**: (待部署)
*   **USDT 地址**: `0x55d398326f99059fF775485246999027B3197955`
*   **启动时间**: 2025-12-09 19:00:00 GMT+0
*   **投资周期**: 25 天 (固定)

---

## 2. 核心数据结构 (Data Structures)

合约使用以下核心结构体来管理用户状态与资金流。

### 2.1 用户档案 (`UserProfile`)
记录用户的身份信息、层级关系及资产总览。

```solidity
struct UserProfile {
    address sponsor;             // 推荐人地址 (0x0 表示无推荐人或顶级账号)
    string username;             // 用户名 (唯一, 3-20字符, 小写字母/数字)
    bool isRegistered;           // 注册状态标识
    uint256 activeInvestments;   // 当前活跃本金总额 (所有未过期订单本金之和)
    uint256 teamActiveVolume;    // 团队(无限代)当前活跃业绩
    uint256 teamTotalVolume;     // 团队(无限代)历史累计业绩
    uint256 totalDeposited;      // 个人历史累计充值/复投总额
    uint256 totalWithdrawn;      // 个人历史累计提现总额
    InvestmentPosition[] investmentPositions; // 个人所有投资订单数组
}
```

### 2.2 投资订单 (`InvestmentPosition`)
每一笔资金入场（存款或复投）都会生成一个独立的订单记录。

```solidity
struct InvestmentPosition {
    uint256 principalAmount;     // 订单本金数量 (Wei)
    uint256 startTimestamp;      // 订单创建时间戳
    uint256 lastRewardTimestamp; // 上次结算收益的时间戳 (Checkpoint)
    uint256 accumulatedRewards;  // 该订单累计已产生的静态收益
    uint8 sourceType;            // 资金来源: 
                                 // 0 = 钱包直接充值 (Wallet Deposit)
                                 // 1 = 静态收益复投 (Daily Compound)
                                 // 2 = 动态奖励复投 (Network Compound)
    bool isActive;               // 状态: true=收益周期内, false=已过期
}
```

### 2.3 奖励账户 (`UserRewards`)
记录用户各类待领取的收益余额。

```solidity
struct UserRewards {
    uint256 referralCommissions;   // 推荐佣金余额 (直推/层级奖)
    uint256 onboardingBonuses;     // 新手奖余额 (给推荐人的奖励)
    uint256 leadershipBonuses;     // 级差奖余额 (团队管理奖)
    uint256 dailyCapitalReserve;   // 静态收益-本金返还部分余额 (4%/天)
    uint256 dailyROIReserve;       // 静态收益-净利润部分余额 (1-2%/天)
    uint256 networkRewardsReserve; // 已归集的动态奖励余额 (历史遗留字段)
    uint32 teamMemberCount;        // 团队总人数 (无限代)
    uint32 directReferralsCount;   // 直推总人数
    uint32 qualifiedDirectsCount;  // 有效直推人数 (满足活跃本金要求的直推)
    uint8 leadershipRank;          // 当前领导等级 (0-6)
}
```

### 2.4 提现统计 (`UserWithdrawalStats`)
记录用户各类资金的流出统计，用于数据分析。

```solidity
struct UserWithdrawalStats {
    uint256 dailyRewardsWithdrawn;  // 已提现的静态收益总额
    uint256 dailyRewardsCompounded; // 已复投的静态收益总额
    uint256 referralWithdrawn;      // 已提现的推荐佣金总额
    uint256 onboardingWithdrawn;    // 已提现的新手奖总额
    uint256 leadershipWithdrawn;    // 已提现的级差奖总额
    uint256 networkCompounded;      // 已复投的动态奖励总额
}
```

### 2.5 动态参数 (`DynamicParams`)
用于 HealthController 动态调整的风控参数。

```solidity
struct DynamicParams {
    uint16 adminFeeBps;             // 管理费率 (基点, e.g. 500 = 5%)
    uint16 reserveFeeBps;           // 储备金费率 (基点)
    uint256 minWithdrawAmount;      // 最小提现金额
    uint256 dailyUserWithdrawLimit; // 单用户单日最大提现额 (0 = 不限)
}
```

---

## 3. 常量与配置 (Constants & Config)

| 常量名 | 值 | 说明 |
| :--- | :--- | :--- |
| `INVESTMENT_CYCLE_DAYS` | 25 | 静态收益周期 (天) |
| `MIN_DEPOSIT_AMOUNT` | 10 USDT | 最小单笔存款金额 |
| `MAX_DEPOSIT_AMOUNT` | 2000 USDT | **单用户钱包充值最大活跃本金限制** |
| `MIN_WITHDRAW_AMOUNT` | 1 USDT | 最小提现金额 (动态可调) |
| `REFERRER_MINIMUM_DEPOSIT` | 10 USDT | 成为推荐人的最低存款门槛 |
| `ADMIN_FEE_PERCENT` | 5% | 默认管理费 |
| `BASE_DAILY_RETURN_PERCENT` | 4% | 固定每日回本比例 |
| `MAX_USER_POSITIONS` | 100 | 单用户最大订单数限制 |

---

## 4. 核心功能逻辑 (Core Functions)

### 4.1 用户注册 (`registerUser`)
*   **输入**: `username` (用户名), `sponsorUsername` (推荐人用户名)
*   **前置检查**:
    1.  协议已启动。
    2.  调用者未注册。
    3.  用户名格式合法且未被占用。
    4.  推荐人存在、已注册且满足 `REFERRER_MINIMUM_DEPOSIT` 门槛。
*   **逻辑**:
    1.  创建 `UserProfile`。
    2.  绑定 `usernameToWallet`。
    3.  记录推荐关系。
    4.  触发 `UserRegistered` 事件。

### 4.2 存款/投资 (`makeDeposit`)
*   **输入**: `amount` (金额)
*   **前置检查**:
    1.  金额 ≥ `MIN_DEPOSIT_AMOUNT`。
    2.  **额度检查**: `_getActiveWalletDeposits(user) + amount <= MAX_DEPOSIT_AMOUNT`。
        *   *注: 仅统计 `sourceType=0` (钱包充值) 的活跃订单，复投订单不占用此额度。*
    3.  用户已注册。
    4.  订单数 < `MAX_USER_POSITIONS`。
*   **资金流**:
    1.  `User -> Contract`: 转入 `amount`。
    2.  `Contract -> Treasury`: 转出 `amount * adminFeeBps` (5%) 作为管理费。
        *   *注: 手续费由协议额外支付，不从用户本金扣除。*
    3.  `Contract -> Reserve`: 转出储备金 (若开启)。
*   **核心逻辑**:
    1.  若非首单，先结算之前未领取的静态收益。
    2.  创建新 `InvestmentPosition` (`sourceType=0`)。
    3.  更新个人及团队业绩 (`_updateTeamVolumeStats`)。
    4.  发放层级佣金 (`_distributeReferralCommissions`)。
    5.  若是首单，发放新手奖 (`_distributeOnboardingBonus`) 并更新团队人数。

### 4.3 领取静态收益 (`claimDailyRewards`)
*   **输入**: `amount` (提取金额, 0 表示全部)
*   **前置检查**:
    1.  余额充足。
    2.  金额 ≥ `minWithdrawAmount`。
    3.  **风控检查**: `userDailyWithdraws[day] + amount <= dailyUserWithdrawLimit` (若已开启软限流)。
*   **核心逻辑**:
    1.  `_calculateAndUpdatePendingRewards`: 实时计算并累加最新收益。
    2.  按比例扣除 `dailyCapitalReserve` (本金) 和 `dailyROIReserve` (利润)。
    3.  **触发级差奖**: 基于本次提取的 **ROI 部分**，计算并立即发放领导奖 (`_distributeLeadershipBonuses`) 给上级。
    4.  `Contract -> User`: 转账 `amount`。
    5.  `Contract -> Treasury`: 转账手续费 (协议承担)。

### 4.4 领取动态奖励 (`claimNetworkRewards`)
*   **输入**: `amount` (提取金额, 0 表示全部)
*   **包含奖励类型**: 直推奖、层级奖、新手奖、级差奖。
*   **前置检查**: 同上。
*   **核心逻辑**:
    1.  汇总所有动态奖励余额。
    2.  扣除余额。
    3.  `Contract -> User`: 转账 `amount`。
    4.  `Contract -> Treasury`: 转账手续费 (协议承担)。

### 4.5 复投 (`compoundDailyRewards` / `compoundNetworkRewards`)
*   **逻辑**:
    1.  不进行 USDT 转账，直接在合约内部划转。
    2.  创建新 `InvestmentPosition` (`sourceType=1` 或 `2`)。
    3.  **关键点**: 复投订单**不占用** `MAX_DEPOSIT_AMOUNT` 额度。
    4.  复投金额同样会产生层级佣金 (`_distributeReferralCommissions`) 并计入业绩。

---

## 5. 经济模型详解 (Economic Model Details)

### 5.1 静态收益计算 (Static Rewards)
每日收益由两部分组成，基于用户的活跃本金总额所在的 **Tier** 决定。

$$ \text{Total Daily Return} = \text{Base Return (4\%)} + \text{Profit ROI (1\%-2\%)} $$

| Tier | 活跃本金区间 (USDT) | Base Return | Profit ROI | 总日化 |
| :--- | :--- | :--- | :--- | :--- |
| 0 | 0 - 499 | 4% | **1.00%** | 5.00% |
| 1 | 500 - 999 | 4% | **1.25%** | 5.25% |
| 2 | 1,000 - 2,499 | 4% | **1.50%** | 5.50% |
| 3 | 2,500 - 4,999 | 4% | **1.75%** | 5.75% |
| 4 | ≥ 5,000 | 4% | **2.00%** | 6.00% |

*   **结算方式**: 秒级结算。`Available = Principal * Rate * (Now - LastCheckTime) / 1 Day`。
*   **生命周期**: 每个订单独立计算 25 天，到期后停止产生收益。

### 5.2 动态奖励体系 (Dynamic Rewards)

#### A. 层级佣金 (Referral Commission)
基于**新增业绩** (存款/复投) 发放。
*   **层数**: 20 层。
*   **比例**: L1 (1.5%), L2 (1.25%), L3-5 (1%), L6-10 (1%), L11-15 (0.5%), L16-20 (0.25%)。
*   **考核**:
    *   拿 L1-5: 自身活跃本金 ≥ 10 U。
    *   拿 L6-10: 有效直推 ≥ 2 人。
    *   拿 L11-15: 有效直推 ≥ 3 人。
    *   拿 L16-20: 有效直推 ≥ 5 人。

#### B. 领导奖 (Leadership Bonus) - 级差制
基于**下级提取静态收益时的 Profit (利润) 部分**发放。
*   **触发**: 下级调用 `claimDailyRewards` 时。
*   **基数**: 下级提取金额中的 ROI 部分。
*   **算法**: `Bonus = ROI_Amount * (UserRankRate - DownlineRankRate)`。
*   **等级表**:
    *   Rank 1: 5% (需 50 人团队 + 2.5k 业绩)
    *   Rank 2: 10% (需 100 人团队 + 5k 业绩)
    *   ...
    *   Rank 7: 50% (最高)

---

## 6. 风控与健康监控 (Health & Risk Control)

合约集成了 `HealthController` 接口，支持动态参数调整。

### 6.1 净流量监控 (`Net Flow`)
合约记录每日资金净流量 `dailyNetFlow`。
*   `Deposit`: +Amount
*   `Withdraw`: -Amount

### 6.2 软限流机制 (`Soft Limits`)
当 `dailyUserWithdrawLimit > 0` 时，合约自动开启软限流模式：
*   **限制对象**: `claimDailyRewards`, `claimNetworkRewards`。
*   **逻辑**: 检查用户当日累计提现额是否超过限额。
*   **目的**: 防止大户在恐慌期挤兑，平滑资金流出。

### 6.3 费用调节
*   **Admin Fee**: 可调 (默认 5%)。
*   **Reserve Fee**: 可调 (默认 0%)，用于强制储蓄以补充流动性。

---

## 7. 交互接口 (View Functions for Frontend)

### 7.1 额度查询
```solidity
function getRemainingDepositQuota(address user) external view returns (uint256);
```
返回用户当前还能充值多少 USDT (基于 2000 U 钱包充值上限)。

### 7.2 用户基础信息 (`getUserProfile`)
```solidity
function getUserProfile(address user) external view returns (
    string memory username,
    address sponsor,
    bool isRegistered,
    uint256 teamActiveVolume,
    uint256 totalDeposited,
    uint256 totalWithdrawn,
    uint32 directReferralsCount,
    uint32 teamMemberCount
);
```

### 7.3 用户投资状态 (`getUserInvestmentInfo`)
```solidity
function getUserInvestmentInfo(address user) external view returns (
    uint256 activeInvestmentsTotal, // 当前活跃本金总额
    uint8 leadershipRank,           // 当前领导等级 (0-6)
    uint8 roiTierLevel,             // 当前收益等级 (0-4)
    uint256 activePositionsCount,   // 活跃订单数量
    uint256 dailyReturnPercentage,  // 当前日化率 (基点, e.g. 500 = 5%)
    uint256 estimatedDailyReturn    // 预估每日静态收益 (Wei)
);
```

### 7.4 奖励详情查询 (`getUserRewardDetails`)
```solidity
function getUserRewardDetails(address user) external view returns (
    uint256 referralCommissions,
    uint256 onboardingBonuses,
    uint256 leadershipBonuses,
    uint256 dailyCapitalReserve,
    uint256 dailyROIReserve,
    uint256 networkRewardsReserve
);
```

---

## 8. 错误码速查 (Error Codes)

| Error String | Meaning |
| :--- | :--- |
| `Platform not launched` | 当前时间早于启动时间 (2025-12-09) |
| `Already registered` | 用户重复注册 |
| `Username already taken` | 用户名冲突 |
| `Sponsor not found` | 推荐人地址不存在 |
| `Sponsor not qualified` | 推荐人未满足 10 U 活跃本金要求 |
| `Exceeds maximum wallet deposit` | 超过 2000 U 钱包充值额度 |
| `Below minimum deposit` | 存款金额 < 10 U |
| `Daily limit exceeded` | 触发单日提现风控限额 |
| `Insufficient balance` | 提现/复投余额不足 |
| `No rewards available` | 当前无可领奖励 |
| `USDT transfer failed` | 代币转账失败 (通常是合约缺 Gas 或缺余额) |
