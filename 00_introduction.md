# Python for Data Science: From Basics to Expert

A complete, self-contained reference covering Python, NumPy, Pandas, Matplotlib,
SciPy, scikit-learn, and Plotly. Written to take you from zero to production-grade
data science work.

## Who this is for

- Beginners who want a structured, no-fluff path into Python and data science.
- Intermediate users who want to fill gaps and understand *why* things work.
- Advanced users who want a dense reference for edge cases and performance.

Each file is standalone but ordered. Read in sequence the first time; use as a
reference afterward.

## Reading order and map

```
                        ┌─────────────────────────────┐
                        │ 00  Introduction (this file)│
                        └──────────────┬──────────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         │                             │                             │
   PYTHON CORE                    NUMERICS                     VISUALIZATION
         │                             │                             │
  01 Fundamentals              06 NumPy Fundamentals         11 Matplotlib Fund.
  02 Data Structures           07 NumPy Advanced             12 Matplotlib Adv.
  03 Functions/Modules              │                        15 Plotly (interactive)
  04 OOP                        08 Pandas Fundamentals
  05 Advanced                   09 Pandas Intermediate
                                10 Pandas Advanced
                                       │
                                  SCIENCE / ML
                                       │
                                13 SciPy
                                14 scikit-learn
                                       │
                                16 Workflow & Best Practices
```

## The data science stack: how the pieces fit

```
   ┌──────────────────────────────────────────────────────────────┐
   │                        Your analysis code                    │
   └──────────────────────────────────────────────────────────────┘
        │            │              │            │            │
        v            v              v            v            v
   ┌─────────┐  ┌─────────┐   ┌──────────┐  ┌──────────┐  ┌─────────────┐
   │ pandas  │  │  scipy  │   │  sklearn │  │matplotlib│  │ plotly      │
   │(tabular)│  │(science)│   │   (ML)   │  │ (static) │  │(interactive)|
   └────┬────┘  └────┬────┘   └────┬─────┘  └────┬─────┘  └────┬────────┘
        │            │             │             │             │
        └────────────┴─────┬───────┴─────────────┴─────────────┘
                           v
                    ┌─────────────┐
                    │    NumPy    │   <- the foundation: the ndarray
                    │  (ndarray)  │      is the shared data structure
                    └──────┬──────┘
                           v
                    ┌─────────────┐
                    │   Python    │
                    └─────────────┘
```

NumPy's `ndarray` is the common currency. Pandas stores columns as NumPy arrays,
scikit-learn consumes NumPy arrays, Matplotlib plots them. Understanding NumPy
deeply pays off everywhere else.

## Environment setup

You do not need to run anything to read this tutorial, but to follow along:

```bash
# Create an isolated environment (recommended). Choose ONE approach.

# Option A: venv (standard library, no extra install)
python3 -m venv .venv
source .venv/bin/activate          # macOS/Linux
# .venv\Scripts\activate           # Windows PowerShell

# Option B: conda (better for compiled scientific packages on some systems)
# conda create -n ds python=3.12
# conda activate ds

# Install the stack
pip install numpy pandas matplotlib scipy scikit-learn plotly jupyter
```

Verify:

```python
import sys, numpy, pandas, matplotlib, scipy, sklearn, plotly
print(sys.version)
for m in (numpy, pandas, matplotlib, scipy, sklearn, plotly):
    print(f"{m.__name__:12s} {m.__version__}")
```

### Why a virtual environment

Isolating dependencies per project prevents version conflicts. A global install
means upgrading one project's pandas can silently break another. The trade-off is
minor overhead (activating the env); the payoff is reproducibility.

## How to run the code

- **Scripts**: save to `foo.py`, run `python foo.py`.
- **REPL**: type `python` for an interactive shell; good for quick checks.
- **Jupyter**: `jupyter lab` or `jupyter notebook`; best for exploratory data work
  because you keep state between cells and see plots inline.
- **IPython**: `pip install ipython`; a better REPL with tab completion, `?` help,
  and `%timeit` magics.

## Conventions used throughout

Standard import aliases (use these everywhere; the whole ecosystem assumes them):

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import scipy as sp
import plotly.express as px
```

- Code blocks are runnable as-is unless marked otherwise.
- Output is shown in comments like `# -> result` to keep blocks copy-pasteable.
- `[OK]`, `[WARN]`, `[ERROR]` mark notes; no emojis are used.
- Version target: Python 3.10+, NumPy 1.24+/2.x, pandas 2.x. Notes are added where
  behavior differs across versions.

## A note on learning strategy

Reading alone will not make you fluent. For each section:

1. Read the concept and the diagram.
2. Type the example yourself (do not copy-paste the first time).
3. Break it deliberately: change a value, remove an argument, see the error.
4. Reproduce the pattern on your own tiny dataset.

The fastest path to expertise is generating and reading many error messages.

Proceed to `01_python_fundamentals.md`.
