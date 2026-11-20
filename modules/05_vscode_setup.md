# Module 5 setup: Python notebooks in VS Code (one time, about 15 minutes)

Modules 5 and 6 run on your Mac in VS Code. You'll set up a **virtual environment**: a private folder of Python packages just for this project, so it can never clash with other projects.

> **🧠 Why a virtual environment?** Every project needs particular package versions. If they're all installed globally, upgrading one project's pandas can break another. A `.venv` folder per project, plus `requirements.txt`, means anyone can recreate your exact setup with one command. Interviewers sometimes ask how you'd make your analysis reproducible, and this is part of the answer.

---

## 1. Check Python

In the VS Code terminal (`` Ctrl+` ``), from the repo folder:

```bash
python3 --version
```

You need **3.10 or newer**. If the command isn't found or the version is older, install Python from <https://www.python.org/downloads/>, then reopen VS Code.

## 2. Create and activate the environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Your terminal prompt now starts with `(.venv)`. Whenever you come back to the project in a new terminal, run `source .venv/bin/activate` again.

## 3. Install the packages

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

This takes a few minutes (pandas, SciPy, statsmodels, matplotlib, pyarrow, Jupyter).

## 4. VS Code extensions

In the Extensions panel (`Cmd+Shift+X`), install **Python** and **Jupyter** (both by Microsoft).

## 5. Open the notebook and pick the kernel

1. Open `python/05_q2_offplan_vs_ready.ipynb`.
2. Top right: click **Select Kernel → Python Environments → `.venv`**.
3. Run the first (Setup) cell with **Shift + Enter**. It should print `Data file found: True`.

If it prints `False`, the Parquet file isn't in `data/extracts/`. Go back to `databricks/05_export_q2_extract.ipynb`.

## 6. Keep the environment out of Git

`.gitignore` already contains `.venv/`. Check that `git status` doesn't list it.

---

**Troubleshooting**

| Problem | Fix |
|---------|-----|
| `ModuleNotFoundError: No module named 'pandas'` | The notebook isn't using `.venv`. Re-select the kernel (step 5) |
| `ImportError: Unable to find a usable engine … parquet` | Run `pip install pyarrow` with `.venv` active |
| `source: no such file` | You're not in the repo folder: `cd` into `Dubai_Real_Estate_Analytics` first |
| Charts don't appear | Run the cell again. The first matplotlib import can be slow |
