# Qlib: handle the MLflow filesystem tracking warning

## Problem

When running Qlib workflows with the default recorder settings, MLflow may print a warning similar to this:

```text
FutureWarning: Filesystem tracking backend (for example, ./mlruns) is deprecated ...
```

This is an MLflow tracking-store warning, not a Qlib model-training failure.

## Short Answer

For reusable experiments, configure Qlib's MLflow tracking URI to a database backend, usually SQLite for local work:

```python
from pathlib import Path

import qlib
from qlib.constant import REG_CN


def mlflow_sqlite_uri(path: str = "mlflow.db") -> str:
    return f"sqlite:///{Path(path).resolve().as_posix()}"


qlib.init(
    provider_uri="~/.qlib/qlib_data/cn_data",
    region=REG_CN,
    exp_manager={
        "class": "MLflowExpManager",
        "module_path": "qlib.workflow.expm",
        "kwargs": {
            "uri": mlflow_sqlite_uri("mlflow.db"),
            "default_exp_name": "Experiment",
        },
    },
)
```

This changes the default experiment-manager URI for later `R.start(...)`, `R.list_experiments()`, and recorder lookup calls in that Qlib process.

## Why It Happens

Qlib's default experiment manager is `MLflowExpManager`. In the default config, its `uri` points to a local filesystem-backed `mlruns` directory under the current working directory. Newer MLflow versions recommend using database-backed tracking stores, such as SQLite, MySQL, or PostgreSQL.

The old local filesystem backend can still work, but the warning is a signal that it is not the best long-term default for reproducible experiment tracking.

## Alternatives

### Set the URI after initialization

If Qlib is already initialized and you only need to change the default recorder URI, use `R.set_uri(...)`:

```python
from pathlib import Path
from qlib.workflow import R

R.set_uri(f"sqlite:///{Path('mlflow.db').resolve().as_posix()}")
```

Use this before starting or querying experiments.

### Set the URI for one run

`R.start(uri=...)` is useful when a single run should use a different tracking store:

```python
from pathlib import Path
from qlib.workflow import R

tracking_uri = f"sqlite:///{Path('mlflow.db').resolve().as_posix()}"

with R.start(experiment_name="train_model", uri=tracking_uri):
    model.fit(dataset)
    R.save_objects(trained_model=model)
```

Important: this URI is active only for that experiment context. If you later resume or query the same experiment, pass the same `uri` again or set it as the default with `qlib.init(..., exp_manager=...)` or `R.set_uri(...)`.

### Ignore or suppress the warning

For quick local tests, keeping the filesystem backend is usually fine. If you only want to hide the warning:

```python
import warnings

warnings.filterwarnings(
    "ignore",
    message=".*Filesystem tracking backend.*",
    category=FutureWarning,
    module="mlflow.tracking._tracking_service.utils",
)
```

This only suppresses the message. It does not change where Qlib stores experiment metadata.

## Code References

- [`qlib/config.py`](https://github.com/microsoft/qlib/blob/main/qlib/config.py): default `exp_manager` and default MLflow URI.
- [`qlib/workflow/__init__.py`](https://github.com/microsoft/qlib/blob/main/qlib/workflow/__init__.py): `R.start(...)`, `R.set_uri(...)`, and recorder-facing APIs.
- [`qlib/workflow/expm.py`](https://github.com/microsoft/qlib/blob/main/qlib/workflow/expm.py): `ExpManager.default_uri`, active URI handling, and `MLflowExpManager`.

## Practical Recommendation

Use `qlib.init(..., exp_manager=...)` for scripts and notebooks that you plan to rerun. Use `R.start(uri=...)` only when you intentionally want one experiment context to write somewhere else. Suppress the warning only for throwaway runs.
