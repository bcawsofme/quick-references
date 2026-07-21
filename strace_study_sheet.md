# strace Study Sheet

`strace` shows the system calls a process makes and the signals it receives. Use it when a program is failing and normal logs do not explain why.

## Mental Model

| Concept | Meaning | Why it matters |
| --- | --- | --- |
| System call | A request from a user-space program to the Linux kernel. | File, network, process, memory, and permission problems often appear here. |
| User space | Where normal application code runs. | Your app cannot directly access hardware or kernel resources. |
| Kernel space | Where the Linux kernel handles privileged operations. | `strace` shows the app crossing into this boundary. |
| Signal | An asynchronous notification sent to a process. | Explains exits, kills, timeouts, and crashes. |
| File descriptor | A numeric handle for an open file, socket, pipe, or device. | Many syscalls operate on file descriptors instead of paths. |
| errno | Numeric error code returned by failed syscalls. | `ENOENT`, `EACCES`, and `ECONNREFUSED` are often the answer. |
| Blocking syscall | A syscall that waits for input, output, locks, or timers. | Useful when a process appears hung. |

## Core Commands

```bash
strace <command>
strace -o trace.log <command>
strace -f <command>
strace -ff -o trace <command>
strace -p <pid>
strace -tt -T <command>
strace -c <command>
```

## Common Options

| Option | Meaning | Use when |
| --- | --- | --- |
| `-o <file>` | Write trace output to a file. | Output is too noisy for the terminal. |
| `-f` | Follow child processes. | The program forks or starts subprocesses. |
| `-ff` | Write each traced process separately with `-o`. | You need per-process trace files. |
| `-p <pid>` | Attach to a running process. | A service is already hung or failing. |
| `-tt` | Show high-resolution timestamps. | You need event ordering. |
| `-T` | Show time spent in each syscall. | You need to find slow syscalls. |
| `-c` | Summarize syscall counts and time. | You need a quick performance overview. |
| `-s <n>` | Increase string size captured. | Arguments are truncated. |
| `-e trace=<set>` | Trace only selected syscall groups. | You want less noise. |
| `-e fault=<syscall>` | Inject syscall faults. | Advanced failure testing. |

## Trace by Category

```bash
strace -e trace=file <command>
strace -e trace=network <command>
strace -e trace=process <command>
strace -e trace=signal <command>
strace -e trace=memory <command>
strace -e trace=desc <command>
```

## File and Permission Problems

```bash
strace -e trace=file <command>
strace -f -e trace=file -o trace.log <command>
grep -E "ENOENT|EACCES|EPERM" trace.log
```

| Error | Meaning | Common cause |
| --- | --- | --- |
| `ENOENT` | No such file or directory. | Missing file, wrong path, missing shared library. |
| `EACCES` | Permission denied. | File mode, directory execute bit, user mismatch. |
| `EPERM` | Operation not permitted. | Missing capability, blocked privileged operation. |
| `EROFS` | Read-only filesystem. | Container or mount is read-only. |
| `ENOSPC` | No space left on device. | Disk full, inode exhaustion, quota. |

## Network Problems

```bash
strace -e trace=network <command>
strace -f -e trace=connect,sendto,recvfrom <command>
strace -f -e trace=network -s 200 -o net.log <command>
```

| Error | Meaning | Common cause |
| --- | --- | --- |
| `ECONNREFUSED` | Remote actively refused the connection. | Service not listening or wrong port. |
| `ETIMEDOUT` | Connection attempt timed out. | Firewall, route, network policy, unreachable host. |
| `ENETUNREACH` | Network unreachable. | Missing route or broken network config. |
| `EHOSTUNREACH` | Host unreachable. | Routing, firewall, or host down. |
| `EADDRINUSE` | Address already in use. | Port already bound. |

## Process and Exec Problems

```bash
strace -f -e trace=process <command>
strace -f -e trace=execve,clone,fork,vfork,wait4 <command>
strace -f -s 200 -o proc.log <command>
```

| Syscall | Meaning | What to look for |
| --- | --- | --- |
| `execve` | Start a new program. | Wrong binary, missing interpreter, environment variables. |
| `clone` | Create a process or thread. | Forking behavior and subprocesses. |
| `wait4` | Wait for a child process. | Parent waiting on a stuck or failed child. |
| `exit_group` | Exit all threads in a process. | Final exit status. |

## Hung Process Triage

```bash
strace -p <pid>
strace -tt -T -p <pid>
strace -f -tt -T -p <pid> -o hung.log
```

| Pattern | Likely meaning |
| --- | --- |
| Repeating `futex(...)` | Waiting on thread lock or synchronization. |
| Blocking `read(...)` | Waiting for input from file, pipe, socket, or stdin. |
| Blocking `poll(...)` or `epoll_wait(...)` | Waiting for socket or file descriptor events. |
| Blocking `connect(...)` | Waiting on network connection. |
| Repeating `stat(...) = -1 ENOENT` | Searching for missing files or libraries. |

## Containers and Kubernetes

```bash
kubectl exec -it <pod> -- strace <command>
kubectl exec -it <pod> -- strace -f -o /tmp/trace.log <command>
kubectl cp <namespace>/<pod>:/tmp/trace.log ./trace.log
kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>
```

Notes:

```text
Some containers do not include strace.
Attaching to an existing process may need SYS_PTRACE capability.
Distroless images usually need an ephemeral debug container.
Security policies may block ptrace.
```

## Reading strace Output

```text
openat(AT_FDCWD, "/etc/app/config.yaml", O_RDONLY) = -1 ENOENT (No such file or directory)
connect(3, {sa_family=AF_INET, sin_port=htons(5432), ...}, 16) = -1 ECONNREFUSED (Connection refused)
read(4, "hello", 4096) = 5
write(1, "hello", 5) = 5
exit_group(1) = ?
```

| Part | Meaning |
| --- | --- |
| `openat` | Syscall name. |
| `AT_FDCWD` | Path is relative to current working directory. |
| `"/etc/app/config.yaml"` | Syscall argument. |
| `O_RDONLY` | Open mode flag. |
| `= -1` | Syscall failed. |
| `ENOENT` | Error code. |
| `(No such file or directory)` | Human-readable error. |

## Useful Pairings

```bash
ps aux | grep <name>
lsof -p <pid>
ss -tulpn
journalctl -u <service> -f
ldd <binary>
file <binary>
which <command>
```

## When to Use strace

| Symptom | Start with |
| --- | --- |
| Command says file missing but path looks right. | `strace -e trace=file <command>` |
| App cannot connect to another service. | `strace -e trace=network <command>` |
| Process hangs with no logs. | `strace -tt -T -p <pid>` |
| Program exits immediately. | `strace -f -s 200 <command>` |
| Subprocess fails silently. | `strace -f -e trace=process <command>` |
| Container works locally but not in Kubernetes. | `kubectl debug` plus `strace`, `ss`, `dig`, and logs. |

## Safety Notes

```text
strace can expose secrets in command arguments, environment, file paths, and network data.
Tracing production services adds overhead.
Prefer short captures with filters.
Store trace logs securely.
Delete trace logs after debugging.
```
