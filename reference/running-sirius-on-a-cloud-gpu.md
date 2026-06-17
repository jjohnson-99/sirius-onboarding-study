# Running Sirius on a cloud GPU

Sirius **cannot run on a Mac** (Apple Silicon or Intel). It is a CUDA program: it routes
execution to an NVIDIA GPU via cuDF/RMM/cuCascade, and its build is pinned to Linux +
NVIDIA. If you've only been *reading* the repo so far, this is the note for when you want to
actually load the extension and watch a query hit the GPU — on a rented Linux + NVIDIA box.

> **You don't need this to study Sirius.** All the file-maps and explainers were written
> from the source; you can keep reading on the Mac indefinitely. Reach for this only when you
> want a live engine (run TPC-H, watch the scheduler, profile a kernel). For the local-only
> on-ramp, see [the "what you *can* do on a Mac" section](#what-you-can-still-do-on-the-mac).

## Why not the Mac (the three hard blockers)

Straight from `sirius/pixi.toml` — these aren't preferences, the build won't configure
otherwise:

| Blocker | Where it's pinned | Why the Mac fails |
|---|---|---|
| **OS** | `platforms = ["linux-64", "linux-aarch64"]` | No `osx-*` target — `pixi run make` won't even configure on macOS. |
| **GPU vendor + CUDA** | `cuda-nvcc`, `cuda-version = 12/13`, `libcudf` deps | Execution is **cuDF/RMM/cuCascade** = NVIDIA-CUDA only. There is no Metal/Apple-GPU backend. |
| **GPU architecture** | `CUDAARCHS = "75-real;80-real;86-real;89-real;90a-real;…"` | Those are NVIDIA **compute capabilities** (Turing → Blackwell). An Apple GPU isn't a CUDA arch at all. |

Your 16 GB of unified memory is *not* the limiter — the OS and the GPU are. (Even GPU VRAM
size is rarely the wall thanks to spilling; see
[GPU memory & spilling](explainers/gpu-memory-and-spilling.md).)

## What you need on the remote box

- **OS:** Linux, x86-64 (`linux-64`) or ARM64 (`linux-aarch64`).
- **GPU(s):** NVIDIA, **compute capability ≥ 7.5** (Turing or newer). Cheapest that qualifies
  is an **NVIDIA T4** (cc 7.5). RTX 20-series and up, A10/A100/L4/H100 all qualify too.
  *Pre-Turing cards (cc < 7.5, e.g. older K80/P100=6.0) are not in `CUDAARCHS` and won't run.*
  **Get two GPUs.** Sirius is built for multi-GPU, and the parts of the codebase you'll study
  in Weeks 4–7 — the task scheduler, per-GPU executors, cross-device data flow, and the
  spill/downgrade paths — only actually *exercise* when there's more than one device. A
  single GPU runs queries fine but silently skips every multi-device branch (the scheduler's
  device-preference matcher degenerates, the per-NUMA/per-GPU placement is moot). So prefer a
  **2× T4** (or 2× L4) box; one GPU is acceptable only if you just want to watch *a* query run.
- **NVIDIA driver:** installed on the host (the cloud GPU images normally ship with it —
  verify with `nvidia-smi`). The CUDA *toolkit* itself comes from pixi, so you don't have to
  hand-install CUDA — but the kernel **driver** must be present and new enough for the CUDA
  version you pick.
- **CUDA 12 vs 13:** match the **driver** on the box. Newer driver → `default` env (CUDA 13);
  older driver → the `cuda12` env. You choose this at build time (below), not at install time.
- **Disk:** the build is heavy (DuckDB submodule + vcpkg/conda deps + CUDA objects). Give the
  instance **plenty of headroom — ~40–60 GB free** is comfortable.

### Picking an instance (aim for 2 GPUs)

Default to a **2-GPU** box so the scheduler/multi-device paths are live:

| Provider | 2-GPU SKU | GPUs | Single-GPU fallback |
|---|---|---|---|
| RunPod / Vast.ai / Lambda | filter pods to **GPU count = 2** | 2× T4 / L4 / A10 | 1× of the same |
| AWS | `g4dn.12xlarge` (4× T4) — or the 2× tiers where offered | 4× T4 | `g4dn.xlarge` (1× T4) |
| GCP | `n1-*` + **2×** `nvidia-tesla-t4`, or `g2` with 2 L4 | 2× T4 / L4 | 1× T4 |

The GPU-specialist hosts (RunPod/Vast/Lambda) are the cheapest and let you pick the GPU
**count** directly — set it to 2. On AWS the clean 2× T4 tier is uneven by region, so the
4×-T4 `g4dn.12xlarge` is the reliable multi-GPU option (pricier — stop it promptly); GCP lets
you attach exactly 2 T4s to an `n1`. A **single-GPU T4 pod** is the cheap fallback, but only if
you've accepted that the Week 4–7 multi-device branches won't run (see the GPU bullet above).

## The setup, end to end

These mirror `sirius/CLAUDE.md` (Build & test) — run them on the **remote Linux box**, not
the Mac.

```bash
# 0. Prereqs on the box: confirm the GPU + driver are visible
nvidia-smi                         # must list your GPU; note the CUDA version it reports

# 1. Get pixi (the project's env manager) if it isn't already installed
curl -fsSL https://pixi.sh/install.sh | bash
#   …then restart the shell / source your rc so `pixi` is on PATH

# 2. Clone Sirius and its submodules
git clone https://github.com/sirius-db/sirius.git
cd sirius
git submodule update --init --recursive   # duckdb, cucascade, substrait, vcpkg, …

# 3. Build  (pick the env that matches the driver)
pixi run make                      # CUDA 13 (the `default` env)
#   OR, on an older driver:
pixi run -e cuda12 make            # CUDA 12

# 4. Sanity-check with the C++ unit tests (what CI runs)
pixi run make test
```

> **CUDA 12 vs 13 = which pixi environment.** `pixi run make` uses `default` (= `dev-libs` +
> `cuda13`). `pixi run -e cuda12 make` switches to the CUDA-12 feature set. Both pull `libcudf`
> and the CUDA toolkit into the pixi env; the only thing the *host* must provide is a driver new
> enough for that CUDA version. If the build/runtime complains about CUDA/driver version
> mismatch, you picked the wrong one — flip to the other env.

> **First build is slow** (DuckDB + deps from source). If a build fails partway, clean before
> retrying: `pixi run make clean`, then build again.

## Running a query

Once built, the extension is at
`build/release/extension/sirius/sirius.duckdb_extension`. Load it into DuckDB and run normal
SQL — Sirius **transparently intercepts** supported queries and runs them on the GPU (no
special syntax; controlled by the `gpu_execution` setting, on by default):

```sql
LOAD 'build/release/extension/sirius/sirius.duckdb_extension';
SELECT count(*) FROM range(1000000);   -- routed to the GPU
-- SET gpu_execution = false;          -- to turn interception off and compare
```

To run the bundled SQL test suite (e.g. TPC-H) instead:

```bash
pixi run build/release/test/unittest --test-dir . test/sql/tpch-sirius.test
```

(That, and how to run a single Catch2 test, are in
[testing Sirius](explainers/testing-sirius.md).)

## Cost & teardown reminders

- GPU instances bill **by the second/hour** and are not cheap — **stop or destroy** the pod
  when you're done; a forgotten running GPU box is the classic surprise bill. A **2-GPU** box
  is ~2× the per-hour rate of one, so the discipline matters more: build, study the scheduler
  path, then tear down (or snapshot — next bullet).
- The **first build dominates** the session; if you'll come back, prefer a provider that lets
  you **snapshot the disk / pause** the instance so you don't rebuild from scratch each time.
- Spot/interruptible instances are fine for studying (cheapest), just expect they can vanish
  mid-session.

## What you can still do on the Mac

You lose the GPU path, not the learning:

- **Read everything** — the source, the file-maps, the explainers. That's the bulk of
  onboarding and it's all local.
- **Run plain DuckDB** — it builds/runs natively on macOS. Sirius plugs *into* DuckDB's
  optimizer + execution; playing with vanilla DuckDB (logical plans, `EXPLAIN`, the SQL
  surface) teaches the layer Sirius hooks (see
  [DuckDB extension API](explainers/duckdb-extension-api.md) and
  [the DuckDB source map](duckdb-source-map.md)).
- **Build the study notes** — exactly what we've been doing.

When you're ready to go live, rent a box and follow the steps above.

## See also

- `sirius/CLAUDE.md` — canonical Build & test commands (source of the steps above).
- [testing Sirius](explainers/testing-sirius.md) — the build → test → pre-commit → PR loop.
- [GPU memory & spilling](explainers/gpu-memory-and-spilling.md) — why VRAM size is rarely the
  wall once it *is* running.
- [NUMA](explainers/numa.md) — relevant only once you move to a *multi-socket, multi-GPU* box.
