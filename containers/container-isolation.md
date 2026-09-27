# Container Isolation

## Core idea

A container is **not a separate machine**.

A container is a group of processes running on the **host Linux kernel**, but Linux gives those processes isolated views of the system.

```text
                    HOST MACHINE
                         │
                    Linux Kernel
                         │
        ┌────────────────┼────────────────┐
        │                │                │
      Host          Container A      Container B
                     │
             ┌───────┼────────┐
             │       │        │
          PID NS   NET NS   Mount NS
             │       │        │
          process   network  filesystem
           view      view      view
```

The main mechanisms are:

- **Namespaces** → isolate what processes can see
- **Cgroups** → control and account for resources
- **OverlayFS** → provides the container's filesystem view
- **Mounts** → construct the filesystem tree visible to the container

---

# 1. Filesystem isolation — OverlayFS

Inside a container:

```text
/
├── bin
├── etc
├── home
├── usr
├── var
└── ...
```

It looks like a complete Ubuntu filesystem.

Docker commonly constructs this using **OverlayFS**.

Example:

```text
overlay on / type overlay (
    lowerdir=A:B:C,
    upperdir=D/diff,
    workdir=D/work
)
```

Conceptually:

```text
                 Container /
                      │
                  OverlayFS
                      │
             ┌────────┴────────┐
             │                 │
        Lower layers       Upper layer
        read-only           writable
             │                 │
       Docker image      container changes
```

### Lower layers

The `lowerdir` contains read-only image layers.

For example:

```text
Layer 1 → base Ubuntu filesystem
Layer 2 → installed packages
Layer 3 → application files
```

### Upper layer

The `upperdir` stores changes made by this particular container.

If the image contains:

```text
/etc/example
```

and the container executes:

```bash
echo hello > /etc/example
```

the original image layer is not modified.

OverlayFS presents the container with the changed version using the writable layer.

---

# 2. `/proc` — process and kernel information

```text
proc on /proc type proc
```

`/proc` is a **virtual filesystem provided by the Linux kernel**.

It is not a normal directory stored on disk.

For example:

```bash
cat /proc/cpuinfo
cat /proc/meminfo
cat /proc/1/status
```

The kernel dynamically provides this information.

`ps aux` also gets much of its process information through `/proc`.

### Important container concept

The container can have a different **view** of `/proc` because of Linux namespaces.

```text
                 HOST KERNEL
                      │
          ┌───────────┴───────────┐
          │                       │
     Host namespace        Container namespace
          │                       │
      /proc view              /proc view
          │                       │
   host processes        container processes
```

Same kernel.

Different view.

---

# 3. `/dev` — devices

```text
tmpfs on /dev type tmpfs
```

The container gets its own `/dev` environment.

For example:

```text
/dev/null
/dev/zero
/dev/random
/dev/urandom
/dev/tty
/dev/console
```

These are interfaces to kernel/device functionality rather than ordinary files from the Ubuntu image.

Docker controls which devices the container can access.

---

# 4. `/dev/pts` — pseudo-terminals

```text
devpts on /dev/pts type devpts
```

`/dev/pts` provides **pseudo-terminal devices**.

When you run:

```bash
docker exec -it <container> bash
```

your interactive terminal uses a pseudo-terminal.

So:

```text
/dev/pts
     │
     └── pseudo-terminals
```

These are provided by the Linux kernel.

---

# 5. `/sys` — kernel/device information

```text
sysfs on /sys type sysfs
```

`sysfs` is another kernel-provided virtual filesystem.

It exposes information about things such as:

```text
devices
drivers
CPU
memory
kernel objects
```

For example:

```bash
ls /sys/devices
```

`/sys` is therefore not simply part of the Ubuntu Docker image.

It is an interface to the kernel.

---

# 6. Cgroups — resource control

The container also has cgroup mounts such as:

```text
cgroup on /sys/fs/cgroup/memory
cgroup on /sys/fs/cgroup/pids
cgroup on /sys/fs/cgroup/cpu,cpuacct
cgroup on /sys/fs/cgroup/cpuset
```

Cgroups allow the Linux kernel to control and account for resources.

Conceptually:

```text
                  Linux kernel
                       │
                    cgroups
                       │
          ┌────────────┼────────────┐
          │            │            │
         CPU         Memory        PIDs
          │            │            │
       limits        limits       limits
```

For example, a container could be restricted to:

```text
CPU    → 2 CPUs
Memory → 512 MB
PIDs   → 100 processes
```

The **Linux kernel enforces these restrictions**.

Docker does not have its own CPU scheduler.

---

# 7. Special container files

Docker also provides special files such as:

```text
/etc/hostname
/etc/hosts
/etc/resolv.conf
```

For example:

```bash
cat /etc/hostname
cat /etc/resolv.conf
```

These allow the container to have its own hostname and appropriate DNS/network configuration.

---

# 8. Why are some `/proc` entries mounted as `tmpfs`?

You may see:

```text
tmpfs on /proc/kcore
tmpfs on /proc/keys
tmpfs on /proc/interrupts
tmpfs on /proc/timer_list
```

Do not interpret these as many independent RAM disks.

Some of these mounts are used to **mask or restrict access to sensitive kernel interfaces**.

The idea is:

```text
Host kernel
     │
     ├── many kernel interfaces
     │
     └── container gets a restricted view
```

This contributes to container isolation.

---

# Putting it together

When you run:

```bash
mount
```

inside a container, you can see several pieces of this isolation.

| Component | Purpose |
|---|---|
| OverlayFS | Container filesystem |
| `/proc` | Process/kernel information |
| `/dev` | Controlled device environment |
| `/dev/pts` | Pseudo-terminals |
| `/sys` | Kernel/device information |
| Cgroups | Resource control |
| Namespaces | Isolation of system views |

---

# The key mental model

Do **not** think of a container as:

```text
Host
├── Host OS
│   └── Kernel
│
└── Container
    └── Kernel
```

That is closer to the VM model.

Instead:

```text
                  PHYSICAL MACHINE
                         │
                    Linux Kernel
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
      Host          Container A       Container B
                        │
                ┌───────┼───────┐
                │       │       │
              PID NS  NET NS  Mount NS
                │       │       │
             process  network filesystem
               view     view      view
```

### Remember

> **A container is not another computer. It is an isolated view of the host system created using Linux kernel mechanisms.**

The container gets its own:

- process view
- network view
- filesystem view
- resource limits
- controlled device access

But all of those are ultimately implemented by the **same host Linux kernel**.

This is the foundation for understanding Docker and Kubernetes.