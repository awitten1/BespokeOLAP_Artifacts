# BespokeOLAP Artifacts

> Generated C++ artifacts for [Bespoke OLAP](https://github.com/DataManagementLab/BespokeOLAP) — synthesized, bespoke implementations of the **TPC-H** and **CEB** analytical benchmarks.

---

## 📂 Contents

| Directory | Benchmark | Execution Model |
|---|---|---|
| [`bespoke_tpch/`](bespoke_tpch/) | TPC-H Q1–Q22 | 🔁 Single-threaded |
| [`bespoke_tpch_multithreading/`](bespoke_tpch_multithreading/) | TPC-H Q1–Q22 | ⚡ Multi-threaded *(new)* |
| [`bespoke_ceb/`](bespoke_ceb/) | CEB Q1a–Q11b | 🔁 Single-threaded |
| [`bespoke_ceb_multithreading/`](bespoke_ceb_multithreading/) | CEB Q1a–Q11b | ⚡ Multi-threaded *(new)* |

---

## ⚙️ Requirements

- [uv](https://docs.astral.sh/uv/) (Python package manager)

```bash
uv sync
```

---

## 🚀 Build & Run

Two runner scripts compile and execute the generated C++ code:

| Script | Variants | Execution model |
|---|---|---|
| `run.py` | `bespoke_tpch/`, `bespoke_ceb/` | single-threaded, via the `bespoke-olap` FastTest/plugin harness |
| `run_mt.py` | `bespoke_tpch_multithreading/`, `bespoke_ceb_multithreading/` | multi-threaded, standalone driver |

```bash
python run.py    [--sf <scale_factor>] [--no-optimize] {tpch,ceb}
python run_mt.py [--sf <scale_factor>] [--no-optimize] [--parquet-dir DIR] [--build-only] {tpch,ceb}
```

### Options

| Argument | Description | Default |
|---|---|---|
| `benchmark` | Benchmark to run (`tpch` or `ceb`) | *(required)* |
| `--sf` | Scale factor | `1` |
| `--no-optimize` | Disable compiler optimizations (`-O0` instead of `-O3`) | *(optimizations enabled)* |
| `--parquet-dir` | *(run_mt.py only)* Override the parquet base dir | `/mnt/labstore/bespoke_olap/<dataset>_parquet` |
| `--build-only` | *(run_mt.py only)* Compile the engine but do not run it | *(build and run)* |

### Examples

```bash
# single-threaded TPC-H at scale factor 20
python run.py --sf 20 tpch

# multi-threaded TPC-H at scale factor 20
python run_mt.py --sf 20 tpch

# multi-threaded CEB
python run_mt.py ceb
```

### How `run_mt.py` works

The multi-threaded artifacts are a self-contained engine with a different
driver contract than the single-threaded harness, so `run_mt.py` does not reuse
the `bespoke-olap` plugin machinery. On each run it:

1. Generates query instantiations with the same generator `run.py` uses.
2. Writes three shim files into `bespoke_<benchmark>_multithreading/build_mt/`
   that the artifacts assume but do not ship: `trace.hpp` (no-op tracing
   macros), `query_api.hpp` (defines `QueryResult`), and a `main.cpp` driver
   that loads the parquet tables, builds the database, reads query lines from
   stdin and runs them.
3. Compiles every implementation `.cpp` (excluding the `*_api.cpp` plugin
   wrappers, which the standalone driver bypasses) plus the driver into
   `build_mt/db_mt`, linked against Arrow/Parquet via `pkg-config`.
4. Runs the engine, feeding the generated query lines on stdin. Per-query
   result rows are written to `bespoke_<benchmark>_multithreading/results/result<N>.csv`,
   and per-query timings are printed to stderr.

> **⚠️ Platform:** the multi-threaded engine targets **Linux**. `cpu_affinity.cpp`
> uses `<sched.h>`/`sched_setaffinity` and does not compile on macOS. Arrow and
> Parquet must be discoverable via `pkg-config`, and the scale-factor parquet
> data must exist under the parquet base dir (`.../sf<N>/`). Set `CXX` to pick a
> specific C++20 compiler.
