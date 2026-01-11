# AlianaProtocol 智能合约说明文档

本文档详细说明了 `AlianaProtocol.sol` 合约的核心参数配置、功能模块及交互逻辑。

## 一、核心常量与配置 (Core Constants)

### 1.1 基础配置
| 参数名 | 值 | 说明 |
| :--- | :--- | :--- |
| `USDT_TOKEN` | `0x55d398326f99059fF775485246999027B3197955` | 绑定的 USDT 代币地址 (BSC 主网) |
| `launchTimestamp` | `1765306800` (2025-12-09) | 项目启动时间戳 |
| `TREASURY_WALLET` | `0x033d...` | 协议金库地址 (用于接收管理费等) |

### 1.2 资金限制
| 参数名 | 值 | 说明 |
| :--- | :--- | :--- |
| `MIN_DEPOSIT_AMOUNT` | 10 USDT | 最小单笔存款金额 |
| `MAX_DEPOSIT_AMOUNT` | 3000 USDT | **单钱包最大有效本金上限** (超过此金额无法继续存款) |
| `MIN_WITHDRAW_AMOUNT` | 1 USDT | 最小提现金额 (可动态调整) |
| `dailyUserWithdrawLimit`| 0 (默认) | **单日提现软限流** (0=无限制，触发风控时会自动下调) |

### 1.3 周期与费率
| 参数名 | 值 | 说明 |
| :--- | :--- | :--- |
| `INVESTMENT_CYCLE_DAYS`| 25 天 | 投资周期 (25天后本金释放/结束) |
| `ADMIN_FEE_PERCENT` | 5% (500 bps) | 管理费率 (提现时扣除) |
| `BASE_DAILY_RETURN_PERCENT`| 4% (400 bps) | 基础日化收益率 |

---

## 二、动态收益体系 (Dynamic ROI)

用户的日化收益率由 **基础日化** + **等级加成** 组成。

### 2.1 投资等级 (Tier System)
根据用户当前的**总存款金额**自动判定等级：

| 等级 (Tier) | 累计存款门槛 | 额外加成 (ROI Bonus) | 总日化 (Total ROI) |
| :--- | :--- | :--- | :--- |
| Tier 0 | 0 - 499 U | +1.00% | **5.00%** |
| Tier 1 | 500 - 999 U | +1.25% | **5.25%** |
| Tier 2 | 1000 - 2499 U | +1.50% | **5.50%** |
| Tier 3 | 2500 - 4999 U | +1.75% | **5.75%** |
| Tier 4 | ≥ 5000 U | +2.00% | **6.00%** |

---

## 三、推荐奖励机制 (Referral System)

### 3.1 推荐佣金 (20层)
根据推荐关系链，上级可获得下级**收益提现额**的百分比奖励（非本金）。

*   **第 1-5 层**：1.5% ~ 1.0%
*   **第 6-10 层**：1.0% (需直推 2 人)
*   **第 11-15 层**：0.5% (需直推 3 人)
*   **第 16-20 层**：0.25% (需直推 5 人)

### 3.2 团队领导奖 (Leadership Bonus)
当团队业绩达到标准，可获得无限代极差奖励。

| 团队人数 | 团队业绩 (USDT) | 奖励比例 |
| :--- | :--- | :--- |
| 50 | 2,500 | 0.5% |
| 100 | 5,000 | 1.0% |
| ... | ... | ... |
| 2,500 | 100,000 | 5.0% |

---

## 四、核心功能接口 (Core Functions)

### 4.1 用户操作
*   **`register(address sponsor, string username)`**
    *   **功能**：注册新用户。
    *   **限制**：用户名 3-20 字符，只能包含字母数字。必须有有效推荐人（首个用户除外）。

*   **`makeDeposit(uint256 amount)`**
    *   **功能**：存入 USDT 进行投资。
    *   **限制**：
        1. 金额 ≥ `MIN_DEPOSIT_AMOUNT`。
        2. 当前钱包有效本金 + 新存金额 ≤ `MAX_DEPOSIT_AMOUNT`。
    *   **逻辑**：创建新的 `InvestmentPosition`，记录开始时间。

*   **`claimDailyRewards(uint256 amount)`**
    *   **功能**：提取静态收益 (ROI)。
    *   **风控**：受 `dailyUserWithdrawLimit` (日限额) 限制。
    *   **费用**：扣除管理费 (`adminFeeBps`) 和储备金 (`reserveFeeBps`)。

*   **`compoundDailyRewards(uint256 amount)`**
    *   **功能**：复投静态收益。
    *   **优势**：复投金额计入 `activeInvestments`，产生新收益，且不扣除管理费。

*   **`claimNetworkRewards(uint256 amount)`**
    *   **功能**：提取动态奖励 (推荐奖 + 领导奖)。
    *   **风控**：受 `dailyUserWithdrawLimit` 限制。

### 4.2 查询功能 (View Functions)
*   **`getDailyRewardsBalance(address user)`**
    *   **功能**：查询当前待领取的静态收益总额。
    *   **逻辑**：遍历所有活跃订单，实时计算应付利息。

*   **`getRemainingDepositQuota(address user)`**
    *   **功能**：查询用户当前还可以存多少钱。
    *   **公式**：`MAX_DEPOSIT_AMOUNT - 当前钱包有效本金`。

*   **`dailyNetFlow(uint256 day)`**
    *   **功能**：查询指定日期的资金净流量（充值 - 提现）。
    *   **用途**：健康监控系统用于判断资金流向。

---

## 五、健康监控与风控 (Health & Risk Control)

合约集成了自动化的健康监控系统，可动态调整参数以应对风险。

### 5.1 动态参数 (`DynamicParams`)
由 `HealthController` 合约根据全网偿付率自动设置：
1.  **`adminFeeBps`**: 管理费率。
2.  **`reserveFeeBps`**: 储备金留存率 (风险越高，留存越多)。
3.  **`minWithdrawAmount`**: 最小提现门槛。
4.  **`dailyUserWithdrawLimit`**: **单用户日提现硬顶** (关键风控手段)。

### 5.2 交互钩子
*   **`miningController`**: 挖矿控制器，每次充值/复投时通知，用于计算 vALI 挖矿产出。
*   **`healthController`**: 健康控制器，拥有调整动态参数的权限。
