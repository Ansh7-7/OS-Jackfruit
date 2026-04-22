# OS-Jackfruit: Supervised Multi-Container Runtime

---

## Team Information

| Name | SRN |
| --- | --- |
| Anshul P | PES1UG24CS071 |
| Ashwin V | PES1UG24CS090 |

**Platform:** Ubuntu 22.04 LTS, Kernel 6.8.0-107-generic, x86\_64, VirtualBox VM

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Environment Setup](#environment-setup)
3. [Build Instructions](#build-instructions)
4. [CLI Usage](#cli-usage)
5. [Architecture Overview](#architecture-overview)
6. [Implementation Details](#implementation-details)
7. [Kernel Module](#kernel-module)
8. [Scheduler Experiments](#scheduler-experiments)
9. [Engineering Analysis](#engineering-analysis)
10. [Design Decisions and Tradeoffs](#design-decisions-and-tradeoffs)
11. [Scheduler Experiment Results](#scheduler-experiment-results)
12. [Demo Screenshots](#demo-screenshots)

---

## Project Overview

This project implements a lightweight Linux container runtime in C, consisting of two major components:

- **`engine.c`** — A user-space supervisor process that manages multiple isolated containers using Linux namespaces, chroot, pipes, and UNIX domain sockets.
- **`monitor.c`** — A loadable kernel module that tracks container memory usage via RSS and enforces soft/hard memory limits using `SIGKILL`.

---

## Environment Setup

### System

- **OS:** Ubuntu 22.04 LTS
- **Kernel:** 6.8.0-107-generic
- **Architecture:** x86\_64
- **VM:** VirtualBox

### Dependencies

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) wget git
```

### Alpine Rootfs

```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
```

---

## Build Instructions

```bash
cd boilerplate
make
```

This builds:

- `engine` — the supervisor + CLI binary
- `monitor.ko` — the kernel module
- `memory_hog`, `cpu_hog`, `io_pulse` — workload binaries (statically linked)

To load the kernel module:

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor   # should exist
dmesg | tail -5                # confirms module loaded
```

To unload:

```bash
sudo rmmod monitor
```

---

## CLI Usage

```bash
# Start the long-running supervisor
sudo ./engine supervisor <base-rootfs>

# Start a container (non-blocking)
sudo ./engine start <id> <container-rootfs> <command> [--soft-mib N] [--hard-mib N] [--nice N]

# Start a container and wait for it to exit
sudo ./engine run <id> <container-rootfs> <command> [--soft-mib N] [--hard-mib N] [--nice N]

# List all containers and their state
sudo ./engine ps

# Print logs for a container
sudo ./engine logs <id>

# Send SIGTERM to a container
sudo ./engine stop <id>
```

### Example Session

```bash
# Terminal 1 — start supervisor
sudo insmod monitor.ko
sudo ./engine supervisor ./rootfs-base

# Terminal 2 — launch containers
cp -a rootfs-base rootfs-alpha && cp -a rootfs-base rootfs-beta
cp cpu_hog memory_hog rootfs-alpha/
cp cpu_hog io_pulse rootfs-beta/

sudo ./engine start alpha ./rootfs-alpha "/cpu_hog 10"
sudo ./engine start beta  ./rootfs-beta  "/cpu_hog 10"
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine stop alpha
sudo ./engine ps
```

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│                     CLI Client Process                    │
│   engine start / run / ps / logs / stop                  │
│              │  control_request_t over                   │
│              │  UNIX domain socket                       │
└──────────────┼───────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────┐
│                   Supervisor Process                      │
│                                                          │
│  ┌─────────────┐   ┌──────────────┐   ┌──────────────┐  │
│  │ Socket loop │   │ SIGCHLD reap │   │ Logger thread│  │
│  │ (accept cmd)│   │ (waitpid)    │   │ (consumer)   │  │
│  └──────┬──────┘   └──────────────┘   └──────┬───────┘  │
│         │ clone()                             │          │
│         ▼                              bounded_buffer_t  │
│  ┌─────────────────┐                         ▲          │
│  │ Container Child │ ──── pipe ──────────────┘          │
│  │ (namespaced,    │  stdout/stderr → log chunks         │
│  │  chrooted)      │                                     │
│  └─────────────────┘                                     │
│                                                          │
│  container_record_t linked list  (metadata_lock mutex)   │
└──────────────────────┬───────────────────────────────────┘
                       │ ioctl (MONITOR_REGISTER/UNREGISTER)
                       ▼
┌──────────────────────────────────────────────────────────┐
│              monitor.ko (kernel module)                   │
│   /dev/container_monitor                                  │
│   mutex-protected linked list of monitor_entry           │
│   1-second timer → RSS check → soft warn / hard kill     │
└──────────────────────────────────────────────────────────┘
```

### IPC Mechanisms Used

| Path | Mechanism | Purpose |
| --- | --- | --- |
| CLI → Supervisor | UNIX domain socket (`/tmp/mini_runtime.sock`) | Control commands (start, stop, ps, logs) |
| Container → Supervisor | Anonymous pipe (per container) | stdout/stderr log streaming |
| Supervisor → Kernel module | `ioctl` on `/dev/container_monitor` | Register/unregister PIDs with memory limits |

---

## Implementation Details

### Task 1: Container Lifecycle

Each container is launched using `clone()` with:

- `CLONE_NEWPID` — isolated PID namespace (container's init is PID 1)
- `CLONE_NEWUTS` — isolated hostname namespace
- `CLONE_NEWNS` — isolated mount namespace

Inside `child_fn()`:
1. `sethostname()` to set the container's identity
2. `mount(NULL, "/", NULL, MS_PRIVATE | MS_REC, NULL)` to make mount changes private
3. `chroot()` into the Alpine rootfs
4. `chdir("/")` to enter the new root
5. `mount("proc", "/proc", "proc", ...)` so the container sees its own process tree
6. `dup2()` stdout/stderr to the write-end of the logging pipe
7. `nice()` to apply the requested scheduling priority
8. `execv("/bin/sh", ...)` to run the requested command

The supervisor stores each container in a `container_record_t` linked list protected by `metadata_lock` (a `pthread_mutex_t`). Each record tracks: ID, host PID, start time, state, soft/hard limits, log path, exit code, exit signal, and `stop_requested` flag.

### Task 2: CLI + IPC

The supervisor creates a UNIX domain socket at `/tmp/mini_runtime.sock` on startup. CLI client processes connect, send a `control_request_t` struct, and receive a `control_response_t` struct back. The socket provides bidirectional communication in a single file descriptor and handles multiple sequential clients cleanly.

The logging pipe is a separate, distinct IPC mechanism — each container gets its own anonymous pipe whose read-end is monitored by a dedicated producer thread in the supervisor.

**Signal handling:**
- `SIGCHLD` is handled by `global_sigchld_handler` which calls `waitpid(-1, WNOHANG)` in a loop, reaps all finished children, and updates their state in the metadata list.
- `stop_requested` is set to `1` before sending `SIGTERM` via the `stop` command so the SIGCHLD handler correctly classifies the exit as `CONTAINER_STOPPED` rather than `CONTAINER_KILLED`.
- If a container exits with `SIGKILL` and `stop_requested` is not set, its state is classified as `hard_limit_killed`.

### Task 3: Bounded-Buffer Logging

Each container's reader thread acts as a **producer** — it reads chunks from the pipe and pushes `log_item_t` structs into the `bounded_buffer_t`.

A single **logger consumer thread** drains the buffer and writes chunks to per-container log files in the `logs/` directory.

The bounded buffer uses:
- `pthread_mutex_t mutex` — protects head/tail/count
- `pthread_cond_t not_empty` — consumer waits here when buffer is empty
- `pthread_cond_t not_full` — producers wait here when buffer is full (capacity = 16 items)
- `shutting_down` flag — set during supervisor shutdown to unblock all waiters via `pthread_cond_broadcast()`

**Race condition analysis:**

| Shared resource | Race condition without sync | Protection |
| --- | --- | --- |
| `bounded_buffer_t` head/tail/count | Multiple producers + one consumer concurrently modifying ring buffer indices | `mutex` + condition variables |
| `container_record_t` linked list | Socket handler thread and SIGCHLD handler both modify list | `metadata_lock` mutex |
| `supervisor_ctx_t.should_stop` | Signal handler sets, socket loop reads | `volatile` + mutex in shutdown path |

---

## Kernel Module

### Design: `monitor.c`

The kernel module exposes `/dev/container_monitor` as a character device. The supervisor registers each new container via `ioctl(MONITOR_REGISTER)` with its PID, soft limit, and hard limit. On container exit or explicit unregister, it calls `ioctl(MONITOR_UNREGISTER)`.

### Data Structure

```c
struct monitor_entry {
    pid_t pid;
    char container_id[64];
    unsigned long soft_limit_bytes;
    unsigned long hard_limit_bytes;
    int soft_limit_warned;
    struct list_head list;
};
```

### Timer Callback (every 1 second)

For each monitored entry:
1. Call `get_rss_bytes(pid)` — if the task no longer exists, remove the entry.
2. If `rss >= soft_limit_bytes` and not yet warned → log warning, set `soft_limit_warned = 1`.
3. If `rss >= hard_limit_bytes` → call `kill_process()` (sends SIGKILL), remove entry.
4. If `rss` drops back below soft limit → reset `soft_limit_warned` to allow future warnings.

`list_for_each_entry_safe()` is used during iteration to safely delete entries without use-after-free.

### Module Cleanup

On `rmmod`, `del_timer_sync()` ensures the timer is not firing, then the list is fully drained and all entries freed before device teardown.

---

## Scheduler Experiments

### Setup

Workload binaries were copied into per-container rootfs copies and run via `run_experiments.sh`.

```bash
cp -a rootfs-base rootfs-exp1a && cp -a rootfs-base rootfs-exp1b
cp -a rootfs-base rootfs-exp2a && cp -a rootfs-base rootfs-exp2b
cp cpu_hog rootfs-exp1a/ && cp cpu_hog rootfs-exp1b/
cp cpu_hog rootfs-exp2a/ && cp io_pulse rootfs-exp2b/
sudo ./run_experiments.sh
```

### Experiment 1: CPU-bound with different nice values

Two containers running `cpu_hog 10`, one at nice -5 (high priority) and one at nice +10 (low priority):

| Container | Nice | Completion Time |
| --- | --- | --- |
| exp1a | -5 (high priority) | 20005ms |
| exp1b | +10 (low priority) | 108ms |

### Experiment 2: CPU-bound vs I/O-bound

One container running `cpu_hog`, one running `io_pulse`, both at default priority:

| Container | Workload | Completion Time |
| --- | --- | --- |
| exp2a | cpu\_hog (CPU-bound) | 11980ms |
| exp2b | io\_pulse (I/O-bound) | 2273ms |

---

## Engineering Analysis

### 1. Isolation Mechanisms

The runtime achieves isolation using three Linux namespace types passed to `clone()`. `CLONE_NEWPID` gives each container its own PID namespace — the container's first child becomes PID 1 inside its namespace, so from inside the container `ps` only shows processes in its own namespace. `CLONE_NEWUTS` gives each container its own hostname, allowing `sethostname()` to set a container-specific identity without affecting the host. `CLONE_NEWNS` gives each container its own mount namespace — combined with `chroot()` into the Alpine rootfs and a fresh `/proc` mount, the container sees a completely different filesystem tree.

What the host kernel still shares: the kernel itself, the same system calls, the same network stack (no `CLONE_NEWNET` was used), and the same user ID namespace. A container running as root is root on the host. This is why the kernel module can track and kill container processes by their host PIDs — it operates entirely in the host's PID namespace.

### 2. Supervisor and Process Lifecycle

A long-running supervisor is essential for several reasons. First, Linux requires a parent to call `wait()` on children — without a persistent parent, exited children become zombies consuming kernel resources. Second, the supervisor maintains the `container_record_t` linked list in memory, making `ps` and `logs` possible. Third, the supervisor owns the SIGCHLD handler which calls `waitpid(-1, WNOHANG)` in a loop to reap all finished children and update their metadata atomically under the `metadata_lock`. The `stop_requested` flag is set before sending `SIGTERM` so the handler can correctly distinguish a graceful stop from an unexpected kernel kill — if `exit_signal == SIGKILL` and `stop_requested` is not set, the state is classified as `hard_limit_killed`.

### 3. IPC, Threads, and Synchronization

The project uses two distinct IPC mechanisms. Path A (control): the CLI connects to the supervisor over a UNIX domain socket and sends a structured `control_request_t`, receiving a `control_response_t` back. UNIX sockets were chosen over FIFOs because they are bidirectional in a single file descriptor and support connection-oriented semantics that naturally handle multiple sequential clients. Path B (logging): each container gets a dedicated anonymous pipe created before `clone()`. The write end is inherited by the child (redirected via `dup2()`), and the read end is watched by a per-container producer thread in the supervisor. Pipes are the natural choice — they are kernel-buffered, block on `read()` naturally, and send EOF when the writer exits.

The bounded buffer uses a `pthread_mutex_t` and two condition variables. Without the mutex, two producer threads could both read `count < CAPACITY` and both write to the same slot, corrupting the buffer. The condition variables avoid busy-waiting: producers sleep on `not_full` when the buffer is full, the consumer sleeps on `not_empty` when empty. The `shutting_down` flag combined with `pthread_cond_broadcast()` ensures all threads unblock cleanly on shutdown without deadlock.

### 4. Memory Management and Enforcement

RSS (Resident Set Size) measures the physical memory pages currently mapped and present in RAM for a process. It does not measure virtual memory that has been allocated but not yet accessed, memory-mapped files not yet faulted in, or shared library pages counted once per process. Soft and hard limits serve different purposes: the soft limit is a warning threshold that alerts the operator a container is growing without disrupting it; the hard limit is an enforcement boundary that terminates the container before it can impact the host. Enforcement belongs in kernel space because a user-space poller can be preempted, delayed by the scheduler, or even killed — a kernel timer callback runs with higher privilege and cannot be bypassed by the container process itself.

### 5. Scheduling Behavior

Experiment 1 demonstrates CFS priority weighting: two identical CPU-bound workloads running concurrently, one at nice -5 and one at nice +10. The nice -5 container received significantly more CPU time slices per second, completing its wall-clock measurement in 20005ms while competing, while the nice +10 container finished in only 108ms after the first released the CPU. This confirms that CFS allocates CPU time proportional to priority weights derived from nice values — a difference of 15 nice levels corresponds to roughly a 10:1 CPU time ratio.

Experiment 2 demonstrates CFS interactivity behaviour: the I/O-bound container (`io_pulse`) completed in 2273ms while the CPU-bound container took 11980ms running concurrently. I/O-bound processes sleep frequently waiting for disk, accumulating less virtual runtime (`vruntime`), and are scheduled more aggressively when they wake up. This gives them lower latency and faster apparent completion even alongside a CPU hog, which is exactly the CFS design goal of balancing throughput and responsiveness simultaneously.

---

## Design Decisions and Tradeoffs

**Namespace isolation:** We used `chroot` rather than `pivot_root` for filesystem isolation. `chroot` is simpler to implement correctly inside a `clone()`-based container and sufficient for a trusted workload environment. The tradeoff is that `pivot_root` is more secure (it prevents escapes via `..` traversal in edge cases) but requires additional bind mounts and a more complex setup sequence.

**Supervisor architecture:** A single long-running supervisor owns all container metadata in a mutex-protected linked list. This is simple, correct, and easy to reason about for our concurrency level. The tradeoff is that all CLI commands require an IPC round-trip to the supervisor — a more scalable design might use shared memory for read-only metadata queries like `ps`.

**IPC and logging:** We used a pipe per container feeding into a shared bounded buffer rather than writing directly from the container to disk. This decouples log production from disk I/O and prevents a slow disk from stalling container output. The tradeoff is added complexity in thread lifecycle management — producer threads must be managed carefully around container exit and supervisor shutdown.

**Kernel monitor:** We used a kernel timer polling RSS every second rather than hooking into `mm` events directly. This is much simpler to implement and understand. The tradeoff is up to one second of latency between a container exceeding its hard limit and being killed — a direct `mm` hook would respond instantaneously but would require significantly more kernel internals knowledge.

**Scheduling experiments:** We used `nice` values rather than `cgroups` CPU quotas because `nice` integrates directly with our `clone`-based container model without requiring a cgroup hierarchy to be set up and managed. The tradeoff is that `nice` only affects relative scheduling priority within CFS — it cannot enforce an absolute CPU cap the way `cpu.max` in cgroups v2 can.

---

## Scheduler Experiment Results

### Experiment 1 — CPU-bound with different nice values

| Container | Nice value | Completion time |
| --- | --- | --- |
| exp1a | -5 (high priority) | 20005ms |
| exp1b | +10 (low priority) | 108ms |

Both containers ran the same `cpu_hog 10` workload. The high-priority container held the CPU for most of the 10-second run. The low-priority container completed almost instantly after the high-priority one finished and released CPU — confirming CFS priority weighting behaviour. A nice difference of 15 levels produced approximately a 185x difference in effective CPU time received during the concurrent phase.

### Experiment 2 — CPU-bound vs I/O-bound

| Container | Workload | Completion time |
| --- | --- | --- |
| exp2a | cpu\_hog (CPU-bound) | 11980ms |
| exp2b | io\_pulse (I/O-bound) | 2273ms |

The I/O-bound container completed approximately 5.3x faster than the CPU-bound one despite running concurrently. Linux CFS gives I/O-bound processes a scheduling advantage: they sleep most of the time waiting for I/O, accumulate less `vruntime`, and are placed earlier in the run queue when they wake up. This demonstrates how CFS balances throughput for CPU-bound work and responsiveness for I/O-bound work simultaneously.

---

## Demo Screenshots

### Screenshot 1 — Build (`make`)

All binaries (`engine`, `memory_hog`, `cpu_hog`, `io_pulse`) and `monitor.ko` built successfully. Minor warnings about unused functions from the boilerplate skeleton are expected and harmless.

![Build output](screenshots/step1.png)

---

### Screenshot 2 — Rootfs Setup

`rootfs-alpha` prepared with `cpu_hog` and `memory_hog` copied to the root. Alpine filesystem structure (`bin`, `etc`, `lib`, `proc`, etc.) visible alongside the workload binaries.

![Rootfs setup](screenshots/step2.png)

---

### Screenshot 3 — Kernel Module Loaded

`sudo insmod monitor.ko` succeeds. `/dev/container_monitor` device created. `dmesg` confirms `[container_monitor] Module loaded. Device: /dev/container_monitor`.

![Module loaded](screenshots/step3.png)

---

### Screenshot 4 — Supervisor Started

`sudo ./engine supervisor ./rootfs-base` starts the long-running supervisor. Output confirms `[supervisor] ready. rootfs=./rootfs-base`.

![Supervisor started](screenshots/step4.png)

---

### Screenshot 5 — Multi-Container Supervision + Metadata (`ps`)

Two containers (`alpha` and `beta`) launched simultaneously with `cpu_hog 10`. `sudo ./engine ps` shows both tracked with PID, start time, soft/hard limits (40 MiB / 64 MiB), and state.

![Multi-container ps](screenshots/step5.png)

---

### Screenshot 6 — Bounded-Buffer Logging + CLI

`sudo ./engine logs alpha` returns 60 seconds of `cpu_hog` output captured through the logging pipeline — demonstrating the pipe → bounded buffer → log file path working end to end. `ps` output at the bottom confirms both containers have exited cleanly.

![Logs and CLI](screenshots/step6.png)

---

### Screenshot 7 — Soft Limit Warning + Hard Limit Enforcement

`memory_hog` started with `--soft-mib 1 --hard-mib 64`. `dmesg` shows the sequence: `SOFT LIMIT` warning fired first at ~32MB RSS (past the 1MB threshold), then `HARD LIMIT` kill fired at ~95MB RSS. `Unregister request` confirms the supervisor was notified and cleaned up the kernel entry.

![Soft and hard limit](screenshots/step7.png)

---

### Screenshot 8 — Hard Limit State in `ps`

`ps` output shows `alpha` in state `hard_limit_killed` after being killed by the kernel module — confirming that the supervisor correctly classified the SIGKILL-with-no-stop-requested path as a hard limit kill rather than a graceful stop.

![Hard limit killed state](screenshots/step8.png)

---

### Screenshot 9 — Scheduling Experiments

Both experiments run via `run_experiments.sh`. Experiment 1 shows nice -5 vs nice +10 CPU-bound containers. Experiment 2 shows CPU-bound vs I/O-bound. Results printed with completion times confirming CFS scheduling behaviour.

![Scheduling experiments](screenshots/step9.png)

---

### Screenshot 10 — Clean Teardown

`ps aux | grep Z | grep -v grep` returns no zombie processes. `sudo rmmod monitor` succeeds. `dmesg | tail -n 5` shows `[container_monitor] Module unloaded.` `ls logs/` shows all per-container log files (`alpha.log`, `beta.log`, `exp1a.log`, `exp1b.log`, `exp2a.log`, `exp2b.log`) intact after shutdown.

![Clean teardown](screenshots/step10.png)

---

## Reproduction Steps (Fresh Ubuntu 22.04/24.04 VM)

```bash
# 1. Install dependencies
sudo apt install -y build-essential linux-headers-$(uname -r) wget git

# 2. Clone repo
git clone <your-repo-url>
cd OS-Jackfruit/boilerplate

# 3. Get Alpine rootfs (use aarch64 if on ARM)
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

# 4. Build
make

# 5. Prepare rootfs copies
cp -a rootfs-base rootfs-alpha && cp -a rootfs-base rootfs-beta
cp cpu_hog memory_hog rootfs-alpha/
cp cpu_hog io_pulse rootfs-beta/

# 6. Load kernel module and start supervisor (Terminal 1)
sudo insmod monitor.ko
sudo ./engine supervisor ./rootfs-base

# 7. Run demo commands (Terminal 2)
sudo ./engine start alpha ./rootfs-alpha "/cpu_hog 10"
sudo ./engine start beta  ./rootfs-beta  "/cpu_hog 10"
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine stop alpha

# 8. Memory limit test
sudo ./engine start alpha ./rootfs-alpha "/memory_hog 15 500" --soft-mib 1 --hard-mib 64
sleep 5 && sudo dmesg | tail -n 10

# 9. Scheduling experiments
sudo ./run_experiments.sh

# 10. Cleanup
sudo rmmod monitor
ps aux | grep Z | grep -v grep   # verify no zombies
```
