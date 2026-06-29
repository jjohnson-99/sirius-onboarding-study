# `scan_manager/split_connector.cpp` → Split-Connector Map

The **hand-off queue** between the scan producer and the scan consumer — the single "meeting
point" of the GPU scan family. A lock-protected `deque` of pre-built splits plus a
`condition_variable`: the producer ([`split_provider`](split_provider.md) workers) enqueues splits
and `close()`s when done; the consumer ([`sirius_gpu_scan_operator`](../op/scan/sirius_gpu_scan_operator.md))
blocks in `get_next_split()` until a split arrives or the queue is closed-and-drained.

This map exists because the scan spine
([`file-maps/README.md` → "The scan spine"](../README.md)) and the scan walkthrough
([`../../notes/walkthroughs/scan-pipeline-and-filter-project.md`](../../notes/walkthroughs/scan-pipeline-and-filter-project.md)
§4–6) keep pointing at the connector as "where the two halves meet," but nothing mapped the
primitive itself.

> Paths relative to the Sirius repo root; study-note links relative to this file. Lines as of
> 2026-06-29 — re-grep if the file moved. Subject file is `src/scan_manager/split_connector.cpp`
> (impl) + `src/include/scan_manager/split_connector.hpp` (the class + the contract docs).

## Where this sits

```
PRODUCER  split_provider worker threads
   apply_balancing(split) → push_to_connector(connector, split)
        └─ (friend) split_connector::push_split  :29   push_back + notify_one
                                  │
                                  ▼
   ══════════════  split_connector  ══════════════   (lock + deque + cv)
                                  │
                                  ▼   blocks until a split arrives OR closed+drained
CONSUMER  sirius_gpu_scan_operator
   get_next_task_input_data → ingestible.consume_next_input(connector)
        └─ split_connector::get_next_split  :53   pop_front / nullopt / rethrow
```

One connector per `GPU_SCAN` operator, swapped in fresh each query by
`sirius_scan_manager::prepare_for_query` (`set_split_connector`, scan_manager `:277`). It carries
**three outcomes** on the consumer side, all through `get_next_split()`:

- **a non-null split** — more work;
- **`std::nullopt`** — closed and drained, no error: clean EOF;
- **throws** — the producer captured an error and passed it to `close(exception_ptr)`; it surfaces
  here once observed (see the short-circuit note below).

## Functions that matter

| Function | Lines | Role |
|---|---|---|
| ctor / dtor | :26-27 | Both defaulted. Non-copyable **and** non-movable (deleted, hpp :55-58) — it is referenced by address (the operator + the provider workers both hold `&connector`), so it must never relocate. |
| `push_split` *(private; `friend split_provider`)* | :29-38 | Producer enqueue. Asserts non-null and **`!_closed`** (push-after-close is a programming error), `push_back` under the lock, then `notify_one` (one waiter is enough — one consumer). Reachable only via [`split_provider::push_to_connector`](split_provider.md), so all producers route through the provider's friendship channel. |
| `close` | :40-51 | Mark closed; **idempotent**. First non-null `exception_ptr` across all calls **wins** (later calls don't overwrite, so the consumer sees the original cause). `notify_all` (wake every waiter to observe EOF/error). |
| `get_next_split` | :53-65 | Consumer pull. Waits on the cv for `!_splits.empty() || _closed`. **Rethrows `_exception` first if set — before checking for remaining splits (:58 precedes :59).** So a recorded producer error short-circuits draining: queued-but-unread splits are abandoned once an error is observed. Otherwise pop front, or return `nullopt` when closed and empty. |
| `is_closed` | :67-71 | `_closed && _splits.empty()` — **closed AND drained**, not merely "close() was called." The operator conjoins this with the ingestible's `consumer_drained()` to decide `all_ports_empty()`. |
| `has_more_splits` | :73-77 | `!_splits.empty()` — a momentary snapshot (queue non-empty *right now*); says nothing about whether the producer is still running. |

## Types fundamental to *this* file

- **`op::operator_data`** *(Sirius)* — the payload moved through the queue
  (`std::unique_ptr<op::operator_data>`). Concretely a `scan_operator_input` wrapping a
  `scan_and_filter_metadata` (the split descriptor the ingestible's `make_batch` / `run_batch`
  built). The connector is type-agnostic — it just transports owned `operator_data`.
- **`std::deque<std::unique_ptr<op::operator_data>> _splits`** — the FIFO of ready splits.
- **`std::exception_ptr _exception`** — the producer's captured failure (first-writer-wins),
  rethrown to the consumer. This is how a metadata-worker throw on a *different* thread becomes a
  query error on the consumer thread.
- **`std::mutex _mutex` / `std::condition_variable _cv` / `bool _closed`** — the standard
  lock+cv+flag triad. `_mutex` is `mutable` so the `const` observers (`is_closed`,
  `has_more_splits`) can lock.

## Why the friend gating

`push_split` is **private**; `split_connector` declares `friend class split_provider` (hpp :83).
Only [`split_provider::push_to_connector`](split_provider.md) (a protected static) can reach it, so
no unrelated code can enqueue a split out-of-band — every producer goes through the provider
abstraction. `close()` and the consumer methods stay public because the scan_manager (driver) and
the scan operator legitimately drive the lifecycle from outside.

## Takeaway

The connector is a deliberately tiny **MPSC-ish queue**: many producer worker threads push (gated
to `split_provider` via `friend`), one consumer blocks-and-pops. Its whole job is to decouple the
fire-and-forget metadata workers from the scheduler-driven consumer and to fuse three signals into
one call — *next split* / *clean EOF* (`nullopt`) / *producer error* (throw). The two non-obvious
contracts to remember: an observed error **short-circuits** any remaining queued splits
(`get_next_split` rethrows before serving more), and `is_closed()` means **closed *and* drained**.
Read with [`split_provider`](split_provider.md) (the producer that fills it) and
[`sirius_scan_manager`](sirius_scan_manager.md) (which wires one onto each operator).
