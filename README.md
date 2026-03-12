# 交易室提成管理系统 — DTPPro8

交易室 31O-1232（Real Trading / DTTW Metro）专用提成计算与保证金管理系统。

---

## 系统架构

| 层级 | 技术 | 用途 |
|------|------|------|
| 前端 | React + Vite + TypeScript | 管理员后台 + 交易员移动端 |
| 数据库 | Supabase (PostgreSQL) | 数据存储、用户认证、RLS 权限控制 |
| 后端逻辑 | Supabase Edge Functions (Deno) | CSV 解析、提成计算 |
| 托管 | GitHub Pages | 静态前端部署 |

**访问地址：** https://sue1807.github.io/DTPPro8/

---

## 交易员配置

| Trader ID | 姓名 | 市场 | 通道开通 | 币种 | 提成比例 | 留存保证金 |
|-----------|------|------|---------|------|---------|----------|
| HONG045 | 王博 | ASX · CHIXA | 2025-08-01 | AUD | 80% | $1,000 USD |
| PENGCDU | 马金斗 | ASX · CHIXA | 2025-06-02 | AUD | 80% | $1,000 USD |
| LULUSHI | 石路路 | HKE | 2024-01-08 | HKD | 100% | $500 USD |

所有交易员保证金余额统一以 **USD** 计算。

---

## 用户账号

| 账号 | 角色 |
|------|------|
| admin@trading.com | 管理员 |
| hong045@trading.com | 交易员（仅查看本人）|
| pengcdu@trading.com | 交易员（仅查看本人）|
| lulushi@trading.com | 交易员（仅查看本人）|

---

## 核心计算公式

### 一、提成计算

#### AUD 交易员（王博 HONG045 / 马金斗 PENGCDU）

```
NET           = Gross − Gateway Charge
Base          = NET × 80%

Entitlements  = (ASX_AUD ÷ 2) + (CHIXA_USD ÷ 2 ÷ fx_aud_usd)
              = 132.22 AUD + (64.50 USD ÷ fx_aud_usd) AUD
              （每人各承担 ASX 和 CHIXA 各一半，CHIXA 按当月汇率换算为 AUD）

可结算 NET    = Base − Exe Fee − Entitlements

USD 金额      = 可结算 NET × fx_aud_usd
Office Fee    = −75 USD（王博、马金斗各自承担一半 Office 费 $150）

每月余额(USD) = USD 金额 − 75
```

#### HKD 交易员（石路路 LULUSHI）

```
NET           = Gross − Gateway Charge
Base          = NET × 100%

Entitlements  = 451.70 HKD（HKE 数据费，石路路独自承担）

可结算 NET    = Base − Exe Fee − 451.70

每月余额(USD) = 可结算 NET × fx_hkd_usd
（无 Office Fee）
```

### 二、Settlement 核对公式

```
AUD Equity = Σ(Gross − Gateway Charge)            所有 AUD 交易员合计
HKD Equity = Gross − Gateway Charge − Sec Fee     LULUSHI（Sec Fee 不计入 Equity）

平台抽成：
  Platform Cut   = Equity × 15%
  Net for Equity = Equity × 85%
```

> 核对通过标准：计算值与 PDF 差值 < 0.05

### 三、结构性利润分析

#### AUD 交易员（结构性盈利）

```
交易室净利 = NET × 5% + Entitlements + $75 Office
```
→ 只要交易员盈利，交易室必然盈利

#### HKD 交易员（结构性风险）

```
交易室净利 = −NET × 15% + 451.70 HKD
```
→ 当 NET > 3,011 HKD 时交易室开始亏损（100% 费率结构）

---

## 费用配置（固定值）

| 费用项 | 总金额 | 币种 | 分摊方式 |
|--------|--------|------|---------|
| ASX 数据费 | 264.44 | AUD | 王博 + 马金斗 各 132.22 |
| CHIXA 数据费 | 129.00 | USD | 王博 + 马金斗 各 64.50（换算为 AUD）|
| HKE 数据费 | 451.70 | HKD | 石路路独自承担 |
| Office 费 | 150.00 | USD | 王博 −75，马金斗 −75；石路路 $0 |
| Wire 费 | 实际发生 | USD | 交易室承担，不扣交易员 |

---

## 保证金规则

| 交易员 | 留存保证金 | 预警线（50%）| 危险线 |
|--------|----------|------------|--------|
| 王博 / 马金斗 | $1,000 USD | < $500 USD | ≤ $0 |
| 石路路 | $500 USD | < $250 USD | ≤ $0 |

**余额构成：**
```
账户总余额 = 初始存入 + 累计提成确认 − 累计发放提取
可提取金额 = 账户总余额 − 留存保证金
```

---

## 提成发放逻辑

```
发放金额(USD) = 账户总余额 − 留存保证金
CNY 到账      = 发放金额 × USD/CNY 汇率（当天银行汇率）

发放后：
  margin_ledger 写入 withdraw 类型记录
  账户总余额 = 留存保证金（发放后归零到留存线）
```

---

## 月度操作流程

```
① 上传 CSV（交易业绩 Tab）
   → 从 DTTW Metro 导出 TRADER_TRADING.csv
   → AUD 文件（HONG045 + PENGCDU）和 HKD 文件（LULUSHI）分两次上传
   → 右上角先选好月份

② 录入 Settlement（Settlement Tab）
   → 上传 Settlement PDF 自动解析
   → 检查数据后保存
   → 汇率自动同步到结算参数

③ 确认结算参数（结算参数 Tab）
   → 确认 AUD/USD、HKD/USD 汇率已填入
   → 其他费用通常不变

④ 提成计算（交易业绩 Tab 下方）
   → ① 计算：生成草稿
   → ② 核对：对比 Settlement PDF（差值 < 0.05 通过）
   → ③ 确认锁定：写入 margin_ledger，余额更新

⑤ 提成发放（提成发放 Tab）
   → 填入当日 USD/CNY 汇率
   → 确认发放：写入 withdraw 记录，余额扣减
```

> **注意：** DTTW Metro 月度数据通常次月 26-28 日校对完成，建议月末操作。
> 月中可预先上传 CSV 测算，确认前可重复上传覆盖。

---

## 历史汇率记录

| 月份 | AUD/USD | HKD/USD |
|------|---------|---------|
| 2025-11 | 0.64670303 | 0.12848634 |
| 2025-12 | 0.64670303 | 0.12848634 |
| 2026-01 | 0.674448 | 0.12451659 |

---

## 数据库表结构

| 表名 | 说明 |
|------|------|
| `traders` | 交易员基本信息（id, name, ccy, rate, platfee_usd）|
| `period_fees` | 月度费用与汇率配置 |
| `trade_records` | 每月 CSV 汇总（唯一键：trader_id + period）|
| `commission_results` | 提成计算结果（draft / confirmed）|
| `margin_ledger` | 保证金流水（deposit / commission / deduct / withdraw）|
| `settlement_records` | Settlement PDF 录入数据（含 JSONB 动态字段）|
| `commission_payouts` | 提成发放记录（USD → CNY）|
| `user_profiles` | 用户角色（admin / trader）|
| `margin_balances` | 视图：当前余额 + 预警状态 |

---

## Edge Functions

| 函数 | 用途 |
|------|------|
| `smooth-service` | 提成计算：calculate / verify / confirm |
| `parse-csv` | CSV 解析入库（支持预览 + 确认两步）|

两个函数均已关闭 JWT 验证，通过 `apikey` header 鉴权。

---

## 前端部署

```powershell
cd D:\trading-v2\frontend
npm run build

# 上传到 GitHub:
# 1. 进入 github.com/sue1807/DTPPro8
# 2. 删除 assets/ 下旧 JS/CSS 文件（文件名含 hash，每次 build 都会变）
# 3. Upload files → 拖入 dist/ 所有内容 → Commit
# 4. 等约 1 分钟 GitHub Pages 重新部署
# 5. 浏览器 Ctrl+Shift+R 强制刷新
```

---

## Supabase 项目信息

- **Project URL:** `https://ieyljmelqcuqfxnchesu.supabase.co`
- **GitHub Repo:** `https://github.com/sue1807/DTPPro8`
- **Pages URL:** `https://sue1807.github.io/DTPPro8/`
