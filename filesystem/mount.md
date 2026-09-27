# Linux `mount` Command

## What does `mount` do?

The `mount` command **attaches a filesystem to a directory in the Linux filesystem tree**, making the files in that filesystem accessible through that directory.

### Basic syntax

```bash
sudo mount <device> <directory>
```

Example:

```bash
sudo mount /dev/sdb1 /mnt
```

This connects:

```text
/dev/sdb1
    │
    │ mount
    ▼
  /mnt
```

After mounting, the files stored on `/dev/sdb1` can be accessed through:

```bash
ls /mnt
```

For example:

```text
/dev/sdb1
└── file.txt
```

becomes accessible as:

```text
/mnt/file.txt
```

---

## Why do we need `mount`?

Linux uses **one unified filesystem tree**:

```text
/
├── home
├── etc
├── usr
├── var
├── mnt
└── ...
```

A separate filesystem can be attached to any directory in this tree.

For example:

```text
          /
          │
    ┌─────┴─────┐
   home         mnt
                 │
              /dev/sdb1
```

After mounting, `/mnt` becomes the entry point to the filesystem on `/dev/sdb1`.

---

## `mount` with no arguments

Running:

```bash
mount
```

does not mount anything.

Instead, it displays the filesystems that are **currently mounted** and where they are mounted.

For example:

```text
overlay on / type overlay (...)
proc on /proc type proc (...)
tmpfs on /dev type tmpfs (...)
sysfs on /sys type sysfs (...)
```

This means:

```text
Filesystem       Mount point
--------------------------------
overlay