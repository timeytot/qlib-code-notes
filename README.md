# qlib-code-notes

This repository collects source-reading notes for [Microsoft Qlib](https://github.com/microsoft/qlib): code entry points, behavior notes, configuration snippets, and issue-oriented Q&A.

It is a lightweight source-notes repository, not a full tutorial. Each note should keep the problem explicit, put the conclusion early, include copyable snippets when useful, and link important claims back to Qlib source files.

## How to Use

- For concrete warnings or common usage questions, start with [QA.md](QA.md).
- For the backtest execution path, start with `qlib/backtest/`.
- For data loading, processing, and filtering, start with `qlib/data/`.
- For experiment tracking and MLflow wrapping, start with `qlib/workflow/`.

## Notes Index

### General

| File | Topic |
| --- | --- |
| [QA.md](QA.md) | MLflow filesystem tracking warning in Qlib workflows |

### Qlib Core

| File | Topic |
| --- | --- |
| [qlib/config.md](qlib/config.md) | Global `C` config and temporary config objects |

### Backtest

| File | Topic |
| --- | --- |
| [qlib/backtest/backtest.md](qlib/backtest/backtest.md) | `PORT_METRIC` and `INDICATOR_METRIC` result structures |
| [qlib/backtest/calendar-index-time-alignment.md](qlib/backtest/calendar-index-time-alignment.md) | Calendar, time-window, and index alignment |
| [qlib/backtest/exchange.md](qlib/backtest/exchange.md) | Volume limits, trading limits, and execution prices |
| [qlib/backtest/executor-generator-flow.md](qlib/backtest/executor-generator-flow.md) | Executor generator and yield-from control flow |
| [qlib/backtest/executor-internal-data.md](qlib/backtest/executor-internal-data.md) | `decision_list` and indicator data inside executors |
| [qlib/backtest/indicator-sum-all-indicators.md](qlib/backtest/indicator-sum-all-indicators.md) | `sum_all_indicators` and `sum_by_index` aggregation |
| [qlib/backtest/indicator-update-flow.md](qlib/backtest/indicator-update-flow.md) | `Account.update_indicator()` and indicator update flow |
| [qlib/backtest/init.md](qlib/backtest/init.md) | Backtest initialization and `format_decisions` |
| [qlib/backtest/nested-executor-infrastructure.md](qlib/backtest/nested-executor-infrastructure.md) | Nested executor infrastructure |
| [qlib/backtest/utils.md](qlib/backtest/utils.md) | Calendar helper functions |

### Data and Dataset

| File | Topic |
| --- | --- |
| [qlib/data/data.md](qlib/data/data.md) | Data provider wrapper and registration system |
| [qlib/data/filter.md](qlib/data/filter.md) | Expression-based dynamic filtering |
| [qlib/data/dataset/handler.md](qlib/data/dataset/handler.md) | `DataHandlerLP` processing pipeline |
| [qlib/data/dataset/loader.md](qlib/data/dataset/loader.md) | `DLWParser._parse_fields_info` |
| [qlib/data/dataset/processor.md](qlib/data/dataset/processor.md) | Labels and `DropnaLabel` |
| [qlib/data/dataset/storage.md](qlib/data/dataset/storage.md) | `HashingStockStorage` |
| [qlib/data/dataset/utils.md](qlib/data/dataset/utils.md) | `fetch_df_by_col` |

### Workflow

| File | Topic |
| --- | --- |
| [qlib/workflow/init.md](qlib/workflow/init.md) | `R`, `ExpManager`, `Experiment`, and `Recorder` |
| [qlib/workflow/record_temp.md](qlib/workflow/record_temp.md) | Record template paths and dependency checks |
| [qlib/workflow/recorder.md](qlib/workflow/recorder.md) | Recorder shutdown and `async_log` waiting |

### Contrib

| File | Topic |
| --- | --- |
| [qlib/contrib/evaluate.md](qlib/contrib/evaluate.md) | `risk_analysis`, `indicator_analysis`, and `backtest_daily` |
| [qlib/contrib/data/handler.md](qlib/contrib/data/handler.md) | `CSZScoreNorm` and `ZScoreNorm` in default handlers |
| [qlib/contrib/eva/alpha.md](qlib/contrib/eva/alpha.md) | Alpha evaluation with MultiIndex labels |
| [qlib/contrib/report/graph.md](qlib/contrib/report/graph.md) | Automatic subplot layout |
| [qlib/contrib/report/utils.md](qlib/contrib/report/utils.md) | `guess_plotly_rangebreaks` |
| [qlib/contrib/report/analysis_model/analysis_model_performance.md](qlib/contrib/report/analysis_model/analysis_model_performance.md) | Group returns and IC analysis |
| [qlib/contrib/report/analysis_position/report.md](qlib/contrib/report/analysis_position/report.md) | Position report preprocessing |
| [qlib/contrib/report/analysis_position/risk_analysis.md](qlib/contrib/report/analysis_position/risk_analysis.md) | Monthly risk aggregation |

### Utils

| File | Topic |
| --- | --- |
| [qlib/utils/__init__.md](qlib/utils/__init__.md) | Placeholder replacement and wrapper registration |
| [qlib/utils/data.md](qlib/utils/data.md) | `deepcopy_basic_type` and `robust_zscore` |
| [qlib/utils/index_data.md](qlib/utils/index_data.md) | Descriptor-powered operator overloading |
| [qlib/utils/paral.md](qlib/utils/paral.md) | `AsyncCaller` and recorder logging |
| [qlib/utils/serial.md](qlib/utils/serial.md) | `Serializable._is_kept` and `__getstate__` |

### Scripts

| File | Topic |
| --- | --- |
| [scripts/dump_bin.md](scripts/dump_bin.md) | `scripts/dump_bin.py` data loading and dump pipeline |

## Writing Rules

- One note should cover one concrete problem or one source entry point.
- Lead with the conclusion, then explain the cause, flow, and tradeoffs.
- Prefer minimal, copyable code snippets.
- Link important behavioral claims back to Qlib source files.
- If a note only suppresses a warning, state clearly that it does not change the underlying configuration.
- Use descriptive H1 titles, not temporary snippet titles such as `After processing`, `Information ratio`, or `Already sorted`.
- Keep English notes ASCII-friendly unless a non-ASCII symbol is part of an API or user-facing output.

## Suggested Structure

```text
# Qlib: short problem title

## Problem
## Short Answer
## Flow
## Code References
## Notes
```

## Review Checks

Before committing broad documentation updates, run these checks locally against this repository and a matching Qlib source checkout:

- all Markdown files have descriptive top-level headings;
- referenced Qlib source paths exist;
- referenced source line anchors are within file bounds;
- obvious temporary phrasing and presentation-only wording are removed;
- script paths match the Qlib repository layout.
