# trade2at4at3 策略说明

> 对应代码：`trade2at4at3`  
> 基线：`trade2at4`（ATR 动态止损 + 10:00/14:30 双检）  
> 本版新增：**收紧 EMA 回踩触及窗口（10 日 → 5 日）**

---

## 1. 优化需求（原文）

调整 EMA 条件过宽：**10 日内任意一天触及 EMA10** → 改为 **5 日内**。

---

## 2. 相对 trade2at4 的改动

| 项目 | trade2at4 | trade2at4at3 |
|------|-----------|--------------|
| EMA 回踩触及窗口 | 近 10 日（`range(-10, -1)`） | **近 5 日（`range(-5, -1)`）** |
| 参数 | — | `g.ema_touch_lookback = 5` |
| 最小 K 线长度 | `len(close) >= 12` | **`len(close) >= lookback + 2`（即 7）** |

**未改动：** ATR 动态止损、14:30 止损复检、楔形/基底形态、通道 3 日、双死叉、仓位管理等 trade2at4 逻辑。

---

## 3. 逻辑说明

### 3.1 `_detect_ema_crossback` 变更

原逻辑（trade2at4）在 `_detect_ema_crossback` 中：

```python
for i in range(-10, -1):
    if close[i] <= ema10[i] * 1.01:
        touched_ema = True
```

即过去 10 个交易日（不含当日）内，**任意一天**收盘价触及 EMA10（±1% 容差）即视为「已回踩」，条件偏宽，容易匹配到较旧的回踩后再突破。

trade2at4at3 改为：

```python
lookback = g.ema_touch_lookback  # 默认 5
for i in range(-lookback, -1):
    if close[i] <= ema10[i] * 1.01:
        touched_ema = True
```

仅要求 **近 5 日内** 有触及 EMA10，强调「新鲜回踩」，过滤掉距回踩较远的滞后信号。

### 3.2 其余 EMA 回踩条件（不变）

| 条件 | 说明 |
|------|------|
| 当日收盘 | `close[-1] >= ema10[-1] * 0.995`（站回 EMA10 上方） |
| 均线排列 | `ema10[-1] > ema20[-1]` |
| EMA10 斜率 | `ema10[-1] > ema10[-3]` |

通过后仍返回形态 `ema_crossback`、基础得分 **85**。

---

## 4. 影响范围

```
get_canslim_pool → detect_cycle_stage
    → _detect_ema_crossback（仅此函数改动）
    → Stage2 候选池 ema_crossback 数量可能减少
    → 排序/买入逻辑不变
```

楔形突破（`wedge_pop`）、基底突破（`base_break`）检测**不受影响**。

---

## 5. 日志示例

```
📊 价格行为选股执行中 · trade2at4at3
📊 Stage2形态通过: 18 只
600519  贵州茅台  | 形态:ema_crossback  得分:85
===== 2022-03-15 trade2at4at3 交易 =====
🟢 买入 600519.XSHG，得分：85，形态：ema_crossback，价格...
```

---

## 6. 回测对比

| 基线 | `trade2at4` |
| 本版 | `trade2at4at3` |

重点观察：`ema_crossback` 候选数量变化、胜率与单笔期望是否因「新鲜回踩」而改善。
