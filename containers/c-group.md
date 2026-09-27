
## What are Cgroups?

**Cgroups (Control Groups)** are a Linux kernel mechanism used to **group processes and control/account for their resource usage**.

They are commonly used by container runtimes such as Docker to control the resources available to a container.

For example, several processes belonging to the same container can be placed into the same cgroup.

```text
                    Linux kernel
                         │
                      Cgroups
                         │
              ┌──────────┴──────────┐
              │                     │
         Process group          Process group
              │                     │
          Container A            Container B
```

The important idea is:

> **A cgroup is a group of processes that the Linux kernel can manage together for resource control and accounting.**

---

# 2. Why do containers need Cgroups?

Suppose a server has:

```text
CPU    → 8 cores
Memory → 16 GB
```

You run a container containing an application.

Without resource limits, that application could potentially consume a large amount of the available resources.

Cgroups allow the container runtime to configure limits.

For example:

```text
Container A

CPU    → 2 CPUs
Memory → 512 MB
PIDs   → 100 processes
```

The Linux kernel then enforces those restrictions.

---

# 3. Resource Control

Cgroups can control/account for resources such as:

```text
CPU
Memory
PIDs
I/O
```

Conceptually:

```text
                    Linux kernel
                         │
                       cgroups
                         │
          ┌──────────────┼──────────────┐
          │              │              │
         CPU           Memory          PIDs
          │              │              │
       limits         limits          limits
          │              │              │
          └──────────────┼──────────────┘
                         │
                     Container
```

For example:

```text
CPU    → 2 CPUs
Memory → 512 MB
PIDs   → 100 processes
```

The important distinction is:

> **Docker configures the cgroup, but the Linux kernel enforces the resource restrictions.**

Docker does **not** have its own CPU scheduler.

Instead:

```text
Docker
  │
  │ configures
  ▼
Cgroup
  │
  │ enforced by
  ▼
Linux kernel
  │
  ├── CPU
  ├── Memory
  ├── PIDs
  └── I/O
```

---

# 4. Cgroups and Docker

When Docker starts a container, it places the container's processes into a cgroup.

Conceptually:

```text
Docker
  │
  │ creates/configures
  ▼
Cgroup
  │
  ├── CPU limits
  ├── Memory limits
  ├── PID limits
  └── I/O controls
       │
       ▼
Container processes
```

This allows the Linux kernel to treat the processes as a group when applying resource controls.

---

# 5. Inspecting Cgroups

Linux exposes information about processes through the `/proc` virtual filesystem.

You can inspect the cgroup information associated with PID 1 using:

```bash
cat /proc/1/cgroup
```

It means:

> **Read the cgroup information of process 1 and print it to the terminal.**

---

## 5.1 `cat`

`cat` is a Linux command used to **read and display the contents of a file**.

Example:

```bash
cat file.txt
```

Means:

> Open `file.txt`, read its contents, and print them.

Therefore:

```bash
cat /proc/1/cgroup
```

means:

> Read `/proc/1/cgroup` and display its contents.

---

## 5.2 `/proc`

`/proc` is a **virtual filesystem provided by the Linux kernel**.

It exposes information about the running system and processes.

Examples:

```text
/proc/cpuinfo       → CPU information
/proc/meminfo       → Memory information
/proc/uptime        → System uptime
/proc/1/            → Information about process 1
```

Think of `/proc` as:

> **A window into information maintained by the Linux kernel.**

The files in `/proc` are generally **not ordinary files stored on your disk**.

---

## 5.3 `/proc/1`

The `1` represents **PID 1 (Process ID 1)**.

Every process running on Linux has a PID.

You can see processes and their PIDs using:

```bash
ps aux
```

PID 1 is special because it is the first userspace process in a particular Linux environment.

On a normal Linux system, PID 1 is commonly:

```text
systemd
```

Inside a Docker container, PID 1 is normally the **main process of the container**.

For example:

```bash
docker run ubuntu sleep 1000
```

Inside that container:

```text
/proc/1/
```

refers to the `sleep` process.

Therefore:

```text
/proc/1
```

means:

> **Information about the process whose PID is 1.**

---

## 5.4 `/proc/1/cgroup`

The final part:

```text
cgroup
```

refers to the cgroup information associated with that process.

Therefore:

```text
/proc/1/cgroup
```

is a virtual file that shows:

> **Which cgroup(s) process 1 belongs to.**

---

# 6. Breaking the Path Apart

```text
/proc/1/cgroup
  │   │   │
  │   │   └── cgroup information
  │   │
  │   └────── process with PID 1
  │
  └────────── kernel's virtual process filesystem
```

---

# 7. Example Output

You might see output such as:

```text
13:hugetlb:/docker/501740...
12:cpuset:/docker/501740...
11:memory:/docker/501740...
10:cpu:/docker/501740...
```

Each line represents information about a **cgroup controller and its hierarchy**.

For example:

```text
11:memory:/docker/501740...
```

can be thought of conceptually as:

```text
memory
   │
   └── memory controller
           │
           └── /docker/501740...
                    │
                    └── cgroup associated with the container
```

The exact format depends on whether the system is using **cgroup v1 or cgroup v2**.

---

# 8. What does `/docker/<container-id>` mean?

A path such as:

```text
/docker/501740...
```

can indicate that the process belongs to a cgroup created as part of Docker's container resource-management hierarchy.

Conceptually:

```text
Docker
  │
  └── Container
        │
        └── Processes
              │
              └── Cgroup
                    │
                    └── /docker/<container-id>
```

The container's processes can therefore be grouped together for resource management.

---

# 9. Cgroups vs Namespaces

Cgroups and namespaces are both important parts of Linux containers, but they solve different problems.

A useful mental model is:

```text
Namespaces → "What can this process see?"
Cgroups    → "How much of a resource can this process use?"
```

For example:

```text
Namespace:
Container sees its own process IDs.

Cgroup:
Container can be limited to 512 MB of memory.
```

So:

> **Namespaces provide isolation of what processes can see, while cgroups provide resource control and accounting.**

They work together to provide important parts of container isolation.

---

# 10. Does Docker enforce the limits?

Docker itself does not directly enforce CPU or memory limits.

Instead:

```text
User
 │
 │ docker run --memory=512m ...
 ▼
Docker
 │
 │ configures
 ▼
Linux cgroup
 │
 │ enforced by
 ▼
Linux kernel
```

The **Linux kernel is responsible for actually enforcing the resource controls**.

This is an important distinction:

> **Docker is the manager/configurator; the Linux kernel is the enforcer.**

---

# 11. Important: `cat /proc/1/cgroup` Does Not Create a Cgroup

The command:

```bash
cat /proc/1/cgroup
```

**does not create or modify a cgroup.**

It only reads information.

```text
Ask for information
       ↓
Read /proc/1/cgroup
       ↓
Print the information
```

`cat` is simply the tool being used to **inspect the information**.

---

# 12. Mental Model

```text
                         Docker
                           │
                           │ configures
                           ▼
                        Cgroups
                           │
                           │ resource control
                           ▼
                     Linux kernel
                           │
              ┌────────────┼────────────┐
              │            │            │
             CPU         Memory        PIDs
              │            │            │
              └────────────┼────────────┘
                           │
                    Container processes
```

To inspect the cgroup membership of PID 1:

```text
You
 │
 │ cat /proc/1/cgroup
 ▼
/proc virtual filesystem
 │
 │ information about PID 1
 ▼
Linux kernel
 │
 ▼
Cgroup information
 │
 ▼
Terminal
```

---

# 13. Key Takeaways

```text
Cgroups
→ Linux mechanism for grouping processes.

Main purpose
→ Resource control and accounting.

Common resources
→ CPU, Memory, PIDs, I/O.

Docker's role
→ Configure/manage cgroups.

Kernel's role
→ Enforce the configured resource controls.

Inspect PID 1's cgroups
→ cat /proc/1/cgroup

Namespaces
→ Control what processes can see.

Cgroups
→ Control/account for how resources are used.
```

### One-line summary

> **Cgroups are a Linux kernel mechanism that groups processes so their resource usage can be controlled and accounted for; Docker uses cgroups to manage container resources.**