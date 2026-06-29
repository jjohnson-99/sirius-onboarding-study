# `scan_manager/split_provider.cpp` → Split-Provider Map

The **per-operator producer driver**: the thing `sirius_scan_manager` fire-and-forgets to fill a
[`split_connector`](split_connector.md). It composes one [`gpu_ingestible`](../op/scan/duckdb_native_gpu_ingestible.md)
(non-owning) and `run()`s it to completion — claiming batches from the ingestible, dispatching one
async task per batch onto a scheduler, and pushing each task's splits into the connector. It also
stamps a GPU placement preference on fresh-read splits via the balancing strategy.

This map covers **both** `src/scan_manager/split_provider.cpp` (only two out-of-line helpers) and
`src/include/scan_manager/split_provider.hpp` — the interesting logic (`run()` and the RAII
`worker_state`) lives in the **header** because `run()` is a template.

> Paths relative to the Sirius repo root; study-note links relative to this file. Lines as of
> 2026-06-29 — re-grep if the file moved. `.cpp` line refs unqualified; header refs tagged `hpp`.
> The `gpu_ingestible`-composition design is post-`#871` (see the historical note below).

## Where this sits

```
sirius_scan_manager::prepare_for_query
   provider = split_provider(op->get_ingestible())     borrows the ingestible (non-owning)   hpp :77
   provider->set_balancing_strategy(round_robin, pid)  shared placement policy                hpp :145
sirius_scan_manager::start_metadata_processing
   provider->run(*_dispatcher, *connector)   ── fire-and-forget ──┐                           hpp :230
                                                                   ▼
   while has_more_splits():                          delegate → ingestible.has_more_splits()  hpp :234
      work = next_split_provider()                   delegate → ingestible.next_split_provider() hpp :235
      scheduler.enqueue([state, work]{                one async task per claimed batch         hpp :239
         splits = work()                              ingestible does the metadata walk
         for split in splits:
            apply_balancing(split)                    pick a GPU (if no preference yet)        :33
            push_to_connector(connector, split)       → split_connector::push_split (friend)   :27
      })
   ── last task drops its worker_state ref ──▶ ~worker_state ──▶ connector.close(error_ptr)    hpp :215
```

The split of `has_more_splits()` (snapshot) from `next_split_provider()` (atomic claim) is what
lets `run()` enqueue **one task per batch** without serialising the metadata walks behind a chain
of "claim-then-run" tasks — the claims happen on the driver thread, the walks happen in parallel on
the pool.

## The control flow (`run` → task → connector)

```
split_provider::run<Scheduler>(scheduler, connector)            hpp :230-251
├─ state = worker_state::create(connector)                      hpp :233   RAII close-on-last-ref
└─ while has_more_splits():                                     hpp :234
   ├─ work = next_split_provider()                              hpp :235   atomic claim of a batch
   ├─ if !work: continue                                        hpp :238   lost a race; skip empty handoff
   └─ scheduler.enqueue(task)  capturing [this, state, work]    hpp :239
         task():
         ├─ try:
         │    ├─ splits = work()                                hpp :241   ingestible walks metadata
         │    └─ for split in splits:
         │         ├─ if split: apply_balancing(*split)         hpp :243 / :33
         │         └─ push_to_connector(state->connector, split) hpp :244 / :27
         └─ catch (...): state->set_error(current_exception())  hpp :246   first-writer-wins CAS
```

`run()` returns to the driver as soon as every batch is **enqueued** (enqueue is non-blocking);
the actual production continues on the pool. EOF/error reach the consumer only when the **last
task** releases its `shared_ptr<worker_state>` and the destructor fires `connector.close()`.

## Functions that matter

| Function | Lines | Role |
|---|---|---|
| `split_provider()` *(default)* | hpp :68 | Legacy ctor for the pre-`#871` subclasses that override both virtuals (their `_ingestible` stays null). Removed when those are deleted. |
| `split_provider(gpu_ingestible&)` | hpp :77 | The live path — **borrows** the ingestible non-owningly. The operator owns the lifetime; the provider is always torn down first (by `scan_manager::reset`). |
| `run<Scheduler>` *(template)* | hpp :230-251 | The driver loop above. `Scheduler` is any type with non-blocking `enqueue(callable)` — `static_thread_pool` and `scoped_dispatcher` both fit. |
| `has_more_splits` *(virtual)* | hpp :106-109 | Default: delegate to `_ingestible->has_more_splits()`. Snapshot check. |
| `next_split_provider` *(virtual)* | hpp :121-125 | Default: delegate to the ingestible; returns a null `std::function` when no batch is left (keeps the contract simple under concurrent observers). |
| `push_to_connector` *(protected static)* | :27-31 | The **only** enqueue path into a connector: reaches `split_connector::push_split` (private) through the `friend` relationship. Protected+static ⇒ only the provider hierarchy can call it. Out-of-line because the `unique_ptr` deleter needs the complete `op::operator_data`. |
| `apply_balancing` | :33-42 | Stamp a GPU on a fresh-read split. No-op if no strategy installed; **skips** resident splits (`is_resident()`, pinned-cache batches already on a device) and splits that already carry a `preferred_device_id`. Otherwise `strategy->get_next_gpu(_pipeline_id, &split)` → `set_preferred_device_id`. |
| `set_balancing_strategy` | hpp :145-149 | Wired by `scan_manager::prepare_for_query`; the strategy may be **shared** across providers so a round-robin balances over the whole scan stage. Null = placement disabled (task creator falls back to its own locality logic). |
| `get_ingestible` | hpp :131 | Accessor; promote to `shared_ptr` via `get_ingestible().shared_from_this()` if needed. |

## Types fundamental to *this* file

- **`io::gpu_ingestible`** *(Sirius, composed non-owningly)* — the per-format decode+split object
  the provider delegates `has_more_splits` / `next_split_provider` to. The polymorphic seam; the
  provider itself is format-agnostic. See
  [`duckdb_native_gpu_ingestible`](../op/scan/duckdb_native_gpu_ingestible.md) /
  [`parquet_gpu_ingestible`](../op/scan/parquet_gpu_ingestible.md) /
  [`pinned_table_gpu_ingestible`](../op/scan/pinned_table_gpu_ingestible.md).
- **`worker_state`** *(Sirius; private nested struct, hpp :199-227)* — **RAII coordination** shared
  by all the enqueued tasks via `shared_ptr`. Holds the `split_connector&` and a first-writer-wins
  error: `set_error` CAS-guards `error_set` (`std::atomic<bool>`) so only the first thrower's
  `exception_ptr` is kept (no mutex). The destructor — firing when the **last** task drops its ref
  — calls `connector.close(error_ptr)` (swallowing any close exception, since destructors must not
  throw). This is the mechanism that turns "all batches done / some batch threw" into a single
  `close()` on the connector.
- **`balancing_strategy`** *(Sirius, shared_ptr)* — the GPU placement policy (e.g.
  `round_robin_strategy`); `get_next_gpu(pipeline_id, split*)` returns a device id or `<0` to
  decline. Shared across providers for stage-wide balance.
- **`split_connector`** *(Sirius)* — the destination queue; see [`split_connector`](split_connector.md).
  The provider is its sole producer (friend-gated).
- **`Scheduler`** *(template param)* — duck-typed `enqueue(callable)`; the scan_manager passes its
  dispatcher/thread-pool.

## Historical note (why the virtuals + default ctor exist)

`split_provider` was **abstract** pre-`gpu_ingestible` refactor: subclasses
`parquet_split_provider`, `duckdb_native_split_provider`, `cached_split_provider` overrode
`has_more_splits` / `next_split_provider` and used the default ctor (null `_ingestible`). They are
slated for deletion (step 10 of that refactor). The **live** path is the concrete one: construct
with an ingestible and let the base's default virtuals delegate to it. So if you see those
subclasses, treat them as on the way out — the composition path is the current design.

## Takeaway

The split_provider is the **producer engine** of the GPU scan: a thin, format-agnostic driver that
`run()`s a composed ingestible, fans one async task per batch onto a scheduler, places each split
on a GPU (balancing strategy), and pushes it through the friend-gated channel into the
[`split_connector`](split_connector.md). Its cleverness is all in lifecycle: a `shared_ptr<worker_state>`
captured by every task closes the connector **exactly once**, when the last task finishes,
forwarding the first error a worker hit. Read with [`split_connector`](split_connector.md) (its
output) and [`sirius_scan_manager`](sirius_scan_manager.md) (which builds and fire-and-forgets it).
