# qlib-code-notes

这个仓库用于整理阅读和使用 [Microsoft Qlib](https://github.com/microsoft/qlib) 时遇到的问题、源码入口和可复用配置片段。

目前它是一个轻量源码笔记库，不是完整教程。每篇笔记应该尽量做到：问题明确、结论先行、代码片段可直接复制、关键判断有源码链接。

## How to Use

- 想快速解决具体报错，先看 [QA.md](QA.md)。
- 想理解 Qlib 回测链路，优先看 `qlib/backtest/`。
- 想理解数据加载、处理和过滤，优先看 `qlib/data/`。
- 想理解实验记录和 MLflow 封装，优先看 `qlib/workflow/`。

## Notes Index

### General

| File | Topic |
| --- | --- |
| [QA.md](QA.md) | Qlib workflow 使用 MLflow filesystem tracking backend 时出现 deprecation warning 的原因和处理方式 |

### Qlib Core

| File | Topic |
| --- | --- |
| [qlib/config.md](qlib/config.md) | Qlib 全局配置对象 `C` 与配置注册流程 |

### Backtest

| File | Topic |
| --- | --- |
| [qlib/backtest/backtest.md](qlib/backtest/backtest.md) | `PORT_METRIC` 与 `INDICATOR_METRIC` 结果结构 |
| [qlib/backtest/calendar-index-time-alignment.md](qlib/backtest/calendar-index-time-alignment.md) | 回测日历、时间窗口和索引对齐 |
| [qlib/backtest/exchange.md](qlib/backtest/exchange.md) | `Exchange` 成交、价格和交易约束处理 |
| [qlib/backtest/executor-generator-flow.md](qlib/backtest/executor-generator-flow.md) | executor generator 与 `yield from` 执行流 |
| [qlib/backtest/executor-internal-data.md](qlib/backtest/executor-internal-data.md) | executor 内部数据结构和 `decision_list` |
| [qlib/backtest/indicator-sum-all-indicators.md](qlib/backtest/indicator-sum-all-indicators.md) | `Indicator.sum_all_indicators()` 聚合逻辑 |
| [qlib/backtest/indicator-update-flow.md](qlib/backtest/indicator-update-flow.md) | `Account.update_indicator()` 与指标更新流程 |
| [qlib/backtest/init.md](qlib/backtest/init.md) | backtest 初始化与 `format_decisions` |
| [qlib/backtest/nested-executor-infrastructure.md](qlib/backtest/nested-executor-infrastructure.md) | 嵌套 executor 的基础设施 |
| [qlib/backtest/utils.md](qlib/backtest/utils.md) | backtest 时间范围工具函数 |

### Data and Dataset

| File | Topic |
| --- | --- |
| [qlib/data/data.md](qlib/data/data.md) | Qlib data provider 系统 |
| [qlib/data/filter.md](qlib/data/filter.md) | `_getFilterSeries()` 与过滤逻辑 |
| [qlib/data/dataset/handler.md](qlib/data/dataset/handler.md) | `DataHandlerLP` 处理流水线 |
| [qlib/data/dataset/loader.md](qlib/data/dataset/loader.md) | `DLWParser._parse_fields_info()` |
| [qlib/data/dataset/processor.md](qlib/data/dataset/processor.md) | label 定义与 `DropnaLabel` |
| [qlib/data/dataset/storage.md](qlib/data/dataset/storage.md) | `HashingStockStorage` 与数据存储 |
| [qlib/data/dataset/utils.md](qlib/data/dataset/utils.md) | `fetch_df_by_col()` 工具函数 |

### Workflow

| File | Topic |
| --- | --- |
| [qlib/workflow/init.md](qlib/workflow/init.md) | `R`、`ExpManager`、`Experiment`、`Recorder` 的关系 |
| [qlib/workflow/record_temp.md](qlib/workflow/record_temp.md) | workflow record template 路径和记录模板 |
| [qlib/workflow/recorder.md](qlib/workflow/recorder.md) | recorder context manager 与运行记录生命周期 |

### Contrib

| File | Topic |
| --- | --- |
| [qlib/contrib/evaluate.md](qlib/contrib/evaluate.md) | evaluation 指标和 information ratio |
| [qlib/contrib/data/handler.md](qlib/contrib/data/handler.md) | contrib data handler 中的归一化选择 |
| [qlib/contrib/eva/alpha.md](qlib/contrib/eva/alpha.md) | alpha evaluation 的 MultiIndex label |
| [qlib/contrib/report/graph.md](qlib/contrib/report/graph.md) | report graph 子图布局 |
| [qlib/contrib/report/utils.md](qlib/contrib/report/utils.md) | report 工具函数和排序行为 |
| [qlib/contrib/report/analysis_model/analysis_model_performance.md](qlib/contrib/report/analysis_model/analysis_model_performance.md) | model performance 分析 |
| [qlib/contrib/report/analysis_position/report.md](qlib/contrib/report/analysis_position/report.md) | position report 数据预处理 |
| [qlib/contrib/report/analysis_position/risk_analysis.md](qlib/contrib/report/analysis_position/risk_analysis.md) | risk analysis groupby 和风险指标计算 |

### Utils

| File | Topic |
| --- | --- |
| [qlib/utils/__init__.md](qlib/utils/__init__.md) | utils 初始化和队列逻辑 |
| [qlib/utils/data.md](qlib/utils/data.md) | config copy/update 行为 |
| [qlib/utils/index_data.md](qlib/utils/index_data.md) | descriptor 驱动的 operator overload |
| [qlib/utils/paral.md](qlib/utils/paral.md) | `AsyncCaller` 与 recorder logging |
| [qlib/utils/serial.md](qlib/utils/serial.md) | `Serializable` 的 state 保留逻辑 |

### Scripts

| File | Topic |
| --- | --- |
| [scripts/dump_bin.md](scripts/dump_bin.md) | `scripts/dump_bin.py` 数据读取和 dump 流程 |

## Writing Rules

- 一篇笔记只解决一个具体问题或一个明确源码入口。
- 先写结论，再写原因、流程和可选方案。
- 代码片段优先给最小可运行配置。
- 涉及 Qlib 内部行为时，补上对应源码文件链接。
- 如果只是临时规避 warning，要明确说明它不会修复底层配置。
- 一级标题应描述主题，不要使用临时片段标题，例如 `After processing`、`Information ratio`、`Already sorted`。

## Suggested Structure

```text
# Qlib: short problem title

## Problem
## Short Answer
## Flow
## Code References
## Notes
```
