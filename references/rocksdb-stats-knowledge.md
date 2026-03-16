---
name: rocksdb-stats-knowledge
description: >-
  Use when adding metrics, tickers, histograms, or PerfContext counters to RocksDB features — reference for the statistics and profiling infrastructure
---

# RocksDB Statistics & Profiling Reference

## Components

| Component | Key Files | Purpose |
|---|---|---|
| Statistics (global) | `include/rocksdb/statistics.h`, `monitoring/statistics_impl.h`, `monitoring/statistics.cc` | Tickers (counters) and Histograms — per-core lock-free |
| PerfContext | `include/rocksdb/perf_context.h`, `monitoring/perf_context_imp.h` | Thread-local operation metrics (~60+ counters) |
| IOStatsContext | `include/rocksdb/iostats_context.h`, `monitoring/iostats_context_imp.h` | Thread-local I/O timing and byte counters |
| PerfLevel | `include/rocksdb/perf_level.h` | Controls profiling granularity per-thread |
| Histogram | `monitoring/histogram.h` | Distribution tracking (median, p95, p99, avg, stddev) |

## Adding New Metrics

### New Ticker

1. Open `include/rocksdb/statistics.h`.
2. Add a new enum value before `TICKER_ENUM_MAX` in the `Tickers` enum.
3. Open `monitoring/statistics.cc`.
4. Add a corresponding human-readable string entry in `TickersNameMap`.
5. Use `RecordTick(stats, YOUR_TICKER_NAME, count)` at the instrumentation site.

### New Histogram

1. Open `include/rocksdb/statistics.h`.
2. Add a new enum value before `HISTOGRAM_ENUM_MAX` in the `Histograms` enum.
3. Open `monitoring/statistics.cc`.
4. Add a corresponding human-readable string entry in `HistogramsNameMap`.
5. Use `RecordInHistogram(stats, YOUR_HISTOGRAM_NAME, value)` at the instrumentation site.

### New PerfContext Counter

1. Open `include/rocksdb/perf_context.h`.
2. Add a new `uint64_t` field to `PerfContextBase`.
3. Open `monitoring/perf_context.cc`.
4. Add the field to `DEF_PERF_CONTEXT_METRICS` so it is included in `Reset()` and `ToString()`.
5. Use `PERF_COUNTER_ADD(your_metric, value)` at the instrumentation site.

### New IOStats Counter

1. Open `include/rocksdb/iostats_context.h`.
2. Add a new `uint64_t` field to `IOStatsContext`.
3. Ensure the field is covered in the existing `Reset()` and `ToString()` methods in `monitoring/iostats_context.cc`.
4. Use `IOSTATS_ADD(your_metric, value)` at the instrumentation site.

## Key Macros

- `RecordTick(stats, TICKER_NAME, count)` — increment a global ticker by `count` (defaults to 1).
- `RecordInHistogram(stats, HISTOGRAM_NAME, value)` — record a single value into a histogram bucket.
- `PERF_TIMER_GUARD(metric)` — scoped wall-time measurement; starts on construction, records elapsed time on destruction.
- `PERF_CPU_TIMER_GUARD(metric, clock)` — scoped CPU-time measurement using the provided clock.
- `PERF_COUNTER_ADD(metric, value)` — increment a thread-local PerfContext counter by `value`.
- `PERF_COUNTER_BY_LEVEL_ADD(metric, value, level)` — increment a per-level PerfContext counter.
- `IOSTATS_TIMER_GUARD(metric)` — scoped I/O timing; records wall time spent in the guarded block.
- `IOSTATS_ADD(metric, value)` — increment a thread-local IOStatsContext counter by `value`.

## StatsLevel

Controls the overhead of the global Statistics object. Progression from least to most overhead:

1. `kDisableAll` — no stats collected.
2. `kExceptHistogramOrTimers` — tickers only, no histograms or timers.
3. `kExceptDetailedTimers` — tickers and histograms, but skip per-mutex and per-compression timers. **This is the default.**
4. `kExceptTimeForMutex` — everything except mutex-wait timing.
5. `kAll` — all stats collected, highest overhead.

## PerfLevel

Controls per-thread profiling granularity for PerfContext and IOStatsContext. Progression from least to most detail:

1. `kDisable` — profiling disabled.
2. `kEnableCount` — count-only metrics (no timing).
3. `kEnableWait` — counts plus wait/delay timing.
4. `kEnableTimeExceptForMutex` — wall-clock timing except mutex waits.
5. `kEnableTimeAndCPUTimeExceptForMutex` — wall-clock and CPU timing except mutex waits.
6. `kEnableTime` — all timing including mutex waits, highest overhead.
