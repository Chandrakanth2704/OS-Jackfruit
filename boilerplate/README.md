## Engineering Analysis

### Isolation Mechanisms

The runtime achieves isolation using Linux namespaces created via `clone()` with the flags:

CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS

- CLONE_NEWPID  
  Each container runs in its own PID namespace. The first process inside the container appears as PID 1 and cannot see or interact with processes outside its namespace.

- CLONE_NEWUTS  
  Each container has an independent hostname, configured using sethostname().

- CLONE_NEWNS  
  Each container has a separate mount namespace. Combined with chroot(rootfs), the container only sees its assigned filesystem.

Although namespaces isolate process views, the host kernel is fully shared across all containers. This includes the system call interface, scheduler, and physical memory management. Namespaces restrict visibility, not kernel execution.

---

### Supervisor and Process Lifecycle

A long-running supervisor process is essential for correct container management:

- It remains active to call waitpid() on exiting child processes  
- It maintains container metadata throughout the lifecycle  
- It handles CLI requests without blocking  

Lifecycle flow:

1. clone() creates a child process with new namespaces  
2. The child executes container_setup():
   - mounts the root filesystem  
   - calls chroot()  
   - executes the workload using exec()  
3. The parent stores the container PID and marks the state as running  
4. When the child exits, a SIGCHLD signal is triggered  
5. The supervisor calls waitpid(-1, &status, WNOHANG) to reap terminated children  
6. The container state is updated to stopped or killed  

Using WNOHANG prevents the supervisor from blocking, allowing it to manage multiple containers concurrently. Without waitpid(), terminated processes would become zombie processes.

---

### IPC, Threads, and Synchronization

Two IPC mechanisms are used:

Pipes (Logging)

- A pipe is created before clone()  
- The child inherits the write end  
- dup2() redirects stdout and stderr to the pipe  
- The parent reads from the pipe  

This enables continuous, one-way streaming of logs from container to supervisor.

UNIX Domain Socket (Control Channel)

- The supervisor binds to /tmp/engine_supervisor.sock  
- It listens for CLI connections  
- Each CLI command connects, sends a request, receives a response, and disconnects  

Bounded Buffer Synchronization

A circular buffer is used for logging with:

- pthread_mutex_t lock for mutual exclusion  
- pthread_cond_t not_empty for consumer waiting  
- pthread_cond_t not_full for producer waiting  

This ensures no data loss, no race conditions, and safe concurrent access.

---

### Memory Management and Enforcement

Memory usage is measured using RSS (Resident Set Size):

get_mm_rss(task->mm) * PAGE_SIZE

RSS represents physical memory currently in RAM and does not include:

- swapped-out pages  
- allocated but unused memory  
- shared library pages  

Soft limit:

- Generates a warning when exceeded  
- Does not terminate the container  
- Uses a flag to ensure the warning is logged only once  

Hard limit:

- Enforced strictly  
- The process is terminated using SIGKILL  

Enforcement is implemented in kernel space because:

- User-space monitoring can be delayed by scheduling  
- A user-space monitor itself can be terminated  
- Only the kernel can safely access mm_struct and enforce limits atomically  

---

### Scheduling Behavior

Linux uses the Completely Fair Scheduler (CFS).

- Each runnable process has a vruntime  
- Processes are maintained in a red-black tree  
- The scheduler selects the process with the smallest vruntime  

vruntime increases proportional to:

(actual CPU time) / (process weight)

Process weight is determined by nice value:

- nice = 0 → weight 1024  
- nice = +5 → weight 335  
- nice = -5 → weight 3121  

Experimental observations:

- Equal priority CPU-bound processes share CPU equally  
- Lower nice value results in higher CPU allocation  
- CPU-bound processes continuously consume CPU  
- I/O-bound processes frequently yield CPU  

This demonstrates fairness and responsiveness of the scheduler.

---

## 5. Design Decisions and Trade-offs

Namespace Isolation

Choice: PID, UTS, and mount namespaces  
Trade-off: No network namespace isolation  
Justification: Focus on core isolation concepts while keeping implementation simple  

---

Supervisor Architecture

Choice: Single supervisor process  
Trade-off: Single point of failure  
Justification: Simplifies coordination, debugging, and control  

---

IPC and Logging

Choice: Pipes with bounded buffer  
Trade-off: Increased synchronization complexity  
Justification: Ensures reliable and lossless logging  

---

Kernel Monitor

Choice: Kernel module with periodic monitoring  
Trade-off: Slight delay in enforcement  
Justification: Provides a stable and simple implementation  

---

## 6. Scheduler Experiment Results

Experiment: CPU-bound vs I/O-bound workloads

| Workload   | Observed Behavior                          |
|------------|--------------------------------------------|
| cpu_hog    | Continuous execution with high CPU usage   |
| io_pulse   | Periodic execution due to I/O waits        |

Observation:

CPU-bound processes continuously utilize CPU resources, while I/O-bound processes frequently yield the CPU. The scheduler balances fairness and responsiveness effectively.

---

## Conclusion

This project demonstrates a complete container runtime implementation including:

- Process isolation using Linux namespaces  
- Supervisor-based lifecycle management  
- IPC mechanisms for control and logging  
- Kernel-level memory monitoring and enforcement  
- Scheduling behavior under different workloads  

The system provides a simplified but accurate representation of how container runtimes operate in modern Linux environments.
