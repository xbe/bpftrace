# BPFtrace

BPFtrace is a [DTrace](http://dtrace.org)-style dynamic tracing tool for linux, based on the extended BPF capabilities available in recent Linux kernels. BPFtrace uses [LLVM](http://llvm.org) as a backend to compile scripts to BPF-bytecode and makes use of [BCC](https://github.com/iovisor/bcc) for interacting with the Linux BPF system.

For instructions on building BPFtrace, see [INSTALL.md](INSTALL.md)

## Examples

Count system calls:
```
kprobe:[Ss]y[Ss]_*
{
  @[func] = count()
}
```
```
Attaching 376 probes...
^C

...
@[sys_open]: 579
@[SyS_ioctl]: 686
@[sys_bpf]: 730
@[sys_close]: 779
@[SyS_read]: 825
@[sys_write]: 1031
@[sys_poll]: 1796
@[sys_futex]: 2237
@[sys_recvmsg]: 2634
```

Produce a histogram of amount of time (in nanoseconds) spent in the `read()` system call:
```
kprobe:sys_read
{
  @start[tid] = nsecs;
}

kretprobe:sys_read / @start[tid] /
{
  @times = quantize(nsecs - @start[tid]);
  @start[tid] = delete();
}
```
```
Attaching 2 probes...
^C

@start[9134]: 6465933686812

@times:
[0, 1]                 0 |                                                    |
[2, 4)                 0 |                                                    |
[4, 8)                 0 |                                                    |
[8, 16)                0 |                                                    |
[16, 32)               0 |                                                    |
[32, 64)               0 |                                                    |
[64, 128)              0 |                                                    |
[128, 256)             0 |                                                    |
[256, 512)           326 |@                                                   |
[512, 1k)           7715 |@@@@@@@@@@@@@@@@@@@@@@@@@@                          |
[1k, 2k)           15306 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[2k, 4k)             609 |@@                                                  |
[4k, 8k)             611 |@@                                                  |
[8k, 16k)            438 |@                                                   |
[16k, 32k)            59 |                                                    |
[32k, 64k)            36 |                                                    |
[64k, 128k)            5 |                                                    |
```

Print paths of any files opened along with the name of process which opened them:
```
kprobe:sys_open
{
  printf("%s: %s\n", comm, str(arg0))
}
```
```
Attaching 1 probe...
git: .git/objects/70
git: .git/objects/pack
git: .git/objects/da
git: .git/objects/pack
git: /etc/localtime
systemd-journal: /var/log/journal/72d0774c88dc4943ae3d34ac356125dd
DNS Res~ver #15: /etc/hosts
DNS Res~ver #16: /etc/hosts
DNS Res~ver #15: /etc/hosts
^C
```

Whole system profiling (TODO make example check if kernel is on-cpu before recording):
```
profile:hz:99
{
  @[stack] = count()
}
```
```
Attaching 1 probe...
^C

...
@[
_raw_spin_unlock_irq+23
finish_task_switch+117
__schedule+574
schedule_idle+44
do_idle+333
cpu_startup_entry+113
start_secondary+344
verify_cpu+0
]: 83
@[
queue_work_on+41
tty_flip_buffer_push+43
pty_write+83
n_tty_write+434
tty_write+444
__vfs_write+55
vfs_write+177
sys_write+85
entry_SYSCALL_64_fastpath+26
]: 97
@[
cpuidle_enter_state+299
cpuidle_enter+23
call_cpuidle+35
do_idle+394
cpu_startup_entry+113
rest_init+132
start_kernel+1083
x86_64_start_reservations+41
x86_64_start_kernel+323
verify_cpu+0
]: 150
```

## Probe types

### kprobes
Attach a BPFtrace script to a kernel function, to be executed when that function is called:

`kprobe:sys_read { ... }`

### uprobes
Attach script to a userland function:

`uprobe:/bin/bash:readline { ... }`

### tracepoints
Attach script to a statically defined tracepoint in the kernel:

`tracepoint:sched:sched_switch { ... }`

Tracepoints are guaranteed to be stable between kernel versions, unlike kprobes.

### timers
Run the script at specified time intervals:

`profile:hz:99 { ... }`

`profile:s:1 { ... }`

`profile:ms:20 { ... }`

`profile:us:1500 { ... }`

### Multiple attachment points
A single probe can be attached to multiple events:

`kprobe:sys_read,kprobe:sys_write { ... }`

### Wildcards
Some probe types allow wildcards to be used when attaching a probe:

`kprobe:SyS_* { ... }`

### Predicates
Define conditions for which a probe should be executed:

`kprobe:sys_open / uid == 0 / { ... }`

## Builtins
The following variables and functions are available for use in bpftrace scripts:

Variables:
- `pid` - Process ID (kernel tgid)
- `tid` - Thread ID (kernel pid)
- `uid` - User ID
- `gid` - Group ID
- `nsecs` - Nanosecond timestamp
- `cpu` - Processor ID
- `comm` - Process name
- `stack` - Kernel stack trace
- `ustack` - User stack trace
- `arg0`, `arg1`, ... etc. - Arguments to the function being traced
- `retval` - Return value from function being traced
- `func` - Name of the function currently being traced

Functions:
- `quantize(int n)` - Produce a log2 histogram of values of `n`
- `count()` - Count the number of times this function is called
- `delete()` - Delete the map element this is assigned to
- `str(char *s)` - Returns the string pointed to by `s`
- `printf(char *fmt, ...)` - Write to stdout
- `sym(void *p)` - Resolve kernel address
- `usym(void *p)` - Resolve user space address (incomplete)
- `reg(char *name)` - Returns the value stored in the named register
