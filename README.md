
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

<img width="1358" height="214" alt="image" src="https://github.com/user-attachments/assets/a85f35aa-7fb5-404b-b007-7637c041e950" />

This screenshot shows two containers (alpha and beta) successfully started and listed using the `engine ps` command, confirming that both are running simultaneously under a single supervisor process.

---

### 2. Metadata Tracking

<img width="763" height="106" alt="image" src="https://github.com/user-attachments/assets/23c2a792-b940-46f6-991b-f76514a4cfcf" />

The `engine ps` output displays container metadata including PID, state, uptime, and memory limits, confirming that the supervisor is correctly tracking all running containers.

---

### 3. Bounded-buffer Logging

<img width="846" height="287" alt="image" src="https://github.com/user-attachments/assets/a2a9cce0-8e87-42e9-8d60-78a130d8abda" />

This screenshot shows logs retrieved using the `engine logs` command, demonstrating that container output is captured and stored through the bounded-buffer logging pipeline with proper synchronization.

---

### 4. CLI and IPC

<img width="1350" height="55" alt="image" src="https://github.com/user-attachments/assets/6ddaaad7-1bea-4f1d-be78-862b1b82fab6" />

This screenshot demonstrates a CLI command (`engine start gamma`) interacting with the supervisor, confirming successful inter-process communication through the control channel.

---

### 5. Soft-limit Warning

<img width="1411" height="68" alt="image" src="https://github.com/user-attachments/assets/344142c9-d0f5-4267-b263-9b063be91e4a" />

Kernel logs show a soft-limit warning when memory usage exceeds the configured threshold, while the container continues execution.

---

### 6. Hard-limit Enforcement

<img width="1411" height="130" alt="image" src="https://github.com/user-attachments/assets/5a0c4fb2-809b-49e0-a0c0-f261bde85326" />

The kernel terminates the container after exceeding the hard memory limit, and logs confirm that the container was killed and enforcement was applied.

---

### 7. Scheduling Experiment

<img width="1026" height="788" alt="image" src="https://github.com/user-attachments/assets/79777a19-b845-48ed-85f3-99632bc135a6" />

CPU-bound (`cpu_hog`) and I/O-bound (`io_pulse`) workloads run concurrently, demonstrating how the scheduler distributes CPU time based on workload characteristics.

---

### 8. Clean Teardown

<img width="1298" height="162" alt="image" src="https://github.com/user-attachments/assets/75c4cb80-0823-4210-a7a1-380c0c7b5184" />

The absence of defunct processes confirms that all containers were properly terminated and no zombie processes remain.

---

## 4. Engineering Analysis

### Isolation Mechanisms
The runtime uses Linux namespaces (PID, UTS, and mount) to isolate containers.

- PID namespace ensures process isolation between containers  
- UTS namespace allows each container to have its own hostname  
- Mount namespace provides an isolated filesystem view  

All containers share the host kernel, making this a lightweight virtualization approach.

---

### Supervisor and Process Lifecycle
A single supervisor process manages the entire container lifecycle.

- Creates containers using clone()  
- Tracks container metadata such as PID, state, and limits  
- Handles termination signals (SIGTERM and SIGKILL)  
- Uses waitpid() to clean up child processes  

This ensures proper cleanup and prevents zombie processes.

---

### IPC, Threads, and Synchronization
Two IPC mechanisms are used in the system:

- Pipes are used to capture container stdout and stderr for logging  
- A control channel (UNIX socket or FIFO) is used for communication between CLI and supervisor  

A bounded buffer is implemented using mutex locks and condition variables to ensure thread-safe logging and avoid race conditions.

---

### Memory Management and Enforcement
Memory usage is monitored using RSS (Resident Set Size).

- Soft limit generates warning messages when exceeded  
- Hard limit enforces termination using SIGKILL  

Enforcement is performed in kernel space to ensure accuracy and reliability.

---

### Scheduling Behavior
The system relies on the Linux Completely Fair Scheduler (CFS).

- CPU time is distributed fairly among processes  
- CPU-bound processes continuously consume CPU  
- I/O-bound processes frequently yield CPU while waiting  

This demonstrates practical scheduling behavior in Linux systems.

---

## 5. Design Decisions and Trade-offs

### Namespace Isolation
Choice: PID, UTS, and mount namespaces  
Trade-off: No network isolation  
Justification: Focus on core container isolation concepts while keeping the implementation simple

---

### Supervisor Architecture
Choice: Single supervisor process  
Trade-off: Single point of failure  
Justification: Simplifies container management and coordination

---

### IPC and Logging
Choice: Pipes with bounded buffer  
Trade-off: Synchronization complexity  
Justification: Ensures reliable logging without data loss

---

### Kernel Monitor
Choice: Kernel module with periodic checks  
Trade-off: Slight delay in enforcement  
Justification: Provides a simple and stable monitoring mechanism

---

## 6. Scheduler Experiment Results

### Experiment: CPU-bound vs I/O-bound workloads

| Workload   | Observed Behavior                          |
|------------|--------------------------------------------|
| cpu_hog    | Continuous execution (approximately steady CPU usage) |
| io_pulse   | Periodic output due to I/O wait operations |

### Observation
CPU-bound processes utilize CPU continuously, while I/O-bound processes frequently yield CPU. The scheduler balances fairness and responsiveness effectively.

---

## Conclusion

This project demonstrates the implementation of a container runtime, including process isolation using namespaces, kernel-level memory monitoring, inter-process communication, and scheduling behavior in Linux systems.

---
