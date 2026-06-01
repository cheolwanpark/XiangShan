# XiangShan utilization stats

This file tracks the utilization counters added around `ad35965` and `61cc282`, and separates stats that are already available from stats that need more RTL counters.

Perf counters here are `XSPerfAccumulate` counters unless noted otherwise. They accumulate within the current perf dump window. Ratios should use counters from the same window. Repeated names such as `util_busy_cycle` are expected; perf logs include the module path prefix, so the module instance disambiguates them.

## Perf dump output format

`XSPerf*` counters are printed through `XSLog` at perf dump time. The exact line format is:

```text
[PERF ][time=<timer>] <module-path>: <counter>, <value>
```

The module path is part of the key. Do not flatten on `<counter>` alone, because names such as `util_busy_cycle` intentionally appear in many modules.

Counter helpers expand as follows:

- `XSPerfAccumulate("name", inc)` prints `name, <accumulated value>`.
- `XSPerfReference("name", value)` prints `name, <current value>`.
- `XSPerfMax("name", value, enable)` prints `name_max, <max value>`.
- `XSPerfHistogram("name", value, enable, start, stop, step)` prints summary counters `name_sum`, `name_mean`, `name_sampled`, `name_underflow`, `name_overflow`, plus bins named `name_<bin-start>_<bin-stop>`.

`DifftestPerf` lines use a different prefix:

```text
[DIFFTEST_PERF][time=<timer>] <counter>, <value>
```

Those are difftest-side counters, not the XSPerf utilization counters listed below.

## Extracting from perf dumps

Typical simulation logs contain the perf dump lines in `build/simv.log`. Filter XSPerf lines with:

```sh
rg '^\[PERF \]\[time=' build/simv.log
```

To extract `time`, `module`, `counter`, and `value` as tab-separated fields:

```sh
perl -ne 'print "$1\t$2\t$3\t$4\n" if /^\[PERF \]\[time=(\d+)\] ([^:]+): ([^,]+),\s*(\d+)/' build/simv.log
```

Use the tuple `(time, module, counter)` to select raw values, then calculate ratios from rows with the same `time` and `module`. For example, for a load unit module path:

```text
utilization       = util_busy_cycle / (util_busy_cycle + util_idle_cycle)
input accept rate = util_input_fire / util_input_valid
input block rate  = util_input_blocked / util_input_valid
```

Perf dump windows are controlled by the simulator:

- `--stat-cycles=N` dumps and then cleans counters every `N` cycles.
- `--warmup-instr=N` dumps and then cleans counters when warmup completes.
- Final simulator stats call `trigger_stat_dump()`, which dumps once without cleaning.

## Already ready or calculatable

| Area | Counters | Calculatable stats |
| --- | --- | --- |
| Function units | `fu_${name}_in_valid`, `fu_${name}_in_fire`, `fu_${name}_in_block`, `fu_${name}_out_valid`, `fu_${name}_out_fire`, `fu_${name}_out_block` | Input accept rate = `in_fire / in_valid`; input block rate = `in_block / in_valid`; output block rate = `out_block / out_valid`; FU throughput = `out_fire / cycles`. |
| Piped function units | `fu_${name}_pipe_busy_cycle`, `fu_${name}_pipe_idle_cycle` | Pipe utilization = `pipe_busy_cycle / (pipe_busy_cycle + pipe_idle_cycle)`. |
| Load unit top | `util_busy_cycle`, `util_idle_cycle`, `util_input_valid`, `util_input_fire`, `util_input_blocked`, `util_scalar_fire`, `util_vector_fire`, `util_replay_fire`, `util_prefetch_fire`, `util_dcache_req_stall`, `util_tlb_req_stall`, `util_*_wb_fire`, `util_wb_blocked` | Load-unit utilization, input accept rate, input mix, replay/prefetch share, DCache/TLB stall cycles, writeback throughput, vector writeback block rate. |
| Load S0 source arbitration | `util_${LoadEntrance}_valid`, `util_${LoadEntrance}_blocked`, `util_${LoadEntrance}_selected`, `util_${LoadEntrance}_fire` for `unalignTail`, `replayHiPrio`, `fastReplay`, `replayLoPrio`, `prefetchHiConf`, `vectorIssue`, `scalarIssue`, `prefetchLoConf` | Source request mix, arbitration loss/block rate, selected-to-fire loss. `selected` means source won arbitration before final S0 fire. |
| Store unit top | `util_busy_cycle`, `util_idle_cycle`, `util_input_valid`, `util_input_fire`, `util_input_blocked`, `util_scalar_fire`, `util_vector_fire`, `util_prefetch_fire`, `util_unalign_tail_fire`, `util_dcache_req_stall`, `util_tlb_req_stall`, `util_*_wb_fire`, `util_wb_blocked` | Store-unit utilization, input accept rate, input mix, DCache/TLB stall cycles, writeback throughput, vector writeback block rate. |
| Store S0 source arbitration | `util_${StoreEntrance}_valid`, `util_${StoreEntrance}_blocked`, `util_${StoreEntrance}_selected`, `util_${StoreEntrance}_fire` for `unalignTail`, `prefetch`, `vectorIssue`, `scalarIssue` | Source request mix, arbitration loss/block rate, selected-to-fire loss. |
| Vector split | `util_busy_cycle`, `util_idle_cycle`, `util_input_fire`, `util_input_blocked`, `util_output_fire`, `util_output_blocked`, `util_inactive_issue` | Split pipeline/buffer utilization, input/output throughput, inactive issue cycles. |
| Vector merge buffer | `util_busy_cycle`, `util_idle_cycle`, `util_allocated_entries`, `util_freelist_valid_count`, `util_split_enq_fire`, `util_split_enq_stall`, `util_pipeline_wb_valid`, `util_pipeline_wb_fire`, `util_final_wb_fire`, `util_final_wb_blocked` | Merge-buffer utilization, average allocated entries, enqueue stall rate, pipeline/final writeback throughput and block rate. |
| Vector segment unit | `util_busy_cycle`, `util_idle_cycle`, `util_input_fire`, `util_input_blocked`, `util_allocated_entries`, `util_dtlb_*`, `util_dcache_*`, `util_sbuffer_*`, `util_writeback_*`, `state_*` | Segment-unit utilization, queue pressure, DTLB/DCache/SBuffer/writeback throughput and stall rate, FSM state residency. |

`61cc282` fixed load-unit `util_wb_blocked` by counting vector writeback backpressure only. Reuse that behavior: scalar load writeback fire remains useful, but scalar ready should not be treated as a load-pipeline block source.

## Implementation required

| Missing stat | Required RTL work | Notes |
| --- | --- | --- |
| Peak occupancy | Add `XSPerfMax` for `util_allocated_entries` or valid-entry count in vector merge/segment buffers and load/store queues. | Current counters support average occupancy, not peak. |
| Occupancy distribution | Add `XSPerfHistogram` around allocated-entry counts. | Use only where distribution matters; histograms add several counters per site. |
| Flat-name consumers | Add unique prefixes to generic `util_*` counters or teach the parser to preserve module path. | Current perf logs print module path, so RTL rename is not required for normal XSPerf logs. |
| CSR/HPM visibility | Add `perfEvents` plumbing and map into CSR perf event tables. | Current utilization list is XSPerf-print oriented, not CSR event oriented. |

## Formulas

- Utilization = `util_busy_cycle / (util_busy_cycle + util_idle_cycle)`.
- Input accept rate = `util_input_fire / util_input_valid`.
- Input block rate = `util_input_blocked / util_input_valid`.
- Source arbitration block rate = `util_${source}_blocked / util_${source}_valid`.
- Source selected-to-fire rate = `util_${source}_fire / util_${source}_selected`.
- Average occupancy = `util_allocated_entries / cycles`.
- State residency = `state_${name} / sum(state_*)`.

Use zero-denominator guards in scripts when a source or state has no samples in a window. Always calculate with counters from the same perf dump `time` and the same module path.
