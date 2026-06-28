# trade2 策略分析与优化方案

> 基于 `trade2` 源码 + `log.txt` 回测日志（2021-07 ~ 2023-06）  
> 目标：为下一版策略（如 `trade2a1`）提供可执行的改造清单

---

## 1. 策略概览

### 1.1 定位

融合 **趋势跟踪仓位管理** + **CANSLIM 基本面** + **价格行为 Stage2 形态** 的 A 股多头策略。

| 模块 | 实现位置 | 说明 |
|------|----------|------|
| 盘前选股 | `before_trading_start` (07:30) | CANSLIM → 技术面 → Stage2 形态 |
| 统一交易 | `market_open_trade` (10:00) | 先卖后买，每日仅一次 |
| 盘后计数 | `after_market_close` (15:00) | 通道跌破天数 + 净值统计 |

### 1.2 核心参数（当前值）

```python
g.max_holdings = 3          # 最多 3 只
g.position_size = 0.8       # 未直接使用，实际由 position_ratio 控制
g.hard_stop = 0.08          # 固定止损 8%
g.max_correction_from_high = 0.30
g.enable_weekly_filter = False   # 周线过滤关闭
g.min_price = 20.0
g.min_avg_money = 1e8         # 20 日均成交额 ≥ 1 亿
g.max_stock_output = 30
```

### 1.3 数据流

```
CANSLIM SQL (150只)
    ↓ ST/北交所/次新过滤
技术面 (200MA、52周高点、价格、成交额)
    ↓ detect_cycle_stage
Stage2 + 形态 (wedge_pop / ema_crossback / base_break)
    ↓ 按 score 排序，取 TOP30
g.today_stocks
    ↓ 10:00 大盘评分 → 仓位比例 → 买入
持仓监控 → 止损 / 通道止盈 / 双死叉 → 卖出
```

---

## 2. 模块详解

### 2.1 大盘评分与仓位

**函数：** `get_market_score` / `calculate_position_ratio`

| 评分区间 | 目标仓位 |
|----------|----------|
| ≥ 70 | 100% |
| ≥ 55 | 80% |
| ≥ 40 | 60% |
| ≥ 25 | 40% |
| < 25 | 20% |

**问题：**

1. **基准不一致**：`set_benchmark('000300.XSHG')` 用沪深300，但 `get_market_score` 用 **上证指数 000001**，熊市/牛市判断可能偏差。
2. **买入门槛过低**：仅 `market_score < 30` 才停买（约 20% 仓位档），2022 熊市仍大量开仓。
3. **`g.position_size = 0.8` 未使用**：实际仓位完全由 `position_ratio` 决定，参数冗余。

**单股目标市值：**

```python
per_stock_target = total_value * position_ratio / g.max_holdings
```

3 只均分，每只约占总资产 20%~33%。

---

### 2.2 选股：CANSLIM 基本面

**函数：** `get_canslim_pool`

| 条件 | 阈值 |
|------|------|
| 流通市值 | 20 ~ 500 亿 |
| 净利润 YoY | > 30% |
| 营收 YoY | > 30% |
| ROE | > 8% |
| EPS | > 0 |

**问题：**

- 高增速 = 高波动，与 8% 固定止损不匹配。
- 无行业分散约束，日志中出现锂电/医药/游戏等同板块反复交易。
- 排序仅按净利润增速，未结合形态得分。

---

### 2.3 选股：Stage2 形态检测

**函数：** `detect_cycle_stage` — 按优先级依次匹配，**命中即返回**：

| 优先级 | 形态 | setup 键 | score | 额外条件 |
|--------|------|----------|-------|----------|
| 1 | 楔形突破 | `wedge_pop` | 100 | 量 > 20日均量 × 1.5 |
| 2 | EMA 回踩 | `ema_crossback` | 85 | e10 > e20，e10 上升 |
| 3 | 基底突破 | `base_break` | 80 | 量 > 20日均量 × 1.3 |

**Stage2 前提：** 收盘价 ≥ MA200 × 0.95

#### EMA 回踩条件（`_detect_ema_crossback`）

```python
# 过去 10 日内任意一天 close <= ema10 * 1.01  → 算"触及"
# 昨日 close >= ema10 * 0.995                  → 算"站稳"
# ema10 > ema20 且 ema10 高于 3 日前
```

**问题（日志验证）：**

- 114 笔卖出 **100% 为 EMA 回踩**，楔形/基底几乎未产生实际持仓。
- EMA 条件过宽：「10 日内任意一天触及 EMA10」大量假信号。
- `enable_weekly_filter = False`，缺少周线趋势确认。

---

### 2.4 买入逻辑

**函数：** `market_open_trade` 买入段

**过滤链：**

1. 不在持仓中
2. 不在 `g.sold_today`（当日刚卖出）
3. 非停牌 / ST / 北交所
4. 非涨停（≥ 涨停价 × 99.8%）
5. 按 `g.today_stocks` 顺序取前 N 个空位

**问题：**

| 问题 | 说明 |
|------|------|
| 无买入冷静期 | 止损后数周可再次买入同股（万泰生物 6 次亏损） |
| 无行业互斥 | 卖出后仍可买同行业 |
| 同日换仓 | 日志 76 次「卖出日立即开新仓」 |
| 形态优先级仅在排序 | 买入时按 score 排序，但 EMA 占绝对多数 |
| 科创板限价 | 688 限价 ±2%，急跌日卖出可能无法成交 |

---

### 2.5 卖出逻辑（优先级顺序）

```
1. 固定止损  pnl <= -8%           → 立即卖
2. 通道止盈  连续 3 日收盘 < 趋势1  → 卖（盈/亏均可）
3. BBI+MACD 双死叉                → 卖
```

#### 通道指标（`calc_dktd_channel`）

加权均价 BBI 类指标 → EMA(13) → × 1.01 为上轨下沿（趋势1）。  
计数在 `after_market_close` 更新，**仅 10:00 检查一次**。

#### 双死叉（BBI 6/12/24/48 + MACD 5/10/15）

昨日收盘下穿 BBI **且** MACD 下穿 Signal → 卖出。

**问题：**

| 规则 | 日志表现 | 问题 |
|------|----------|------|
| 固定止损 -8% | 39 次，胜率 0%，均亏 -10.1% | 仅 10:00 检查；跳空滑点；15 笔实际亏损 >10% |
| 通道 3 日跌破 | 51 次，胜率 35%，**净贡献 +110%** | 唯一盈利来源；>30 天持仓胜率 80% |
| 双死叉 | 24 次，胜率 8%，净 -81% | 盈利单也被提前砍掉；与「让利润奔跑」冲突 |

---

## 3. 回测日志诊断（log.txt）

> 周期：2021-07-01 ~ 2023-06-01，114 笔完整卖出

### 3.1 总体指标

| 指标 | 数值 |
|------|------|
| 胜率 | **17.5%**（20 盈 / 94 亏） |
| 平均盈利 | +11.83% |
| 平均亏损 | -6.42% |
| 盈亏比 | 1.84 |
| 每笔期望 | **-3.29%** |

结构正确（赚大亏小），但胜率过低导致期望为负。

### 3.2 按卖出原因

| 原因 | 次数 | 胜率 | 净贡献 |
|------|------|------|--------|
| 固定止损 -8% | 39 | 0% | **-407%** |
| BBI+MACD 双死叉 | 24 | 8% | **-81%** |
| 通道 3 日跌破 | 51 | 35% | **+110%** |

### 3.3 按持仓天数

| 持仓 | 次数 | 胜率 | 平均收益 |
|------|------|------|----------|
| ≤ 5 天 | 28 | 4% | -6.6% |
| 6~15 天 | 40 | 5% | -6.4% |
| 16~30 天 | 32 | 16% | -4.6% |
| **> 30 天** | 15 | **80%** | **+14.1%** |

盈利均持 35 天，亏损均持 12 天 — **策略需要时间展开趋势**。

### 3.4 重复亏损股票（TOP）

| 股票 | 亏损次数 | 累计 |
|------|----------|------|
| 万泰生物 | 6 | -32.3% |
| 瑞丰新材 | 3 | -26.1% |
| 晨光新材 | 5 | -23.4% |
| 中矿资源 | 3 | -20.7% |

### 3.5 按年份

| 年份 | 交易数 | 胜率 |
|------|--------|------|
| 2021 | 31 | 23% |
| 2022 | 58 | **12%** |
| 2023 | 25 | 24% |

2022 熊市交易最多、胜率最低 → 大盘风控不足。

---

## 4. 核心问题总结

```
                    ┌─────────────────────────────────┐
                    │         策略期望为负             │
                    └─────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          ▼                           ▼                           ▼
   【入场过宽】                 【止损过紧】                 【过早卖出】
   EMA回踩占100%              8%固定+10:00单次              双死叉砍盈利
   无冷静期/行业过滤           跳空滑点至-10~20%            短持(<15天)胜率5%
          │                           │                           │
          └───────────────────────────┴───────────────────────────┘
                                      │
                              【通道止盈有效但未充分受益】
                              >30天持仓胜率80%，但被前两者抵消
```

---

## 5. 优化方案（分阶段）

### Phase A — 风控与重复交易（`trade2a1` 已列需求）

#### A1. 个股 20 天买入冷静期

**需求：** 买入后（或止损卖出后）20 个交易日内不再买入同一只股票；**在选股阶段过滤**。

**实现要点：**

```python
# initialize 新增
g.cooldown_days = 20
g.cooldown_until = {}   # {stock: date}

# 止损/任意卖出时记录
g.cooldown_until[stock] = context.current_dt.date() + timedelta(days=g.cooldown_days)

# get_canslim_pool 或 before_trading_start 末尾过滤
if stock in g.cooldown_until and context.current_dt.date() < g.cooldown_until[stock]:
    continue
```

**注意：** 用 **交易日** 而非自然日（可用聚宽 `get_trade_days`）。

#### A2. 同行业互斥

**需求：** 某行业股票卖出后，不再买入同行业。

**实现要点：**

```python
g.blocked_industries = set()   # 当日/冷却期内禁买行业

# 卖出时
industry = get_industry(stock)  # 或 get_security_info + 申万行业 API
g.blocked_industries.add(industry)
g.industry_cooldown[industry] = context.current_dt.date() + timedelta(days=20)

# 选股/买入时
if get_industry(stock) in g.blocked_industries:
    continue
```

聚宽可用 `get_industry(stocks, date)` 或 fundamentals 行业字段。

#### A3. 分级卖出：盈利后关闭双死叉

**需求：**

- 盈利 **> 15%**：跳过 BBI+MACD 双死叉，仅保留通道止盈（+ 止损）
- 盈利 **< 5%**：保留双死叉作为保护

**修改位置：** `market_open_trade` 双死叉块（约 L286-L321）

```python
pnl_ratio = (current_price - cost) / cost

# ... 止损、通道止盈 ...

if pnl_ratio > 0.15:
    continue   # 盈利>15%，不做双死叉

if pnl_ratio < 0.05:
    # 执行双死叉检查
    ...
# 5%~15% 区间：可选 — 仅 BBI 死叉或仅 MACD，建议回测对比
```

---

### Phase B — 入场质量

#### B1. 收紧 EMA 回踩

```python
def _detect_ema_crossback(...):
    # 1. 回踩日成交量萎缩
    if volume[-2] > v20 * 0.85:  # 回踩日量不能太大
        return False, 0
    # 2. 昨日阳线确认
    if close[-1] <= close[-2]:
        return False, 0
    # 3. 站稳更强
    if close[-1] < ema10[-1] * 1.005:
        return False, 0
```

#### B2. 启用周线过滤

```python
g.enable_weekly_filter = True
```

#### B3. 楔形优先于 EMA（排序 + 互斥）

当前 `detect_cycle_stage` 已按 wedge → ema → base 顺序返回，但 wedge 条件严、ema 过宽。

**建议：**

1. 先收紧 EMA（B1），自然提高 wedge 占比。
2. `before_trading_start` 排序后，买入时 **同分优先 wedge_pop**。
3. 可选：ema_crossback 单独降低 `max_holdings` 权重（如只占总仓位 50%）。

#### B4. 大盘买入门槛提高

```python
# 原：market_score < 30 停买
# 建议：
if market_score < 40:
    log.info("大盘评分过低，暂停买入")
elif market_score < 55:
    g.effective_max_holdings = 1   # 仅 1 只
else:
    g.effective_max_holdings = g.max_holdings
```

同时将 `get_market_score` 改为 **000300.XSHG** 与 benchmark 一致。

---

### Phase C — 止损优化

#### C1. 动态止损（ATR）

```python
atr = talib.ATR(high, low, close, 14)[-1]
stop_price = cost - 2.0 * atr
if current_price <= stop_price:
    # 止损
```

高波动股宽止损，低波动股窄止损。

#### C2. 增加盘中检查

```python
run_daily(market_open_trade, time='10:00')
run_daily(check_stop_loss, time='14:00')   # 新增：仅止损，不买入
```

减少隔夜跳空至 -10% 以上的情况。

#### C3. 止损放宽至 10% 或 EMA20 跌破

固定 8% 在 CANSLIM 高 beta 池中过紧；可与 ATR 取 max(8%, 2×ATR)。

---

### Phase D — 止盈优化

#### D1. 盈利分层通道规则

| 浮盈 | 通道跌破天数 | 双死叉 |
|------|--------------|--------|
| < 5% | 3 日 | 启用 |
| 5%~15% | 3 日 | 可选 |
| > 15% | **5 日** | 关闭 |
| > 30% | 跟踪止损（如最高回撤 10%） | 关闭 |

#### D2. 减少同日换仓

卖出后 **T+1** 才允许买入（不仅 `sold_today`，而是全局 `g.last_sell_date`）。

---

## 6. 建议参数对照表

| 参数 | trade2 现值 | 建议值 | 理由 |
|------|-------------|--------|------|
| `g.hard_stop` | 0.08 | 0.10 或 ATR×2 | 减少假杀 |
| `g.enable_weekly_filter` | False | **True** | 顺大势 |
| 买入最低 market_score | 30 | **40~55** | 2022 教训 |
| `get_market_score` 标的 | 000001 | **000300** | 与 benchmark 一致 |
| 个股冷静期 | 无 | **20 交易日** | 防重复踩坑 |
| 行业冷静期 | 无 | **20 交易日** | 防板块集中 |
| 盈利 >15% 双死叉 | 始终启用 | **关闭** | 保留 +110% 通道收益 |
| 盈利 <5% 双死叉 | 始终启用 | **保留** | 控制小亏 |
| EMA 回踩成交量 | 无 | 萎缩确认 | 提高胜率 |

---

## 7. `trade2a1` 改造清单（可直接对照改代码）

| # | 任务 | 改动函数 | 优先级 |
|---|------|----------|--------|
| 1 | 20 天个股冷静期（选股阶段过滤） | `initialize`, `get_canslim_pool` / `before_trading_start`, 卖出处写 cooldown | P0 |
| 2 | 行业卖出后禁买同行业 | 卖出处 + 买入过滤 | P0 |
| 3 | 盈利>15% 关闭双死叉；<5% 保留 | `market_open_trade` L286+ | P0 |
| 4 | 楔形优先（收紧 EMA + 排序） | `_detect_ema_crossback`, `before_trading_start` | P1 |
| 5 | 大盘评分改 000300 + 提高买入门槛 | `get_market_score`, `market_open_trade` | P1 |
| 6 | 启用周线过滤 | `g.enable_weekly_filter = True` | P1 |
| 7 | ATR 动态止损 + 14:00 复检 | 新增 `check_stop_loss` | P2 |
| 8 | 盈利>15% 通道改为 5 日 | `market_open_trade` + `g.break_upper_count` 阈值 | P2 |
| 9 | 卖出日志统一格式（含行业、持仓天数） | 卖出各分支 | P3 |

---

## 8. 验收标准（回测对比）

改造前后用相同区间 **2021-07-01 ~ 2023-06-01** 对比：

| 指标 | trade2 基线 | 目标 |
|------|-------------|------|
| 胜率 | 17.5% | ≥ 25% |
| 每笔期望 | -3.29% | ≥ 0% |
| 固定止损次数 | 39 | ↓ 30% |
| 固定止损净贡献 | -407% | 明显改善 |
| >30 天持仓占比 | 13% | ↑ 25% |
| 同股重复亏损 | 万泰 6 次 | ≤ 2 次 |

---

## 9. 代码结构速查

```
trade2 (743 行)
├── initialize              L23-64    参数初始化
├── before_trading_start    L130-183  盘前选股
├── market_open_trade       L226-436  核心交易（卖+买）
├── _safe_order_target_value L438-457 科创板限价
├── after_market_close      L459-520  通道计数+净值
├── detect_cycle_stage      L523-571  Stage2+形态
├── _detect_wedge_pop       L585-603
├── _detect_ema_crossback   L605-620  ← 放宽，需收紧
├── _detect_base_break      L622-635
├── get_canslim_pool        L637-740  CANSLIM+技术面
├── get_market_score        L82-113
├── calculate_position_ratio L115-128
└── calc_dktd_channel       L185-224
```

---

## 10. 一句话结论

**trade2 的盈利引擎是「通道 3 日跌破止盈 + 长期持有」，失血点是「8% 固定止损 + BBI/MACD 双死叉 + EMA 回踩过宽」。**

优化主线：

1. **少亏** — 动态止损、提高入场质量、冷静期  
2. **留住利润** — 盈利后关双死叉、延长通道天数  
3. **少交易** — 大盘门槛、行业分散、避免同日换仓  

按 Phase A → B → C 顺序在 `trade2a1` 中逐步实现，每阶段单独回测验证后再合并。
