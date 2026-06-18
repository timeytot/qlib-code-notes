# Qlib Backtest Executor Generator and yield-from Control Flow

## 1. Core Idea

In Qlib's backtest execution flow, `yield` and `yield from` mainly serve two purposes:

```text
1. Expose trade_decision to the outer caller
   This is mainly used for RL training or data collection.

2. Receive the execution result res from the inner layer
   This allows executors and strategies at different levels to keep advancing the backtest.
```

In one sentence:

```text
yield:
The current generator pauses and sends a value to its caller.

yield from:
The current generator delegates execution to an inner generator.
It forwards the inner generator's yielded values outward,
and receives the inner generator's final return value.
```

---

## 2. Overall Call Chain

A normal backtest entry looks roughly like this:

```python
return_value = {}

for _decision in collect_data_loop(
    start_time,
    end_time,
    trade_strategy,
    trade_executor,
    return_value,
):
    pass

portfolio_dict = return_value.get("portfolio_dict")
indicator_dict = return_value.get("indicator_dict")
```

The execution chain can be understood as:

```text
backtest_loop
  `-- collect_data_loop
       `-- yield from trade_executor.collect_data(...)
            `-- BaseExecutor.collect_data
                 |-- if track_data: yield trade_decision
                 |-- obj = self._collect_data(...)
                 |-- if obj is generator: yield_res = yield from obj
                 |-- res, kwargs = yield_res / obj
                 |-- trade_account.update_bar_end(...)
                 |-- trade_calendar.step()
                 `-- return res
```

---

## 3. What `for _decision in collect_data_loop(...): pass` Does

This loop is not mainly used to consume `_decision`.

Its main purpose is to keep driving the generator so that the whole backtest process can run to completion.

```python
for _decision in collect_data_loop(...):
    pass
```

If an executor has `track_data=True`, it will internally run:

```python
yield trade_decision
```

This `trade_decision` will be forwarded by the outer `yield from` to:

```python
for _decision in collect_data_loop(...):
    pass
```

At that point, `_decision` receives the `trade_decision` yielded from inside the executor.

However, in the original `backtest_loop`, the loop body is only `pass`, so the yielded decision is received and then discarded.

This means:

```text
Normal backtesting does not use the yielded trade_decision.
yield trade_decision is mainly an interface reserved for RL training or data collection.
```

If we actually want to collect these decisions, we can write:

```python
return_value = {}
decisions = []

for trade_decision in collect_data_loop(
    start_time,
    end_time,
    trade_strategy,
    trade_executor,
    return_value,
):
    decisions.append(trade_decision)

portfolio_dict = return_value["portfolio_dict"]
indicator_dict = return_value["indicator_dict"]
```

---

## 4. `yield trade_decision` and `_execute_result` Are Not the Same Thing

Inside `collect_data_loop`, there is this line:

```python
_execute_result = yield from trade_executor.collect_data(
    _trade_decision,
    level=0,
)
```

This is easy to misunderstand.

During execution, `yield from` may indeed forward this value:

```python
yield trade_decision
```

But `_execute_result` does not receive this `trade_decision`.

Instead, `_execute_result` receives the final return value of `trade_executor.collect_data()`:

```python
return res
```

That is the execution result.

So:

```text
yield trade_decision
= An intermediate value exposed to the outside for observation or data collection.

return res
= The final value returned by the generator to yield from when the generator finishes.
```

The relationship is:

```text
for trade_decision in collect_data_loop(...)
    receives the value yielded by yield trade_decision.

_execute_result = yield from trade_executor.collect_data(...)
    receives the final res returned by collect_data().
```

---

## 5. How `return res` Inside a Generator Is Retrieved

Once a function contains `yield`, it becomes a generator function.

A generator can still contain:

```python
return res
```

However, this is not a normal function return value.

Instead, it becomes the final value of the generator when the generator stops.

A normal `for` loop cannot retrieve this `return res`.

But `yield from` can automatically receive it.

Conceptually:

```python
_execute_result = yield from trade_executor.collect_data(...)
```

is roughly equivalent to:

```python
gen = trade_executor.collect_data(...)

try:
    while True:
        value = next(gen)
        yield value
except StopIteration as e:
    _execute_result = e.value
```

Therefore, the source code does not need to manually write:

```python
except StopIteration as e:
```

because `yield from` already handles that internally.

---

## 6. What `yield_res = yield from obj` Means

Inside `BaseExecutor.collect_data()`, there is this logic:

```python
obj = self._collect_data(trade_decision=trade_decision, level=level)

if isinstance(obj, GeneratorType):
    yield_res = yield from obj
    assert isinstance(yield_res, tuple) and len(yield_res) == 2
    res, kwargs = yield_res
else:
    res, kwargs = obj
```

This is used to support two kinds of executors.

---

### 6.1 Innermost Executor: Directly Returns a Tuple

For example, `SimulatorExecutor._collect_data()` is an innermost executor.

It does not call another nested executor.

It directly executes orders:

```python
trade_val, trade_cost, trade_price = self.trade_exchange.deal_order(...)
```

Then it directly returns:

```python
return execute_result, {"trade_info": execute_result}
```

So here:

```python
obj = self._collect_data(...)
```

already gets:

```python
(
    execute_result,
    {"trade_info": execute_result}
)
```

Therefore, the code goes into:

```python
else:
    res, kwargs = obj
```

---

### 6.2 Nested Executor: Returns a Generator

`NestedExecutor._collect_data()` still needs to call an inner executor:

```python
_inner_execute_result = yield from self.inner_executor.collect_data(...)
```

Because it uses `yield from` internally, `_collect_data()` itself becomes a generator.

At this point:

```python
obj = self._collect_data(...)
```

does not get the final result.

It gets a generator object.

Therefore, the outer layer needs:

```python
yield_res = yield from obj
```

This means: run this `_collect_data()` generator to completion.

At the end, `NestedExecutor._collect_data()` returns:

```python
return execute_result, {
    "inner_order_indicators": inner_order_indicators,
    "decision_list": decision_list,
}
```

So:

```python
yield_res = yield from obj
```

eventually receives:

```python
(
    execute_result,
    {
        "inner_order_indicators": inner_order_indicators,
        "decision_list": decision_list,
    }
)
```

Then it is unpacked:

```python
res, kwargs = yield_res
```

which means:

```python
res = execute_result
kwargs = {
    "inner_order_indicators": inner_order_indicators,
    "decision_list": decision_list,
}
```

---

## 7. How `res` and `kwargs` Are Used Later

Regardless of whether `_collect_data()` returns directly or returns through a generator, the final output is always:

```python
res, kwargs
```

Then the code enters the unified account update logic:

```python
self.trade_account.update_bar_end(
    trade_start_time,
    trade_end_time,
    self.trade_exchange,
    atomic=atomic,
    outer_trade_decision=trade_decision,
    indicator_config=self.indicator_config,
    **kwargs,
)
```

That means:

```text
res
= The execution result of the current executor level.

kwargs
= Extra information passed to update_bar_end() for updating account state and indicators.
```

For the innermost executor, `kwargs` is:

```python
{"trade_info": execute_result}
```

because the innermost executor has the real order execution details.

For the nested executor, `kwargs` is:

```python
{
    "inner_order_indicators": inner_order_indicators,
    "decision_list": decision_list,
}
```

because the nested executor does not directly execute orders.

Instead, it aggregates indicators from the inner executor.

---

## 8. The Role of the `return_value` Dictionary

`collect_data_loop()` itself is a generator.

Its final backtest result is not returned through a normal:

```python
return portfolio_dict, indicator_dict
```

Instead, it writes the result back into the external `return_value` dictionary:

```python
return_value.update({
    "portfolio_dict": portfolio_dict,
    "indicator_dict": indicator_dict,
})
```

So the outer caller retrieves the result like this:

```python
return_value = {}

for _decision in collect_data_loop(..., return_value):
    pass

portfolio_dict = return_value["portfolio_dict"]
indicator_dict = return_value["indicator_dict"]
```

The reason is:

```text
A normal for loop can only consume values yielded by the generator.
A normal for loop cannot directly retrieve the generator's final return value.
Therefore, Qlib uses the return_value dictionary as a side channel
to store the final backtest metrics.
```

---

## 9. The Most Accurate Interpretation

```text
yield trade_decision
= Exposes the decision to the outer caller in the middle of execution.
= In normal backtesting, it is usually received by the for loop and discarded.
= In RL or data collection scenarios, it can be consumed by the external loop.

yield from inner_generator
= Pauses the current layer and delegates execution to the inner generator.
= Values yielded by the inner generator are forwarded outward.
= The inner generator's final returned res is received by yield from
  and used by the current layer.

return res
= The final execution result of the current generator.
= A normal for loop cannot receive it.
= yield from can receive it.
```

Essentially, Qlib uses this design to:

```text
Use generator / yield from to transfer control across multiple executor levels.
Use yield to expose intermediate trade_decision values.
Use return res to pass execution results back to the upper layer.
Use the return_value dictionary to store final metrics for normal backtesting.
```

---

## 10. One-Sentence Summary

```text
In Qlib backtesting, yield from acts as a middle proxy layer:
it forwards inner yielded trade_decision values outward,
while collecting inner returned execute_result values inward.

This allows normal backtesting, nested executors,
and RL data collection to share the same execution framework.
```
