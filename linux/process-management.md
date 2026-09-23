# Linux Process Management

## `ps aux`

`ps aux` displays information about processes currently running on the system.

### Command

```bash
ps aux
```

### Options

* `a` — show processes belonging to all users, including processes not associated with the current terminal.
* `u` — display processes in a user-oriented format.
* `x` — include processes that do not have a controlling terminal, such as many background services.

### Important columns

```text
USER    PID    %CPU   %MEM   VSZ   RSS   TTY   STAT   START   TIME   COMMAND
```

* `USER` — user who owns the process
* `PID` — Process ID
* `%CPU` — CPU usage
* `%MEM` — memory usage
* `VSZ` — virtual memory size
* `RSS` — physical RAM currently occupied
* `TTY` — terminal associated with the process
* `STAT` — process state
* `START` — process start time
* `TIME` — CPU time consumed
* `COMMAND` — command/program that started the process

## Useful commands

Find a particular process:

```bash
ps aux | grep java
```

Find processes by name:

```bash
pgrep java
```

Inspect a process:

```bash
ps -p <PID>
```

Monitor processes continuously:

```bash
top
```

Terminate a process:

```bash
kill <PID>
```

## Example

```text
vasu   18231   4.2   3.1   ...   java -jar user-service.jar
```

Here:

* `18231` is the PID.
* `4.2` means the process is currently using about 4.2% CPU.
* `3.1` means about 3.1% of system memory.
* The command that started it was Java.

