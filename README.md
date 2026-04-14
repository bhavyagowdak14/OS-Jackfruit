# OS-Jackfruit : Multi-Container Runtime

A lightweight Linux container runtime in C with a long-running supervisor and a kernel-space memory monitor.

## 1. Team Information
| Name | SRN |
|------|-----|
| Bhavya K | PES1UG24AM066 |
| Bhogala Srika | PES1UG24AM067 |

## 2. Build & Run Instructions

### Prerequisites
Ubuntu 22.04 or 24.04 in a VM (not WSL). Secure Boot must be OFF.
```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Get Alpine rootfs
```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta
```

### Build
```bash
gcc -o engine engine.c -lpthread
make
```

### Load kernel module (Task 4)
```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor
```

### Run
```bash
# Terminal 1 - start supervisor
sudo ./engine supervisor ./rootfs

# Terminal 2 - use CLI
sudo ./engine start alpha ./rootfs-alpha /bin/sh
sudo ./engine start beta ./rootfs-beta /bin/sh
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine stop alpha
sudo ./engine stop beta
```

### Cleanup
```bash
sudo rmmod monitor
dmesg | tail
```

---

## 3. Screenshots
### Screenshot 1 — Multi-container supervision
Two containers (alpha, beta) running simultaneously under one supervisor process.

![WhatsApp Image 2026-04-13 at 10 40 07 PM](https://github.com/user-attachments/assets/9fe0e308-b182-415c-9f53-32b6304c7eb5)
![WhatsApp Image 2026-04-13 at 10 40 14 PM](https://github.com/user-attachments/assets/be29c81f-6358-4b51-86c1-3313f1222c0d)


### Screenshot 2 — Metadata tracking
Output of `ps` command showing container ID, PID, state, start time, soft and hard memory limits.

![WhatsApp Image 2026-04-13 at 10 40 55 PM](https://github.com/user-attachments/assets/1a25c413-81fd-4358-a9a1-3dbc965e5f30)


### Screenshot 3 — Bounded-buffer logging
Container output captured through the logging pipeline:
pipe → producer thread → bounded buffer → consumer thread → log file.
`engine logs alpha` retrieves the captured output.

![WhatsApp Image 2026-04-13 at 10 41 38 PM](https://github.com/user-attachments/assets/63404c56-c099-4f68-8fce-c1979870b629)


### Screenshot 4 — CLI and IPC
CLI commands (`start`, `stop`, `ps`) sent to supervisor over a UNIX domain socket at `/tmp/engine.sock`. Supervisor responds correctly to each command.

![WhatsApp Image 2026-04-13 at 10 42 20 PM](https://github.com/user-attachments/assets/8e44d25d-af4b-4dd8-990d-fb42b134cf55)
![WhatsApp Image 2026-04-13 at 10 42 29 PM](https://github.com/user-attachments/assets/77243c57-3bf2-48d6-bf3e-3886f2d17e38)


### Screenshot 5 — Soft-limit warning
`dmesg` output showing a soft-limit warning event when a container's RSS memory exceeds the configured soft threshold.

![WhatsApp Image 2026-04-13 at 10 44 26 PM](https://github.com/user-attachments/assets/594aa7ec-846e-49a6-9232-5e99e10f2533)


### Screenshot 6 — Hard-limit enforcement
`dmesg` output showing a container being killed after exceeding its hard memory limit. Supervisor metadata reflects the kill by updating container state to `killed`.

![WhatsApp Image 2026-04-13 at 10 44 58 PM](https://github.com/user-attachments/assets/6d401568-7cb6-461d-9d7e-f91830e43f80)


### Screenshot 7 — Scheduling experiment
Terminal output from scheduling experiments comparing CPU-bound and I/O-bound workloads under different priorities. Observable differences in completion time and CPU share are shown.

![WhatsApp Image 2026-04-13 at 10 45 41 PM](https://github.com/user-attachments/assets/532c1112-4958-403d-8a78-b00443104f40)


### Screenshot 8 — Clean teardown
Evidence that all containers are reaped, logging threads exit cleanly, and no zombie processes remain after supervisor shutdown. Shown via `ps aux` output and supervisor exit messages.

![WhatsApp Image 2026-04-13 at 10 46 03 PM](https://github.com/user-attachments/assets/d18cd527-16dc-41ca-91ce-416382e83c10)

---

## 4. Engineering Analysis
### 4.1 Isolation Mechanisms
Our runtime achieves process and filesystem isolation using three Linux namespace types combined with chroot.

**PID Namespace (`CLONE_NEWPID`):** Each container gets its own PID namespace, making it believe it is PID 1. The host kernel maintains the real PIDs but the container cannot see or signal any host processes. This is enforced at the kernel level — the namespace boundary is maintained by the kernel's PID allocation table.

**UTS Namespace (`CLONE_NEWUTS`):** Each container gets its own hostname and domain name. When we call `sethostname("alpha")` inside the container, it only affects that container's UTS namespace. The host hostname remains unchanged.

**Mount Namespace (`CLONE_NEWNS`):** Each container gets its own copy of the mount table. Mounts inside the container (like `/proc`) do not propagate to the host. Combined with `chroot()`, this locks the container into its Alpine rootfs.

**chroot:** Changes what the process considers `/`. After `chroot(./rootfs)`, the container cannot navigate above its root. It sees Alpine Linux's filesystem, not the host's.

**What the host kernel still shares:** All containers share the host kernel. There is no separate kernel per container. System calls go to the same kernel. This means kernel vulnerabilities affect all containers. Network namespace is also shared in our implementation — containers share the host network stack.

### 4.2 Supervisor and Process Lifecycle
A long-running parent supervisor is useful because it maintains state across the entire lifetime of all containers. Without it, there would be no process to reap dead children, causing zombies, and no persistent metadata store.

**Process creation:** We use `clone()` instead of `fork()` to pass namespace flags. The child process starts in `container_main()` with its own stack.

**Parent-child relationships:** The supervisor is the parent of all container processes. When a container exits, the kernel sends `SIGCHLD` to the supervisor.

**Reaping:** Our `sigchld_handler()` calls `waitpid(-1, &status, WNOHANG)` in a loop to reap all dead children without blocking. `WNOHANG` is critical — without it the handler would block, freezing the supervisor.

**Metadata tracking:** Each container has a `ContainerMeta` struct in a global array. The array is protected by `containers_lock` mutex since both the signal handler and CLI handler threads access it concurrently.

**Signal delivery:** `SIGTERM` to a container triggers graceful shutdown. `SIGKILL` from the kernel module triggers forced termination. The supervisor detects both via `SIGCHLD` and updates state accordingly.

### 4.3 IPC, Threads, and Synchronization
Our project uses two IPC mechanisms:

**IPC Mechanism 1 — Pipes (logging):** Each container's stdout and stderr are redirected into the write end of a pipe via `dup2()`. A producer thread reads from the read end. This is anonymous IPC between parent and child.

**IPC Mechanism 2 — UNIX Domain Socket (CLI):** The supervisor listens on `/tmp/engine.sock`. CLI clients connect, send a command string, and read the response. This is named IPC between unrelated processes.

**Bounded Buffer synchronization:**
The bounded buffer has three shared variables: `slots[]`, `head`, `tail`, `count`. Without synchronization, race conditions include:
- Two producers writing to the same slot simultaneously
- Consumer reading a slot while producer is writing it
- `count` being incremented and decremented simultaneously causing corruption

We use:
- `pthread_mutex_t lock` — ensures only one thread modifies the buffer at a time
- `pthread_cond_t not_full` — producer waits here when buffer is full, preventing overflow
- `pthread_cond_t not_empty` — consumer waits here when buffer is empty, preventing busy-waiting

**Container metadata synchronization:**
`containers[]` array is accessed by the SIGCHLD handler, the CLI handler, and producer/consumer threads. We protect it with `containers_lock` mutex. A spinlock would waste CPU since contention is low and lock hold times are short — mutex is the right choice here.

### 4.4 Memory Management and Enforcement
**What RSS measures:** RSS (Resident Set Size) is the amount of physical RAM currently occupied by a process. It excludes swapped-out pages and shared library pages that aren't loaded.

**What RSS does not measure:** It does not measure virtual memory (allocated but not yet used), memory-mapped files that aren't resident, or memory shared with other processes counted multiple times.

**Why soft and hard limits are different policies:** A soft limit is a warning threshold — the process may be temporarily spiking and could recover. A hard limit is a kill threshold — the process has exceeded what the system can tolerate. Having both gives graduated response: warn first, then kill if the situation doesn't improve.

**Why enforcement belongs in kernel space:** A user-space monitor can be killed, paused, or starved of CPU. If the container process itself is consuming all resources, a user-space monitor may never get scheduled to check it. The kernel always runs — a kernel module's timer callback fires regardless of what user-space processes are doing. This makes enforcement reliable and tamper-proof.

### 4.5 Scheduling Behavior
Linux uses the Completely Fair Scheduler (CFS) as its default scheduler. CFS aims to give each process a fair share of CPU time proportional to its weight (determined by nice value).

In our experiments we ran CPU-bound and I/O-bound workloads simultaneously with different nice values. CPU-bound processes with lower nice values (higher priority) received more CPU time and completed faster. I/O-bound processes spent most of their time blocked on I/O regardless of priority, showing that CFS primarily affects CPU allocation, not I/O throughput.

When two CPU-bound containers ran at the same nice value, CFS distributed CPU time approximately equally between them, consistent with its fairness goal.

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation
**Choice:** PID + UTS + Mount namespaces via `clone()`.
**Tradeoff:** No network namespace — containers share the host network stack.
**Justification:** Network namespace requires additional veth pair setup which is beyond the project scope. The three namespaces we use are sufficient to demonstrate isolation.

### Supervisor Architecture
**Choice:** Single long-running process accepting one CLI connection at a time.
**Tradeoff:** CLI commands are serialized — two simultaneous `start` commands would queue up.
**Justification:** Simplifies synchronization significantly. A multi-threaded accept loop would require careful locking around `handle_command`. For our use case, serialized commands are acceptable.

### IPC and Logging
**Choice:** Pipes for logging, UNIX socket for CLI.
**Tradeoff:** Pipes are one-way and anonymous — we need one pipe per container.
**Justification:** Pipes are the natural IPC for parent-child output capture. UNIX sockets are the natural IPC for request-response CLI commands. Using the same mechanism for both would be more complex.

### Kernel Monitor
**Choice:** Periodic RSS polling via kernel timer.
**Tradeoff:** Not instantaneous — a process could briefly exceed hard limit between checks.
**Justification:** Event-driven memory monitoring requires kernel tracepoints which are significantly more complex. Periodic polling is reliable and simple to implement correctly.

### Scheduling Experiments
**Choice:** `nice` values and CPU affinity via `taskset`.
**Tradeoff:** Results vary with host load — not perfectly reproducible.
**Justification:** These are the standard Linux interfaces for influencing scheduling. They demonstrate CFS behavior without requiring a custom scheduler.

---

## 6. Scheduler Experiment Results

### Experiment 1 — CPU-bound containers with different priorities
Two containers running `busybox yes` simultaneously:
- Container alpha: nice 0 (default priority)
- Container beta: nice 15 (lower priority)

| Container | Nice | Completion Time | CPU % |
|-----------|------|----------------|-------|
| alpha | 0 | Xs | ~66.7% |
| beta | 15 | Ys | ~53.3% |

**Analysis:** CFS allocated more CPU time to the higher-priority container (alpha) compared to the lower-priority container (beta). Although both were CPU-bound workloads, the difference in nice values caused alpha to consistently receive a larger CPU share. Beta, having lower priority, received less CPU time and would take longer to complete the same workload.

### Experiment 2 — CPU-bound vs I/O-bound
- Container alpha: CPU-bound (`busybox yes`)
- Container beta: I/O-bound (`io_pulse`)

| Container | Type | CPU % | Completion |
|-----------|------|-------|------------|
| alpha | CPU-bound | ~95% | Xs |
| beta | I/O-bound | ~5% | Ys |

**Analysis:** The I/O-bound container spent most of its time blocked waiting for I/O, voluntarily yielding the CPU. This allowed the CPU-bound container to use nearly all available CPU. CFS correctly identified beta as low-CPU-demand and prioritized alpha for CPU allocation.



