# Qlib Calendar, Time, and Index Alignment

## Core Idea

Qlib backtest does not use only one calendar.

It has several related but different time/index systems:

```text
1. Global data calendar: Cal
2. Current executor calendar: TradeCalendarManager
3. Strategy calendar: obtained from executor LevelInfrastructure
4. BaseTradeDecision start_time / end_time: snapshot of current strategy step
5. Order start_time / end_time: time window used by Exchange to query prices
6. TradeRangeByTime: optional intraday execution range
7. Exchange / Quote time window: market data query range
```

The most important rule:

```text
Executor controls step movement.
Strategy uses executor's calendar through LevelInfrastructure.
Decision records the current strategy step time.
Order uses step time to query Exchange.
Exchange uses order time to fetch quote data.
```

---

## 1. Backtest Entry

File:

```text
qlib/backtest/__init__.py
```

Main entry:

```python
def backtest(
    start_time,
    end_time,
    strategy,
    executor,
    benchmark="SH000300",
    account=1e9,
    exchange_kwargs={},
    pos_type="Position",
):
    trade_strategy, trade_executor = get_strategy_executor(
        start_time,
        end_time,
        strategy,
        executor,
        benchmark,
        account,
        exchange_kwargs,
        pos_type=pos_type,
    )
    return backtest_loop(start_time, end_time, trade_strategy, trade_executor)
```

Meaning:

```text
backtest(start_time, end_time)
    ↓
get_strategy_executor(...)
    ↓
create Account
create Exchange
create CommonInfrastructure
create Strategy
create Executor
    ↓
backtest_loop(...)
```

Important distinction:

```text
start_time / end_time are used in two different places:

1. Executor calendar
   → controls when the executor steps

2. Exchange quote window
   → controls what market data can be queried
```

---

## 2. Exchange Time Window

File:

```text
qlib/backtest/__init__.py
```

Code:

```python
def get_exchange(
    exchange=None,
    freq="day",
    start_time=None,
    end_time=None,
    codes="all",
    subscribe_fields=[],
    open_cost=0.0015,
    close_cost=0.0025,
    min_cost=5.0,
    limit_threshold=None,
    deal_price=None,
    **kwargs,
):
    if exchange is None:
        exchange = Exchange(
            freq=freq,
            start_time=start_time,
            end_time=end_time,
            codes=codes,
            deal_price=deal_price,
            subscribe_fields=subscribe_fields,
            limit_threshold=limit_threshold,
            open_cost=open_cost,
            close_cost=close_cost,
            min_cost=min_cost,
            **kwargs,
        )
        return exchange
```

Meaning:

```text
get_exchange(...)
    ↓
Exchange(freq, start_time, end_time, codes, deal_price)
    ↓
Exchange quote data window
```

This is not the executor calendar.

It only defines what quote data can be queried later.

---

## 3. Exchange Loads Quote Data

File:

```text
qlib/backtest/exchange.py
```

Code:

```python
class Exchange:
    def __init__(
        self,
        freq="day",
        start_time=None,
        end_time=None,
        codes="all",
        deal_price=None,
        subscribe_fields=[],
        ...
    ):
        self.freq = freq
        self.start_time = start_time
        self.end_time = end_time
        ...
        self.get_quote_from_qlib()
```

Then:

```python
def get_quote_from_qlib(self):
    self.quote_df = D.features(
        self.codes,
        self.all_fields,
        self.start_time,
        self.end_time,
        freq=self.freq,
        disk_cache=True,
    )
```

Meaning:

```text
Exchange.start_time / end_time
    ↓
D.features(...)
    ↓
Exchange.quote_df
    ↓
Exchange.quote
```

Exchange is responsible for:

```text
close price
deal price
volume
factor
limit up/down
suspension check
tradability check
```

---

## 4. Executor Creates Its Trade Calendar

File:

```text
qlib/backtest/executor.py
```

Code:

```python
def reset(self, common_infra=None, **kwargs):
    if "start_time" in kwargs or "end_time" in kwargs:
        start_time = kwargs.get("start_time")
        end_time = kwargs.get("end_time")

        self.level_infra.reset_cal(
            freq=self.time_per_step,
            start_time=start_time,
            end_time=end_time,
        )

    if common_infra is not None:
        self.reset_common_infra(common_infra)
```

Meaning:

```text
executor.reset(start_time, end_time)
    ↓
executor.level_infra.reset_cal(...)
    ↓
TradeCalendarManager is created or reset
    ↓
executor.trade_calendar becomes available
```

Important detail:

```python
if "start_time" in kwargs or "end_time" in kwargs:
```

This checks whether the key exists in `kwargs`.

It does not check whether the value is valid.

So:

```python
executor.reset(start_time=None, end_time=None)
```

still enters `reset_cal(...)`.

But:

```python
executor.reset()
```

does not enter `reset_cal(...)`.

Therefore:

```text
If executor.reset() is called without start_time/end_time:
    - it will not create a new trade calendar
    - it will not reset the existing calendar
    - if no calendar existed before, accessing executor.trade_calendar may fail
```

---

## 5. Executor Trade Calendar Property

File:

```text
qlib/backtest/executor.py
```

Code:

```python
@property
def trade_calendar(self):
    return self.level_infra.get("trade_calendar")
```

Meaning:

```text
executor.trade_calendar
= executor.level_infra.get("trade_calendar")
```

Executor does not store calendar as a direct normal attribute.

It reads it from `LevelInfrastructure`.

---

## 6. Backtest Loop Passes Executor Infrastructure to Strategy

File:

```text
qlib/backtest/backtest.py
```

Code:

```python
def collect_data_loop(
    start_time,
    end_time,
    trade_strategy,
    trade_executor,
    return_value=None,
):
    trade_executor.reset(
        start_time=start_time,
        end_time=end_time,
    )

    trade_strategy.reset(
        level_infra=trade_executor.get_level_infra()
    )

    while not trade_executor.finished():
        _trade_decision = trade_strategy.generate_trade_decision(
            _execute_result
        )

        _execute_result = yield from trade_executor.collect_data(
            _trade_decision,
            level=0,
        )
```

Meaning:

```text
collect_data_loop()
    ↓
trade_executor.reset(start_time, end_time)
    ↓
executor creates trade_calendar
    ↓
trade_strategy.reset(level_infra=trade_executor.get_level_infra())
    ↓
strategy receives executor's LevelInfrastructure
    ↓
strategy.generate_trade_decision(...)
```

This is the key source-level chain.

---

## 7. Strategy Does Not Own a Separate Calendar

File:

```text
qlib/strategy/base.py
```

Code:

```python
def reset(
    self,
    level_infra=None,
    common_infra=None,
    outer_trade_decision=None,
    **kwargs,
):
    self._reset(
        level_infra=level_infra,
        common_infra=common_infra,
        outer_trade_decision=outer_trade_decision,
    )
```

Then:

```python
def _reset(
    self,
    level_infra=None,
    common_infra=None,
    outer_trade_decision=None,
):
    if level_infra is not None:
        self.reset_level_infra(level_infra)
```

Then:

```python
def reset_level_infra(self, level_infra):
    if not hasattr(self, "level_infra"):
        self.level_infra = level_infra
    else:
        self.level_infra.update(level_infra)
```

Strategy calendar property:

```python
@property
def trade_calendar(self):
    return self.level_infra.get("trade_calendar")
```

Meaning:

```text
trade_strategy.reset(level_infra=trade_executor.get_level_infra())
    ↓
strategy.level_infra = executor.level_infra
    ↓
strategy.trade_calendar
    = strategy.level_infra.get("trade_calendar")
    = executor.level_infra.get("trade_calendar")
    = executor.trade_calendar
```

Strictly speaking:

```text
strategy.trade_calendar = executor.trade_calendar
```

is only a simplified explanation.

The source-level truth is:

```text
strategy.level_infra = executor.level_infra
strategy.trade_calendar is a property reading trade_calendar from level_infra
```

---

## 8. Strategy Generates Trade Decision

File:

```text
qlib/backtest/backtest.py
```

Code:

```python
while not trade_executor.finished():
    _trade_decision = trade_strategy.generate_trade_decision(
        _execute_result
    )

    _execute_result = yield from trade_executor.collect_data(
        _trade_decision,
        level=0,
    )
```

Meaning:

```text
executor current step
    ↓
strategy uses executor calendar through level_infra
    ↓
strategy.generate_trade_decision(...)
    ↓
BaseTradeDecision / TradeDecisionWO
```

---

## 9. BaseTradeDecision Records Current Step Time

File:

```text
qlib/backtest/decision.py
```

Code:

```python
class BaseTradeDecision:
    def __init__(self, strategy, trade_range=None):
        self.strategy = strategy
        self.start_time, self.end_time = strategy.trade_calendar.get_step_time()
        self.total_step = None

        if isinstance(trade_range, tuple):
            trade_range = IdxTradeRange(*trade_range)

        self.trade_range = trade_range
```

Meaning:

```text
strategy.trade_calendar.get_step_time()
    ↓
decision.start_time
decision.end_time
```

`BaseTradeDecision` records the time range of the strategy's current step.

Example:

```text
Daily strategy step:
2026-03-31

decision.start_time = 2026-03-31 00:00:00
decision.end_time   = 2026-03-31 23:59:59...
```

This is a snapshot of the current strategy step.

---

## 10. TradeDecisionWO Fills Order Time

File:

```text
qlib/backtest/decision.py
```

Code:

```python
class TradeDecisionWO(BaseTradeDecision):
    def __init__(self, order_list, strategy, trade_range=None):
        super().__init__(strategy, trade_range=trade_range)

        self.order_list = order_list

        start, end = strategy.trade_calendar.get_step_time()

        for o in order_list:
            if o.start_time is None:
                o.start_time = start
            if o.end_time is None:
                o.end_time = end
```

Meaning:

```text
strategy.trade_calendar.get_step_time()
    ├─ decision.start_time / decision.end_time
    └─ order.start_time / order.end_time
```

Important detail:

```text
Order start_time/end_time are not copied from decision.start_time/end_time directly.
TradeDecisionWO calls strategy.trade_calendar.get_step_time() again.
```

Normally they are the same because they are generated in the same strategy step.

---

## 11. BaseExecutor Executes the Decision

File:

```text
qlib/backtest/executor.py
```

Code:

```python
def collect_data(self, trade_decision, return_value=None, level=0):
    atomic = not issubclass(self.__class__, NestedExecutor)

    obj = self._collect_data(
        trade_decision=trade_decision,
        level=level,
    )

    ...

    trade_start_time, trade_end_time = self.trade_calendar.get_step_time()

    self.trade_account.update_bar_end(
        trade_start_time,
        trade_end_time,
        self.trade_exchange,
        atomic=atomic,
        outer_trade_decision=trade_decision,
        indicator_config=self.indicator_config,
        **kwargs,
    )

    self.trade_calendar.step()
```

Meaning:

```text
executor.collect_data(decision)
    ↓
executor._collect_data(decision)
    ↓
execute orders or run nested executor
    ↓
get executor current step time
    ↓
Account.update_bar_end(...)
    ↓
executor.trade_calendar.step()
```

Executor step time is used again at bar end to update account value and indicators.

---

## 12. SimulatorExecutor Sends Orders to Exchange

File:

```text
qlib/backtest/executor.py
```

Code:

```python
class SimulatorExecutor(BaseExecutor):
    def _collect_data(self, trade_decision, level=0):
        execute_result = []

        for order in self._get_order_iterator(trade_decision):
            trade_val, trade_cost, trade_price = self.trade_exchange.deal_order(
                order,
                trade_account=self.trade_account,
                dealt_order_amount=self.dealt_order_amount,
            )

            execute_result.append(
                (order, trade_val, trade_cost, trade_price)
            )

        return execute_result, {"trade_info": execute_result}
```

Meaning:

```text
TradeDecisionWO
    ↓
order_list
    ↓
SimulatorExecutor
    ↓
Exchange.deal_order(order)
```

For normal daily TopK backtest, this is the main execution path.

---

## 13. Exchange Uses Order Time to Query Quote

File:

```text
qlib/backtest/exchange.py
```

Code:

```python
def deal_order(self, order, trade_account=None, position=None, dealt_order_amount=...):
    if not self.check_order(order):
        order.deal_amount = 0.0
        return 0.0, 0.0, np.nan

    trade_price, trade_val, trade_cost = self._calc_trade_info_by_order(
        order,
        trade_account.current_position if trade_account else position,
        dealt_order_amount,
    )

    if trade_val > 1e-5:
        if trade_account:
            trade_account.update_order(
                order=order,
                trade_val=trade_val,
                cost=trade_cost,
                trade_price=trade_price,
            )

    return trade_val, trade_cost, trade_price
```

Then deal price:

```python
def get_deal_price(
    self,
    stock_id,
    start_time,
    end_time,
    direction,
    method="ts_data_last",
):
    if direction == OrderDir.SELL:
        pstr = self.sell_price
    elif direction == OrderDir.BUY:
        pstr = self.buy_price

    deal_price = self.quote.get_data(
        stock_id,
        start_time,
        end_time,
        field=pstr,
        method=method,
    )

    return deal_price
```

Meaning:

```text
order.stock_id
order.start_time
order.end_time
order.direction
    ↓
Exchange.get_deal_price(...)
    ↓
Quote.get_data(...)
```

At this stage, order time becomes quote query time.

---

## 14. Quote Actually Slices Market Data

File:

```text
qlib/backtest/high_performance_ds.py
```

Code:

```python
def get_data(self, stock_id, start_time, end_time, field, method=None):
    if stock_id not in self.get_all_stock():
        return None

    if is_single_value(start_time, end_time, self.freq, self.region):
        try:
            return self.data[stock_id].loc[start_time, field]
        except KeyError:
            return None
    else:
        data = self.data[stock_id].loc[start_time:end_time, field]

        if data.empty:
            return None

        if method is not None:
            data = self._agg_data(data, method)

        return data
```

Meaning:

```text
Quote data
    ↓
stock_id
start_time
end_time
field
method
    ↓
self.data[stock_id].loc[start_time:end_time, field]
```

Common methods:

```text
ts_data_last  → last non-NaN value
sum           → sum over time range
all           → all values are True
None          → return time series / IndexData
```

---

## 15. Account Updates at Bar End

File:

```text
qlib/backtest/account.py
```

Code:

```python
def update_bar_end(
    self,
    trade_start_time,
    trade_end_time,
    trade_exchange,
    atomic,
    outer_trade_decision,
    trade_info=[],
    inner_order_indicators=[],
    decision_list=[],
    indicator_config={},
):
    self.update_current_position(
        trade_start_time,
        trade_end_time,
        trade_exchange,
    )

    if self.is_port_metr_enabled():
        self.update_portfolio_metrics(
            trade_start_time,
            trade_end_time,
        )
        self.update_hist_positions(trade_start_time)

    self.update_indicator(...)
```

Meaning:

```text
executor current step time
    ↓
Account.update_bar_end(...)
    ↓
update current position price
update account value
update portfolio metrics
update historical positions
update trade indicators
```

---

## 16. Normal Daily Backtest Full Flow

```text
qlib/backtest/__init__.py

backtest(start_time, end_time, strategy, executor, ...)
    ↓
get_strategy_executor(...)
    ↓
create Account
create Exchange
create CommonInfrastructure
create Strategy
create Executor
    ↓
backtest_loop(...)

qlib/backtest/backtest.py

collect_data_loop(...)
    ↓
trade_executor.reset(start_time, end_time)
    ↓
executor.level_infra.reset_cal(...)
    ↓
executor.trade_calendar is available
    ↓
trade_strategy.reset(level_infra=trade_executor.get_level_infra())
    ↓
strategy.level_infra = executor.level_infra
    ↓
strategy.trade_calendar reads executor calendar from level_infra
    ↓
while not executor.finished():
    ↓
    strategy.generate_trade_decision(...)
        ↓
        BaseTradeDecision records current step time
        ↓
        TradeDecisionWO fills order start/end
    ↓
    executor.collect_data(decision)
        ↓
        SimulatorExecutor._collect_data(decision)
            ↓
            Exchange.deal_order(order)
                ↓
                Exchange checks tradability
                Exchange gets deal price
                Quote slices market data
                Account updates order result
        ↓
        Account.update_bar_end(...)
        ↓
        executor.trade_calendar.step()
```

---

## 17. NestedExecutor: Outer Step Becomes Inner Calendar

File:

```text
qlib/backtest/executor.py
```

Code:

```python
def _init_sub_trading(self, trade_decision):
    trade_start_time, trade_end_time = self.trade_calendar.get_step_time()

    self.inner_executor.reset(
        start_time=trade_start_time,
        end_time=trade_end_time,
    )

    sub_level_infra = self.inner_executor.get_level_infra()
    self.level_infra.set_sub_level_infra(sub_level_infra)

    self.inner_strategy.reset(
        level_infra=sub_level_infra,
        outer_trade_decision=trade_decision,
    )
```

Meaning:

```text
outer executor current step
    ↓
outer trade_start_time / trade_end_time
    ↓
inner_executor.reset(start_time=outer_step_start, end_time=outer_step_end)
    ↓
inner executor calendar
    ↓
inner strategy receives inner executor level_infra
```

Example:

```text
Outer executor: day
Inner executor: 1min

Outer current step:
2026-03-31

Inner executor reset range:
2026-03-31 09:30 ~ 2026-03-31 15:00

Result:
inner calendar contains minute bars of that day
```

---

## 18. TradeRangeByTime Converts Clock Time to Inner Calendar Index

File:

```text
qlib/backtest/decision.py
```

Code:

```python
class TradeRangeByTime(TradeRange):
    def __call__(self, trade_calendar):
        start_date = trade_calendar.start_time.date()

        val_start = concat_date_time(
            start_date,
            self.start_time,
        )

        val_end = concat_date_time(
            start_date,
            self.end_time,
        )

        return trade_calendar.get_range_idx(
            val_start,
            val_end,
        )
```

Meaning:

```text
TradeRangeByTime("09:35", "14:50")
    ↓
use inner_calendar.start_time.date()
    ↓
2026-03-31 09:35 ~ 2026-03-31 14:50
    ↓
inner_calendar.get_range_idx(...)
    ↓
inner start_idx / end_idx
```

This converts human clock time into executor step index.

---

## 19. BaseTradeDecision.update Sets total_step

File:

```text
qlib/backtest/decision.py
```

Code:

```python
def update(self, trade_calendar):
    self.total_step = trade_calendar.get_trade_len()
    return self.strategy.update_trade_decision(
        self,
        trade_calendar,
    )
```

Meaning:

```text
outer decision
    ↓
update(inner_calendar)
    ↓
decision.total_step = inner_calendar.get_trade_len()
```

This is used later to clip invalid range indexes.

---

## 20. BaseTradeDecision.get_range_limit Clips Index Range

File:

```text
qlib/backtest/decision.py
```

Code:

```python
def get_range_limit(self, **kwargs):
    try:
        _start_idx, _end_idx = self._get_range_limit(**kwargs)
    except NotImplementedError:
        if "default_value" in kwargs:
            return kwargs["default_value"]
        else:
            raise

    if getattr(self, "total_step", None) is not None:
        if _start_idx < 0 or _end_idx >= self.total_step:
            _start_idx = max(0, _start_idx)
            _end_idx = min(self.total_step - 1, _end_idx)

    return _start_idx, _end_idx
```

Meaning:

```text
trade_range calculates raw start_idx / end_idx
    ↓
if total_step is available
    ↓
clip range into [0, total_step - 1]
```

This prevents invalid inner calendar index.

---

## 21. NestedExecutor Uses Range Limit

File:

```text
qlib/backtest/executor.py
```

Code:

```python
while not self.inner_executor.finished():
    trade_decision = self._update_trade_decision(trade_decision)

    sub_cal = self.inner_executor.trade_calendar

    start_idx, end_idx = get_start_end_idx(
        sub_cal,
        trade_decision,
    )

    if not self._align_range_limit or start_idx <= sub_cal.get_trade_step() <= end_idx:
        res = self.inner_strategy.generate_trade_decision(
            _inner_execute_result
        )

        _inner_trade_decision = res

        trade_decision.mod_inner_decision(
            _inner_trade_decision
        )

        _inner_execute_result = yield from self.inner_executor.collect_data(
            trade_decision=_inner_trade_decision,
            level=level + 1,
        )
    else:
        sub_cal.step()
```

Meaning:

```text
for each inner executor step:
    ↓
update outer decision with inner calendar
    ↓
convert outer decision trade_range to inner index range
    ↓
if current inner step is inside range:
        run inner strategy
        run inner executor
    else:
        skip this step
```

---

## 22. Data Calendar Range Is Different From Executor Step Range

File:

```text
qlib/backtest/decision.py
```

Code:

```python
def get_data_cal_range_limit(self, rtype="full", raise_error=False):
    day_start = pd.Timestamp(self.start_time.date())
    day_end = epsilon_change(day_start + pd.Timedelta(days=1))

    freq = self.strategy.trade_exchange.freq

    _, _, day_start_idx, day_end_idx = Cal.locate_index(
        day_start,
        day_end,
        freq=freq,
    )

    if self.trade_range is None:
        if raise_error:
            raise NotImplementedError
        else:
            return 0, day_end_idx - day_start_idx

    else:
        if rtype == "full":
            val_start, val_end = self.trade_range.clip_time_range(
                day_start,
                day_end,
            )
        elif rtype == "step":
            val_start, val_end = self.trade_range.clip_time_range(
                self.start_time,
                self.end_time,
            )

        _, _, start_idx, end_index = Cal.locate_index(
            val_start,
            val_end,
            freq=freq,
        )

        return start_idx - day_start_idx, end_index - day_start_idx
```

Meaning:

```text
decision time range
    ↓
trade_range.clip_time_range(...)
    ↓
Cal.locate_index(...)
    ↓
global data calendar index
    ↓
convert to intraday relative data index
```

This is not the same as executor step index.

Important distinction:

```text
get_range_limit(...)
    → executor calendar index

get_data_cal_range_limit(...)
    → global data calendar Cal index converted to daily relative index
```

---

## 23. Signal Alignment

File:

```text
qlib/backtest/signal.py
```

Code:

```python
class SignalWCache(Signal):
    def get_signal(self, start_time, end_time):
        signal = resam_ts_data(
            self.signal_cache,
            start_time=start_time,
            end_time=end_time,
            method="last",
        )
        return signal
```

Meaning:

```text
strategy current step time
    ↓
signal.get_signal(start_time, end_time)
    ↓
resample signal_cache
    ↓
use the last available signal inside this step
```

Example:

```text
strategy step:
2026-03-31 00:00 ~ 2026-03-31 23:59

signal_cache:
2026-03-30
2026-03-31

result:
use 2026-03-31 signal
```

---

## 24. Practical Mental Model

Use this simplified model:

```text
TradeCalendarManager
    = current executor's step coordinate system

BaseTradeDecision
    = snapshot of current strategy step time

TradeDecisionWO / Order
    = concrete order list with executable time windows

TradeRangeByTime
    = optional intraday execution restriction

Cal
    = global data calendar

Exchange
    = quote and execution simulation layer

Quote
    = actual time slicing over market data
```

---

## 25. Most Common Confusions

### Confusion 1

```text
strategy.trade_calendar is a separate calendar
```

Correct:

```text
strategy.trade_calendar is read from strategy.level_infra.
strategy.level_infra usually comes from executor.level_infra.
So strategy and executor share the same current-level calendar.
```

---

### Confusion 2

```text
Exchange calendar and executor calendar are the same thing
```

Correct:

```text
Executor calendar controls step movement.
Exchange quote window controls market data availability.
They often use the same start_time/end_time, but they are different concepts.
```

---

### Confusion 3

```text
Order start/end are always copied from decision start/end
```

Correct:

```text
TradeDecisionWO fills order start/end by calling strategy.trade_calendar.get_step_time().
Normally this equals decision.start_time/end_time, but it is not a direct copy.
```

---

### Confusion 4

```text
TradeRangeByTime directly executes orders
```

Correct:

```text
TradeRangeByTime only converts clock time to executor calendar index.
NestedExecutor decides whether current inner step is inside that range.
```

---

### Confusion 5

```text
get_data_cal_range_limit() returns executor step index
```

Correct:

```text
get_data_cal_range_limit() uses Cal.locate_index().
It returns global data calendar index converted to intraday relative index.
It is different from executor calendar step index.
```

---

## 26. For Daily Qlib + miniQMT Live Trading

For daily live trading, focus first on these three alignments:

```text
1. Qlib trade date
   → executor.trade_calendar current step

2. Strategy current step
   → signal.get_signal(start_time, end_time)

3. Order start/end
   → Exchange / quote price query
   → later mapped to miniQMT order generation
```

Usually not the first priority for daily live trading:

```text
NestedExecutor
TradeRangeByTime
get_data_cal_range_limit()
minute-level PA / FFR
intraday execution splitting
```

Daily live trading main path:

```text
miniQMT real account
    ↓
convert cash / positions into Qlib Account / Position
    ↓
Qlib strategy reads current step signal
    ↓
generate target position / dry-run orders
    ↓
compare with miniQMT current position
    ↓
generate real miniQMT orders
    ↓
send orders through miniQMT
```

---

## 27. One-Sentence Summary

```text
Qlib time alignment works by passing backtest start/end into the executor calendar, giving that executor infrastructure to the strategy, letting the strategy generate decisions based on the current executor step, filling orders with that step time, and finally using order time to query Exchange quote data for execution and account updates.
```
