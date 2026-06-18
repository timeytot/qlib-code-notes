# Qlib Backtest Calendar Helpers: get_data_cal_range and get_range_idx

[https://github.com/microsoft/qlib/blob/main/qlib/backtest/utils.py#L133](https://github.com/microsoft/qlib/blob/main/qlib/backtest/utils.py#L133)

## Core Idea

Both functions convert time information into integer indexes, but they use different coordinate systems.

`get_range_idx()` returns indexes relative to the current `TradeCalendarManager`.

`get_data_cal_range()` returns indexes relative to the beginning of the current day in the data calendar.

The most important difference:

**Their zero point is different.**

---

## 1. `get_range_idx(start_time, end_time)`

### What it answers

Where is this time range inside the current executor's own trading calendar?

### Zero point

`0 = self.start_time` of the current `TradeCalendarManager`.

### Example

Assume the current executor runs from `10:00` to `15:00`.

Then its internal calendar is:

* `10:00 = step 0`
* `11:00 = step 60`
* `12:00 = step 120`
* `13:00 = step 180`
* `15:00 = step 300`

So:

`get_range_idx("12:00", "13:00")`

returns approximately:

`(120, 180)`

Because `12:00 ~ 13:00` is 120 to 180 steps after the executor's own start time.

### Source-level behavior

The function first searches positions in the global calendar, then subtracts `self.start_index`.

That means the final result is relative to the current `TradeCalendarManager`, not relative to the whole day.

### Main usage

`get_range_idx()` is mainly used for execution range control, especially in nested executors.

For example, if an outer decision says the inner executor should only trade from `12:00` to `13:00`, Qlib needs to convert that time range into the inner executor's step indexes.

---

## 2. `get_data_cal_range(rtype="full" | "step")`

### What it answers

Where is this trading range inside the current day's data calendar?

### Zero point

`0 = day_start`

For crypto minute-level data, this usually means:

* `00:00 = index 0`
* `01:00 = index 60`
* `10:00 = index 600`
* `12:00 = index 720`
* `13:00 = index 780`

### Example

Assume the current step is:

`12:00 ~ 13:00`

Then:

`get_data_cal_range("step")`

returns approximately:

`(720, 780)`

Because `12:00 ~ 13:00` is located at the 720th to 780th minute of the day.

This is true even if the current executor itself starts from `10:00`.

### Source-level behavior

The function first computes the beginning of the current day:

`day_start = pd.Timestamp(self.start_time.date())`

Then it locates the index of `day_start` in the data calendar.

Finally, it subtracts `day_start_idx`.

That means the final result is relative to the start of the day, not relative to the executor's own start time.

### Main usage

`get_data_cal_range()` is more data-calendar oriented.

It is useful when Qlib needs to know the intraday data position of a trading range, such as:

* current range is the 120th to 180th minute of the trading day
* current range is the 720th to 780th minute of a 24-hour crypto day

---

## 3. Side-by-side Example

Assume:

* Data calendar: `00:00 ~ 24:00`, 1-minute frequency
* Current executor: `10:00 ~ 15:00`
* Target time range: `12:00 ~ 13:00`

| Function                          | Zero Point              | Returned Meaning                           | Approx Result |
| --------------------------------- | ----------------------- | ------------------------------------------ | ------------- |
| `get_range_idx("12:00", "13:00")` | Executor start: `10:00` | Step index inside the current executor     | `(120, 180)`  |
| `get_data_cal_range("step")`      | Day start: `00:00`      | Intraday data index inside the current day | `(720, 780)`  |

Same time range, different coordinate systems.

---

## 4. Practical Rule

`get_range_idx()` means:

Where is this time range inside my current executor?

`get_data_cal_range()` means:

Where is this time range inside today's data bars?

Or shorter:

`get_range_idx()` uses executor-relative indexes.

`get_data_cal_range()` uses day-relative data indexes.

---

## 5. Why This Matters for Crypto Adaptation

For crypto, this distinction is important because crypto trading is usually 24/7.

If the data is minute-level:

`1 day = 1440 minute bars`

Qlib's original intraday assumptions are more stock-market oriented, where one trading day may be treated like a trading session with about 240 minutes.

Therefore:

`get_range_idx()` is usually safer because it only depends on the current executor's own calendar.

`get_data_cal_range()` needs more attention in crypto adaptation because it explicitly calculates the index relative to the day start and assumes a day-based intraday data calendar.

---

## Final Summary

| Function               | Input                                | Output                                        | Zero Point                      | Main Use                                 |
| ---------------------- | ------------------------------------ | --------------------------------------------- | ------------------------------- | ---------------------------------------- |
| `get_range_idx()`      | explicit `start_time` and `end_time` | indexes inside current `TradeCalendarManager` | current executor's `start_time` | execution range control                  |
| `get_data_cal_range()` | `rtype="full"` or `rtype="step"`     | indexes inside current day's data calendar    | `day_start`                     | data-calendar / intraday bar positioning |

The most important point:

**They may return different numbers for the same time range because they use different zero points.**
