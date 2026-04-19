# Multi-Container Runtime with Supervisor and Kernel Memory Monitor

A lightweight Linux container runtime in C that supports multiple concurrent containers under a long-running supervisor, bounded-buffer logging, a CLI control plane, kernel-space memory monitoring with soft and hard limits, and scheduling experiments on Linux.

---

## Team Information

- **Member 1:** <br />
Name : S Mayur <br />
SRN : PES1UG24CS391 <br />
- **Member 2:** <br />
Name : S S Adhithya Sriram <br />
SRN : PES1UG24CS393 <br />

---

## Project Overview

This project implements a small Linux container runtime with two tightly integrated components:

1. **User-Space Runtime + Supervisor (`engine.c`)**
   - Starts and manages multiple containers concurrently
   - Maintains per-container metadata
   - Exposes a CLI for lifecycle operations
   - Captures container `stdout` and `stderr`
   - Uses a bounded-buffer producer-consumer logging pipeline
   - Reaps children correctly and avoids zombies

2. **Kernel-Space Memory Monitor (`monitor.c`)**
   - Exposes `/dev/container_monitor`
   - Accepts container host PIDs from user space via `ioctl`
   - Tracks monitored tasks in kernel space
   - Enforces **soft** and **hard** memory limits
   - Logs a warning on soft-limit crossing
   - Kills the container on hard-limit crossing

The runtime is designed for **Ubuntu 22.04/24.04 running inside a VM** with Secure Boot disabled.

---

## Features

- Multi-container supervision under one parent supervisor
- PID, UTS, and mount namespace isolation
- Per-container writable root filesystem
- `/proc` mounted inside containers
- Supervisor CLI using a dedicated IPC control path
- Pipe-based log capture from container `stdout`/`stderr`
- Bounded-buffer logging with synchronization
- Persistent per-container log files
- Soft and hard memory enforcement through a Linux kernel module
- Controlled scheduling experiments using CPU-bound and I/O-bound workloads
- Clean teardown with child reaping, thread shutdown, and resource cleanup

---

## Repository Structure

```bash
.
├── Output Screenshots/
├── boilerplate/
    ├── Makefile
    ├── cpu_hog.c
    ├── engine.c
    ├── environment_check.sh
    ├── io_pulse.c
    ├── memory_hog.c
    ├── monitor.c
    ├── monitor_ioctl.h        
├── .gitignore          
├── project_guide.md          
└── README.md
```

---

## Environment

This project is intended to run in:

* **Ubuntu 22.04 or 24.04**
* **Inside a VM**
* **Secure Boot OFF**
* **No WSL**

Recommended setup used for this project:

* Ubuntu VM on UTM / VirtualBox
* Linux headers installed for the running kernel
* Root access available for `insmod`, `rmmod`, namespace operations, and `ioctl`

---

## Dependencies

Install the required build tools and kernel headers:

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

---

## Build Instructions

Build all required binaries and the kernel module:

```bash
make
```

Expected outputs include:

* `engine`
* `monitor.ko`
* workload/test binaries such as:

  * `cpu_hog`
  * `io_pulse`
  * `memory_hog`

---

## Root Filesystem Setup

Prepare the base root filesystem and create one writable copy per container.

### 1. Create the base Alpine rootfs

```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
```

### 2. Create writable copies for containers

```bash
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```

### 3. Copy workload binaries if needed

If helper binaries must run *inside* a container, copy them into the container filesystem before launch:

```bash
cp cpu_hog ./rootfs-alpha/
cp io_pulse ./rootfs-alpha/
cp memory_hog ./rootfs-alpha/

cp cpu_hog ./rootfs-beta/
cp io_pulse ./rootfs-beta/
cp memory_hog ./rootfs-beta/
```

> `rootfs-base/` and per-container `rootfs-*` directories are local runtime artifacts and need not be pushed to GitHub.

---

## CLI Contract

The runtime supports the following command interface:

```bash
engine supervisor <base-rootfs>
engine start <id> <container-rootfs> <command> [--soft-mib N] [--hard-mib N] [--nice N]
engine run   <id> <container-rootfs> <command> [--soft-mib N] [--hard-mib N] [--nice N]
engine ps
engine logs <id>
engine stop <id>
```

### Semantics

* `supervisor` starts the long-running parent process
* `start` launches a background container and returns once metadata is recorded
* `run` launches a container and waits until it exits
* `ps` prints tracked container metadata
* `logs` shows the corresponding log file
* `stop` requests clean termination of a running container

Default limits if not provided:

* **Soft limit:** 40 MiB
* **Hard limit:** 64 MiB

Each running container must have its **own writable rootfs copy**.

---

## How the Runtime Works

The project uses one binary, `engine`, in two roles:

### 1. Supervisor Daemon

Started once using:

```bash
sudo ./engine supervisor ./rootfs-base
```

This process:

* stays alive
* tracks all containers
* owns metadata
* owns the logging pipeline
* receives CLI control commands

### 2. CLI Client

Each command such as:

```bash
sudo ./engine start alpha ./rootfs-alpha /bin/sh
```

acts as a short-lived client process that:

* connects to the supervisor through the control IPC channel
* sends a request
* receives the response
* exits

---

## IPC Design

This project uses **two distinct IPC paths**, as required.

### Path A: Logging IPC

**Container → Supervisor**

* Container `stdout` and `stderr` are redirected into pipes
* Producer thread(s) read from these pipes
* Data is pushed into a bounded shared buffer
* Consumer thread(s) remove entries and write them to per-container log files

### Path B: Control IPC

**CLI Client → Supervisor**

* Used for commands such as `start`, `run`, `ps`, `logs`, and `stop`
* Implemented separately from the pipe-based logging path
* Allows the supervisor to remain long-running while CLI invocations stay short-lived

This separation keeps lifecycle control independent from log transport and simplifies reasoning about correctness.

---

## Typical Run Sequence

### 1. Build

```bash
make
```

### 2. Load the kernel module

```bash
sudo insmod monitor.ko
```

### 3. Verify the control device

```bash
ls -l /dev/container_monitor
```

### 4. Start the supervisor

```bash
sudo ./engine supervisor ./rootfs-base
```

### 5. Create writable rootfs copies

```bash
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```

### 6. Start containers

```bash
sudo ./engine start alpha ./rootfs-alpha /bin/sh --soft-mib 48 --hard-mib 80
sudo ./engine start beta ./rootfs-beta /bin/sh --soft-mib 64 --hard-mib 96
```

### 7. Inspect metadata

```bash
sudo ./engine ps
```

### 8. Inspect logs

```bash
sudo ./engine logs alpha
```

### 9. Stop containers

```bash
sudo ./engine stop alpha
sudo ./engine stop beta
```

### 10. Check kernel messages

```bash
dmesg | tail
```

### 11. Unload the module

```bash
sudo rmmod monitor
```

---

## Demonstration Commands and Screenshots

This section documents the commands used to generate the required screenshots for evaluation.

---

### 1. Multi-Container Supervision

Start the supervisor:

```bash
sudo ./engine supervisor ./rootfs-base
```

In another terminal:

```bash
sudo ./engine start alpha ./rootfs-alpha /bin/sh
sudo ./engine start beta ./rootfs-beta /bin/sh
```

<img width="1512" height="982" alt="Multi-Container Supervision" src="https://github.com/user-attachments/assets/bdbadc0b-267f-49c2-bbdc-594d93af1199" />

* *Single supervisor process managing multiple concurrent containers.*

---

### 2. Metadata Tracking

```bash
sudo ./engine ps
```
<img width="1512" height="982" alt="Metadata Tracking" src="https://github.com/user-attachments/assets/5fa2ace7-47c9-4468-ba77-f03563e5693b" />

* *Supervisor metadata table showing tracked containers and runtime state.*

---

### 3. Bounded-Buffer Logging

Run a workload that produces output:

```bash
./cpu_hog &
```

Then inspect logs:

```bash
cat logs/alpha.log
```

or:

```bash
tail -f logs/alpha.log
```

<img width="1512" height="982" alt="Bounded-Buffer Logging" src="https://github.com/user-attachments/assets/5cb039c7-00da-492a-b8ab-dbe62d32bd4e" />

* *Container output captured through the bounded-buffer logging pipeline and written to a persistent log file.*

---

### 4. CLI and IPC

Use any CLI command that demonstrates control-path communication, for example:

```bash
sudo ./engine ps
```

or:

```bash
sudo ./engine start gamma ./rootfs-alpha /bin/sh
```

<img width="1512" height="982" alt="CLI and IPC" src="https://github.com/user-attachments/assets/f9475b0c-8b2f-45ad-b005-a3a5c4787644" />

* *CLI request sent over the control IPC channel and processed by the long-running supervisor.*

---

### 5. Soft-Limit Warning

Run the memory workload:

```bash
./memory_hog
```

Then inspect kernel logs:

```bash
sudo dmesg | tail
```

<img width="1512" height="982" alt="Soft-Limit Warning" src="https://github.com/user-attachments/assets/44e99aaf-0802-4fec-8291-2d5599fd46b7" />

* *Kernel monitor warning when a container first exceeds its configured soft RSS limit.*

---

### 6. Hard-Limit Enforcement

Continue the memory workload until the process is terminated, then inspect kernel logs:

```bash
sudo dmesg | tail
```

<img width="1512" height="982" alt="Soft-Limit Killed" src="https://github.com/user-attachments/assets/55a13b83-7c63-4aa2-b193-ecfad11681b1" />

* *Kernel monitor hard-limit enforcement terminating a container that exceeds its maximum RSS limit.*

---

### 7. Scheduling Experiment

Run both workloads:

```bash
./cpu_hog &
./io_pulse &
top
```

<img width="1512" height="982" alt="Scheduling Experiement-1" src="https://github.com/user-attachments/assets/2a1f96ea-c577-4325-8ed3-378c9bb9fe42" />
<img width="1512" height="982" alt="Scheduling Experiement-2" src="https://github.com/user-attachments/assets/fd4726d8-9ece-496e-9d38-c4703e73adeb" />

* *Scheduling experiment comparing CPU-bound and I/O-bound workload behavior under Linux scheduling.*

---

### 8. Clean Teardown

Stop containers and clean helper processes:

```bash
sudo ./engine stop alpha
sudo ./engine stop beta

pkill cpu_hog
pkill io_pulse

ps aux | grep engine
```

<img width="1512" height="982" alt="Clean Teardown" src="https://github.com/user-attachments/assets/b4f53abf-9215-4747-bc91-3b9ec8c099e7" />

* *Clean teardown after stopping containers, with no lingering zombies or stale runtime processes.*

---

## Engineering Analysis

This section explains the OS concepts exercised by the implementation.

---

### 1. Isolation Mechanisms

The runtime achieves isolation primarily through Linux namespaces and a separate root filesystem for each container.

* **PID namespace** isolates process numbering inside the container. A process may appear as PID 1 inside the container while still having a different host PID externally.
* **UTS namespace** isolates the hostname and related system identity fields.
* **Mount namespace** isolates the visible mount table so that container-specific mounts such as `/proc` do not alter the host's mount layout.
* **`chroot` or `pivot_root`** changes the container's visible filesystem root to its assigned `container-rootfs`.

This creates a restricted process and filesystem view for each container. However, the host kernel is still shared across all containers. Containers are not full virtual machines; they isolate views of kernel-managed resources while still relying on the same Linux kernel for scheduling, memory management, system calls, and device handling.

The separate writable rootfs copy per container also prevents interference between live containers. If two running containers shared the same writable rootfs, changes from one would immediately affect the other and break filesystem isolation semantics.

---

### 2. Supervisor and Process Lifecycle

A long-running parent supervisor is useful because container execution is not just about launching a child process. The runtime must also:

* track metadata
* reap exited children
* coordinate logs
* classify termination reasons
* respond to CLI requests over time

When the supervisor creates a container using `clone()`, the child becomes part of a managed lifecycle. The supervisor retains the host PID, records the start time, and tracks the container state. Later, when the child exits, the supervisor handles `SIGCHLD`, reaps it using `waitpid`, and updates final metadata such as exit code, signal, or kill reason.

Without a persistent parent process, there is no clean place to centralize lifecycle state, logging ownership, or child reaping. That would lead to zombie risk, inconsistent metadata, and fragmented control flow.

Signal handling is also more coherent with a supervisor design. Manual stop requests can be distinguished from kernel-enforced kills by setting an internal `stop_requested` flag before signaling the container. This makes final attribution reliable in `ps` output.

---

### 3. IPC, Threads, and Synchronization

This project uses two IPC mechanisms for two different purposes:

#### Control path

A dedicated control IPC channel is used between the CLI client and the long-running supervisor. This allows each CLI invocation to be short-lived while the supervisor persists in the background.

#### Logging path

Container `stdout` and `stderr` flow to the supervisor through pipes. Producer threads read these file descriptors and insert log entries into a bounded shared buffer. Consumer threads remove entries from the buffer and write them to the correct log file.

#### Why synchronization is required

Several race conditions exist without synchronization:

* Two producers writing to the same bounded buffer could corrupt head/tail indices
* A consumer could read partially written data
* Metadata could be read while another thread is updating process state
* Shutdown could occur while producers or consumers are still active, causing lost log entries or dangling file descriptors

#### Synchronization choice

A standard producer-consumer design using:

* **Mutexes** to protect shared buffer state and shared metadata
* **Condition variables** to block producers when the buffer is full and block consumers when it is empty

This design is appropriate because it avoids busy-waiting, preserves correctness under concurrency, and provides a clean shutdown path when threads need to drain remaining entries before exit.

The bounded-buffer design also prevents unbounded memory growth. Instead of allowing logging to allocate indefinitely, it imposes backpressure while still preserving correctness.

---

### 4. Memory Management and Enforcement

The kernel monitor enforces memory limits using **RSS** (Resident Set Size), which measures the portion of a process's memory currently resident in physical RAM. RSS is useful because it reflects real memory pressure more directly than virtual address space size alone.

However, RSS does **not** measure everything:

* it does not equal total virtual memory
* it does not directly represent swapped-out pages
* it may not perfectly capture shared-memory attribution in a simple way

This is why RSS is a practical but not perfect enforcement metric.

The distinction between **soft** and **hard** limits is policy-driven:

* A **soft limit** is a warning threshold. It signals that the container is consuming more memory than expected, but still allows execution to continue.
* A **hard limit** is a safety boundary. Once crossed, the process is terminated to protect overall system stability and enforce the configured cap.

This enforcement belongs in **kernel space** rather than only user space because the kernel has authoritative visibility into process memory state and can act reliably even if the process is misbehaving, looping, or ignoring user-space checks. A user-space-only monitor would be less trustworthy because it depends on scheduling, responsiveness, and cooperative behavior from the very system it is trying to control.

---

### 5. Scheduling Behavior

The scheduling experiment compares workloads with different execution characteristics:

* **CPU-bound workload**: `cpu_hog`
* **I/O-bound or sleep-heavy workload**: `io_pulse`

The CPU-bound task tends to consume a high percentage of CPU time because it remains runnable almost continuously. The I/O-bound workload yields more often, sleeps frequently, and therefore consumes less CPU while often remaining responsive.

This reflects Linux scheduling goals:

* **Fairness:** runnable tasks compete for CPU time based on scheduling policy and priority
* **Responsiveness:** tasks that sleep and wake frequently often get scheduled quickly so interactive or I/O-driven behavior remains responsive
* **Throughput:** CPU-intensive tasks can still make steady forward progress when there is available compute capacity

If two CPU-bound tasks are run with different `nice` values, the scheduler should bias CPU allocation toward the higher-priority process. If a CPU-bound and I/O-bound task run together, the CPU-bound process typically dominates raw CPU percentage while the I/O-bound task retains good responsiveness due to its sleep/wake pattern.

---

## Design Decisions and Tradeoffs

---

### 1. Namespace Isolation

**Design choice:** PID, UTS, and mount namespaces with a per-container rootfs.

**Tradeoff:** This is lighter than full virtualization, but it still shares the host kernel.

**Justification:** The assignment explicitly targets Linux container mechanisms rather than full VMs, so namespace-based isolation is the correct abstraction.

---

### 2. Long-Running Supervisor

**Design choice:** One persistent parent process owns metadata, lifecycle, logging, and control handling.

**Tradeoff:** Centralization increases implementation complexity and requires careful synchronization.

**Justification:** It is the cleanest way to support multiple concurrent containers, structured CLI interaction, and correct child reaping.

---

### 3. Separate IPC Paths

**Design choice:** One IPC mechanism for control, another for logging.

**Tradeoff:** More moving parts than a single unified channel.

**Justification:** Logs and control traffic have different semantics. Separation reduces coupling and makes debugging and reasoning about correctness easier.

---

### 4. Bounded-Buffer Logging

**Design choice:** Producer-consumer logging with a fixed-capacity shared buffer.

**Tradeoff:** Producers may block under heavy output bursts.

**Justification:** This prevents unbounded memory use while still preserving correctness and persistent logging.

---

### 5. Kernel-Space Memory Enforcement

**Design choice:** Soft warnings and hard kills implemented in a Linux kernel module.

**Tradeoff:** Kernel code is harder to debug and requires exact version-compatible headers.

**Justification:** Enforcement must be authoritative and reliable, which is best achieved in kernel space.

---

### 6. Scheduling Experiments

**Design choice:** Compare CPU-bound and I/O-bound workloads and observe behavior using standard Linux tools.

**Tradeoff:** Results are partly workload- and system-dependent.

**Justification:** The goal is not a mathematically perfect benchmark but a concrete experimental connection between runtime behavior and Linux scheduling policy.

---

## Scheduler Experiment Results

### Workloads Used

* `cpu_hog` — tight compute loop, CPU-bound
* `io_pulse` — periodic sleep / output behavior, I/O-bound
* Optionally different `nice` settings for priority comparison

### Commands

```bash
./cpu_hog &
./io_pulse &
top
```

### Example Observations

| Workload | Behavior | CPU Usage | Scheduler Interpretation |
|---|---|---|---|
| `cpu_hog` | Continuously runnable | High | Receives substantial CPU share because it is always ready to run |
| `io_pulse` | Sleeps periodically | Low to moderate | Uses less CPU, wakes periodically, remains responsive |
| Two CPU-bound tasks with different nice values | Both runnable | Higher share to lower nice task | Scheduler biases CPU time according to priority |

### Analysis

The experiment shows that Linux scheduling attempts to balance fairness with responsiveness. CPU-bound tasks naturally consume more compute time because they are continuously runnable. I/O-bound tasks tend to use less CPU because they frequently sleep, but they can still feel responsive because the scheduler handles wakeups quickly. When priority is varied using `nice`, Linux biases CPU allocation rather than enforcing a rigid fixed percentage.

> Replace the table values with your actual observed results if you measured specific percentages or runtimes.

---

## Cleanup and Teardown

To stop containers and clean up:

```bash
sudo ./engine stop alpha
sudo ./engine stop beta
pkill cpu_hog
pkill io_pulse
dmesg | tail
sudo rmmod monitor
```

Validation steps:

* No unreaped children
* No zombie container processes
* Logging threads exit cleanly
* File descriptors closed
* Kernel monitor entries removed on unload
* No stale metadata left in supervisor state

---

## Notes

* Each running container must use its own writable rootfs copy.
* Do not share the same writable rootfs between live containers.
* Helper binaries should be copied into a container's rootfs if they must execute inside that container.
* The project is intended for Linux VMs, not macOS or WSL.

---

## Conclusion

This project demonstrates the core OS concepts behind Linux containers: namespace-based isolation, supervised process lifecycle management, IPC design, concurrent logging with synchronization, kernel-mediated memory enforcement, and measurable scheduling behavior. Together, these components form a compact but realistic container runtime that exposes how Linux manages process isolation, resource control, and system responsiveness.
