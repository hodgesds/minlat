# MINLAT: Minimum-Latency Scheduling Class for Linux

A new Linux scheduler class that replaces CFS/EEVDF for all fair-class tasks,
optimized for minimum wake-to-run latency.

## Overview

MINLAT (`SCHED_CLASS_MINLAT`) is a vruntime-based proportional-share scheduler
that sits between the RT and fair classes in priority. When enabled via
`CONFIG_SCHED_CLASS_MINLAT=y`, it takes over all `SCHED_NORMAL`, `SCHED_BATCH`,
and `SCHED_IDLE` tasks automatically. It can be toggled at runtime without
rebooting.

## Design

### Core Scheduling

MINLAT uses **pure vruntime-based fairness** — the same proportional-share
model as pre-EEVDF CFS. Tasks are ordered in a red-black tree by virtual
runtime. Each task's vruntime advances proportionally to its weight (derived
from nice value), ensuring weighted fair sharing.

Key mechanisms:

- **O(1) tick preemption**: Instead of scanning the rb-tree for a competitor
  (O(k) where k = delayed entities), MINLAT compares `delta_exec` against
  `ideal_runtime` directly. Uses `resched_curr_lazy()` to batch context
  switches, reducing preemption storms in producer-consumer workloads.

- **Delayed dequeue**: Short-running tasks (IPC pattern: pipe, futex, message
  passing) stay on the runqueue after sleeping. When they wake up,
  `ttwu_runnable()` re-enables them in O(1) without the full `try_to_wake_up`
  path. Capped at `max(2, nr_running/4)` delayed entities per CPU to bound
  scan cost while scaling with queue depth.

- **Wakeup buddy**: `WF_SYNC` wakeups (where the waker is about to sleep) set
  the wakee as a "buddy." The `__pick_next_task` fast path picks the buddy in
  O(1) without any tree traversal — critical for pipe/IPC workloads.

- **Adaptive wakeup preemption**: Non-sync wakeups preempt only when the
  wakee's vruntime advantage exceeds a configurable threshold (default 1ms).
  Below 2 effective tasks, preemption fires immediately with min-granularity
  protection to avoid preempting batch wake loops.

### Tradeoffs vs CFS/EEVDF

| Aspect | CFS/EEVDF | MINLAT |
|--------|-----------|--------|
| Fairness model | EEVDF virtual deadlines | Pure vruntime (CFS-classic) |
| Eligibility check | Per-entity on enqueue/wakeup | None |
| Tick preemption | O(1) via protect_slice | O(1) via delta_exec check |
| Wakeup latency | Full ttwu path | O(1) via delayed dequeue + buddy |
| Context switch cost | Higher (more entity state) | Lower (minimal per-entity state) |
| Strict fairness | Stronger (deadline guarantees) | Weaker (vruntime-only) |
| Starvation protection | EEVDF eligibility | Min-granularity + delayed cap |

**When MINLAT wins**: IPC-heavy workloads (pipe, futex, hackbench),
latency-sensitive interactive use, producer-consumer patterns, schbench-style
request processing.

**When EEVDF may be better**: Workloads that depend on strict short-term
fairness guarantees, or pathological cases where vruntime-only scheduling
allows temporary unfairness.

### Task Placement

- **Per-tgid LLC affinity**: Tracks which LLC a process's threads prefer and
  packs new threads into the same LLC. Reduces cross-LLC cache misses on
  context switch.
- **LLC stickiness**: Tasks must run at least N times (default 1) on their
  current LLC before becoming eligible for cross-LLC pull. Prevents migration
  ping-pong.
- **SMT-aware idle core preference**: Prefers idle cores over idle SMT siblings
  to avoid resource contention.
- **NUMA balancing**: Page-fault-based memory migration with preferred node
  tracking. `select_task_rq` prefers CPUs on the task's preferred NUMA node.
- **Wake-affine**: Heuristic that keeps communicating tasks on the same or
  nearby CPUs, with wake-wide override for fan-out patterns.

### Load Balancing

- **PELT-driven periodic balancing**: Walks the sched domain hierarchy, scoring
  migration candidates by LLC affinity, cache hotness, migration cooldown, and
  NUMA saturation.
- **Idle pull**: On `newidle_balance`, scans overloaded CPUs in the same domain
  for stealable tasks.
- **Active balancing**: When idle pull fails, the tick detects idle CPUs and
  dispatches a `cpu_stopper` to pull tasks. Only targets idle CPUs to avoid
  stopper storms that disrupt producer-consumer locality.
- **Overloaded tracking**: Atomic counter provides O(1) "any CPU overloaded?"
  check, avoiding unnecessary balance attempts.

### PELT and Frequency Scaling

- Per-entity and per-rq PELT load tracking, integrated with `schedutil` for
  CPU frequency scaling.
- `UTIL_EST`: EWMA-based utilization estimation that predicts CPU demand across
  sleep/wake cycles.
- Misfit detection for asymmetric capacity (big.LITTLE / Intel hybrid) systems.
- Energy Aware Scheduling (EAS) for heterogeneous CPU topologies.
- Uclamp-aware task placement (behind `CONFIG_UCLAMP_TASK`).

### Cgroup Support

- `cpu.weight`: Proportional CPU sharing between cgroups using weight-based
  vruntime scaling.
- `cpu.idle`: SCHED_IDLE semantics for background cgroups.
- `cpu.max` / `cpu.burst`: CFS bandwidth throttling, reusing the existing
  `cfs_bandwidth` infrastructure. Throttled tasks are dequeued from the rb-tree
  into a per-rq list and re-enqueued on unthrottle.

### Latency Nice

Per-task latency tuning via `sched_setattr()` with `sched_latency_nice` field
(-20 to 19). Maps to the nice weight table for exponential (~1.25x per step)
scaling of:
- Preemption threshold (how long curr must run before preemption)
- Wakeup placement credit (vruntime discount on wake)
- Min-granularity (minimum timeslice)

## Building

### Prerequisites

- Linux kernel source tree (tested on v7.0.0-rc3)
- Standard kernel build tools (`gcc`, `make`, `flex`, `bison`, etc.)

### Apply Patches

```bash
cd /path/to/linux
git am ~/minlat/patches/000[1-5]-*.patch
```

### Configure

```bash
# Enable MINLAT in an existing config
scripts/config --enable CONFIG_SCHED_CLASS_MINLAT

# Update config defaults
make olddefconfig
```

### Build

```bash
make -j$(nproc) bzImage
```

### Install

```bash
# Standard kernel install
sudo make modules_install
sudo make install

# Or for QEMU/virtme testing
vng -r arch/x86/boot/bzImage --disable-microvm --cpus 16 -- "your_command"
```

## Runtime Control

### Toggle On/Off

```bash
# Check current state (1 = enabled, 0 = disabled)
cat /sys/kernel/debug/sched/minlat/enabled

# Disable (switch all tasks back to CFS/EEVDF)
echo 0 > /sys/kernel/debug/sched/minlat/enabled

# Re-enable
echo 1 > /sys/kernel/debug/sched/minlat/enabled
```

### Tunable Parameters

All tunables are in `/sys/kernel/debug/sched/minlat/`:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `latency_ns` | 1500000 | Target scheduling latency (ns) |
| `min_granularity_ns` | 500000 | Minimum timeslice (ns) |
| `cache_hot_ns` | 500000 | Cache-hot threshold for migration |
| `wakeup_preempt_thresh_ns` | 1000000 | Non-sync wakeup preemption threshold |
| `wake_affine` | 1 | Enable wake-affine heuristic |
| `fork_imbalance_pct` | 25 | Cross-LLC fork imbalance threshold (%) |
| `fork_numa_imbalance_pct` | 50 | Cross-NUMA fork imbalance threshold (%) |
| `llc_stickiness` | 1 | Min runs before cross-LLC pull |
| `migration_cooldown_ns` | 4000000 | Min time between migrations |
| `numa_imbalance_min` | 2 | Min task imbalance for NUMA migration |
| `numa_saturated_pct` | 75 | NUMA saturation threshold (%) |
| `interactive_big_prefer` | 0 | Prefer big cores for interactive tasks |
| `compute_big_prefer` | 0 | Prefer big cores for compute tasks |
| `preempt_resist_ns` | 2000000 | Min runtime before preemption allowed (ns) |

### Explicit Policy

When MINLAT is enabled, all `SCHED_NORMAL`/`SCHED_BATCH`/`SCHED_IDLE` tasks
are automatically scheduled by MINLAT. To explicitly set the `SCHED_MINLAT`
policy (policy 8) with a specific priority (0-7, 0 = highest), use
`sched_setattr()` directly since standard `chrt` has no support for it:

```c
struct sched_attr attr = {
    .size = sizeof(attr),
    .sched_policy = 8,       /* SCHED_MINLAT */
    .sched_priority = 0,     /* 0 = highest, 7 = lowest */
};
sched_setattr(pid, &attr, 0);
```
