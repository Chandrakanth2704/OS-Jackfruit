
## 1. Team Information

| Name               | SRN                |
| ------------------ | ------------------ |
| Chandrakanth V G   |      PES1UG25CS810 |
| Deekshith Gowda G  |      PES1UG24CS142 |

---

## 2. Build, Load, and Run Instructions

The following steps reproduce the complete setup on a fresh Ubuntu 22.04/24.04 VM.

---

### Step 1: Build the Project

```bash id="u4xq1a"
make
```

This compiles:

* engine (user-space runtime + supervisor)
* monitor.ko (kernel module)
* cpu_hog, memory_hog, io_pulse workloads

---

### Step 2: Load Kernel Module

```bash id="jv3w1q"
sudo insmod monitor.ko
```

---

### Step 3: Verify Control Device

```bash id="9d0v8y"
ls -l /dev/container_monitor
```

If not present:

```bash id="0p7h2x"
sudo mknod /dev/container_monitor c 239 0
sudo chmod 666 /dev/container_monitor
```

---

### Step 4: Start Supervisor

```bash id="q3k8m1"
sudo ./engine supervisor ./rootfs-base
```

The supervisor is a long-running process that manages containers, handles CLI requests, and coordinates logging.

---

### Step 5: Create Per-Container Root Filesystems

```bash id="s6l9z0"
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```

Each container uses its own writable root filesystem.

---

### Step 6: Open a New Terminal

Run CLI commands in a separate terminal while the supervisor is running.

---

### Step 7: Start Containers

```bash id="g2t5n7"
sudo ./engine start alpha ./rootfs-alpha /bin/sh --soft-mib 48 --hard-mib 80
sudo ./engine start beta ./rootfs-beta /bin/sh --soft-mib 64 --hard-mib 96
```

---

### Step 8: List Containers

```bash id="k8c2r1"
sudo ./engine ps
```

---

### Step 9: Inspect One Container

```bash id="z7y1m3"
sudo ./engine logs alpha
```

---

### Step 10: Run Workloads Inside Container

```bash id="t5m0q8"
cp cpu_hog ./rootfs-alpha/
cp memory_hog ./rootfs-alpha/
```

Run memory test:

```bash id="f1b9e6"
sudo ./engine start memhog ./rootfs-alpha /memory_hog --soft-mib 32 --hard-mib 64
```

---

### Step 11: Run Scheduling Experiment

```bash id="w3r4p2"
./cpu_hog &
./io_pulse &
```

---

### Step 12: Stop Containers

```bash id="c9d6l1"
sudo ./engine stop alpha
sudo ./engine stop beta
```

---

### Step 13: Stop Supervisor

Press:

```
Ctrl + C
```

---

### Step 14: Inspect Kernel Logs

```bash id="n2v8x5"
sudo dmesg | tail
```

---

### Step 15: Unload Kernel Module

```bash id="p0k7u3"
sudo rmmod monitor
```

---

## 3. Demo with Screenshots

---

### 1. Multi-container Supervision

![img](./screenshots/Screenshot1.png)

This screenshot shows two containers (alpha and beta) successfully started and listed using the `engine ps` command, confirming that both are running simultaneously under a single supervisor process.

---

### 2. Metadata Tracking

![img](./screenshots/Screenshot2.png)

The `engine ps` output displays container metadata including PID, state, uptime, and memory limits, confirming that the supervisor is correctly tracking all running containers.

---

### 3. Bounded-buffer Logging

![img](./screenshots/Screenshot3.png)

This screenshot shows logs retrieved using the `engine logs` command, demonstrating that container output is captured and stored through the bounded-buffer logging pipeline with proper synchronization.

---

### 4. CLI and IPC

![img](./screenshots/Screenshot4.png)

This screenshot demonstrates a CLI command (`engine start gamma`) interacting with the supervisor, confirming successful inter-process communication through the control channel.

---

### 5. Soft-limit Warning

![img](./screenshots/Screenshot5.png)

Kernel logs show a soft-limit warning when memory usage exceeds the configured threshold, while the container continues execution.

---

### 6. Hard-limit Enforcement

![img](./screenshots/Screenshot6.png)

The kernel terminates the container after exceeding the hard memory limit, and logs confirm that the container was killed and enforcement was applied.

---

### 7. Scheduling Experiment

![img](./screenshots/Screenshot7.png)

CPU-bound (`cpu_hog`) and I/O-bound (`io_pulse`) workloads run concurrently, demonstrating how the scheduler distributes CPU time based on workload characteristics.

---

### 8. Clean Teardown

![img](./screenshots/Screenshot8.png)

The absence of defunct processes confirms that all containers were properly terminated and no zombie processes remain.

---

## 4. Engineering Analysis

### Isolation Mechanisms

Linux namespaces (PID, UTS, and mount) are used to isolate containers. PID namespaces prevent processes from interacting across containers, UTS namespaces provide separate hostnames, and mount namespaces isolate filesystem views. The host kernel is still shared.

---

### Supervisor and Process Lifecycle

The supervisor manages container creation, tracks metadata, and handles termination signals. It uses `waitpid()` to clean up child processes and prevent zombie processes.

---

### IPC, Threads, and Synchronization

Pipes are used for logging, and a control channel is used for CLI communication. A bounded buffer with mutexes and condition variables ensures safe concurrent logging without race conditions.

---

### Memory Management and Enforcement

RSS measures physical memory usage. Soft limits generate warnings, while hard limits enforce termination. Enforcement is done in kernel space for accuracy and safety.

---

### Scheduling Behavior

The Completely Fair Scheduler distributes CPU time fairly. CPU-bound tasks consume more CPU, while I/O-bound tasks yield CPU frequently.

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation

Choice: PID, UTS, mount namespaces
Tradeoff: No network isolation
Justification: Focuses on core container isolation concepts

---

### Supervisor Architecture

Choice: Single supervisor process
Tradeoff: Single point of failure
Justification: Simplifies management and coordination

---

### IPC and Logging

Choice: Pipes with bounded buffer
Tradeoff: Synchronization complexity
Justification: Ensures reliable logging without data loss

---

### Kernel Monitor

Choice: Kernel module with periodic checks
Tradeoff: Slight delay in enforcement
Justification: Simpler and stable implementation

---

## 6. Scheduler Experiment Results

### Experiment: CPU-bound vs I/O-bound

| Workload | Observed Behavior                    |
| -------- | ------------------------------------ |
| cpu_hog  | Continuous execution (~1 update/sec) |
| io_pulse | Periodic output due to I/O waits     |

---

### Observation

CPU-bound processes utilize CPU continuously, while I/O-bound processes frequently yield. The scheduler balances fairness and responsiveness.

---

## Conclusion

This project demonstrates container runtime implementation, process isolation, kernel monitoring, IPC mechanisms, and scheduling behavior in Linux.

---
