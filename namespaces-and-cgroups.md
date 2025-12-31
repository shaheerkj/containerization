## 1. The Core Problem Containers Solve

Before containers, Linux had:

- Strong **process isolation** (process IDs, users)
    
- Strong **resource accounting** (per-process limits via `ulimit`)
    
- But **no unified way** to:
    
    - Isolate groups of processes from each other
        
    - Enforce resource limits _collectively_
        
    - Present each group with its own “mini system view”
        

Containers emerged to solve:

> “Run multiple isolated application environments on the same kernel **safely and efficiently**.”

---

## 2. Linux Namespaces — _Isolation_

### What a Namespace Is

A **namespace** isolates a _system resource view_ so that processes think they have their own instance of that resource.

> Namespaces answer:  
> **“What can a process see?”**

### Key Namespace Types (Docker-relevant)

|Namespace|Isolates|Effect|
|---|---|---|
|`pid`|Process IDs|Container sees PID 1 as its first process|
|`mnt`|Mount points|Separate filesystem view|
|`net`|Networking|Own interfaces, IPs, routing|
|`uts`|Hostname/domain|Container hostname isolation|
|`ipc`|IPC mechanisms|No shared shared memory/semaphores|
|`user`|User IDs|Root in container ≠ root on host|
|`cgroup`|cgroup hierarchy|Visibility of resource limits|

### Example (conceptual)

Inside a container:

`ps aux`

You might see:

`PID 1  nginx`

But on the host:

`ps aux | grep nginx`

That same process might be PID 24561.

This illusion is created by the **PID namespace**.

---

## 3. Linux cgroups — _Resource Control_

### What cgroups Are

**Control Groups (cgroups)** limit, account for, and isolate **resource usage** of process groups.

> cgroups answer:  
> **“How much can a process use?”**

### Resources Controlled by cgroups

|Resource|What cgroups control|
|---|---|
|CPU|Shares, quotas|
|Memory|Hard/soft limits, OOM behavior|
|Block I/O|Disk throughput|
|Network|Bandwidth (via tc integration)|
|PIDs|Process count limits|

### Example

Limit a container to:

- 1 CPU core
    
- 512 MB RAM
    

Without cgroups:

- One process could consume **all CPU/RAM**
    
- System becomes unstable
    

With cgroups:

- Kernel enforces limits **at runtime**
    

---

## 4. Why Namespaces Alone Were Not Enough

Namespaces provide **isolation**, but **no resource limits**.

Without cgroups:

- A container could still:
    
    - Consume all CPU
        
    - Trigger host OOM
        
    - Starve other workloads
        

Thus:

- **Namespaces = isolation**
    
- **cgroups = fairness and safety**
    

Containers require **both**.

---

## 5. How Docker Uses Namespaces + cgroups

Docker is **not virtualization**. It does **not** emulate hardware.

Instead, Docker:

1. Uses **namespaces** to isolate:
    
    - Process tree
        
    - Network stack
        
    - Filesystem
        
    - Users
        
2. Uses **cgroups** to constrain:
    
    - CPU
        
    - Memory
        
    - I/O
        
3. Uses **Union filesystems** (overlayfs) for images
    
4. Uses **capabilities** to drop kernel privileges
    

### Docker Container =

`Processes + Namespaces (view isolation) + cgroups (resource limits) + filesystem layers + security policies`

---

## 6. Step-by-Step: What Happens When You Run a Container

When you run:

`docker run nginx`

Under the hood:

1. Docker creates new namespaces:
    
    - `clone()` system call with namespace flags
        
2. Assigns process to cgroups:
    
    - CPU, memory, pids
        
3. Mounts container filesystem:
    
    - overlayfs
        
4. Drops privileges:
    
    - Linux capabilities
        
5. Starts PID 1 inside container
    

All of this is **native kernel functionality**.

---

## 7. Why Containers Were Impossible Before cgroups + Namespaces

### Before Namespaces

- No PID isolation
    
- No network isolation
    
- Processes could see each other
    

### Before cgroups

- No group-based resource limits
    
- One workload could crash the system
    

### Together They Enabled:

- Multi-tenant systems
    
- Secure workload isolation
    
- Predictable resource usage
    
- Fast startup (no hypervisor)
    

---

## 8. Containers vs Virtual Machines (Kernel Perspective)

|Aspect|VM|Container|
|---|---|---|
|Kernel|Per VM|Shared|
|Isolation|Hardware-level|Kernel-level|
|Startup|Minutes|Seconds|
|Overhead|High|Low|
|Density|Low|High|

Containers trade **kernel sharing** for **performance**.

---

## 9. Security Implications (Important for You)

From a security standpoint:

- Containers are **not security boundaries**
    
- Kernel vulnerability = container escape
    
- This is why:
    
    - Seccomp
        
    - AppArmor / SELinux
        
    - User namespaces  
        are critical
        

This is exactly why **DevSecOps and cloud security** emphasize:

- Least privilege
    
- Runtime protection
    
- Image scanning
    

---

## 10. Why This Matters for Docker, Kubernetes, and Cloud

- Docker standardized container UX
    
- Kubernetes orchestrates cgroups + namespaces at scale
    
- Cloud services (AKS, EKS, GKE) rely on this model
    
- Serverless platforms use containers under the hood