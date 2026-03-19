# FajarOS v3.2 "Surya Rising"

> The world's first operating system written 100% in [Fajar Lang](https://github.com/fajarkraton/fajar-lang) — where kernel safety, hardware drivers, and AI inference share one language, one type system, and one compiler.

**Target Hardware:** [Radxa Dragon Q6A](https://radxa.com/products/dragon/q6a/) (Qualcomm QCS6490)

## Quick Start

```bash
# Build compiler
cd ~/fajar-lang && cargo build --release --features native

# Build kernel
cd ~/fajar-os
fj build kernel/boot/start.fj --target aarch64-unknown-none --no-std -o fajaros.elf

# Run on QEMU
qemu-system-aarch64 -M virt,gic-version=3 -cpu cortex-a76 -m 512M \
  -kernel fajaros.elf -nographic -serial mon:stdio
```

## Features

| Feature | Detail | Status |
|---------|--------|--------|
| **Preemptive scheduler** | 16 PIDs, 4 priority levels (IDLE/NORMAL/HIGH/REALTIME), round-robin | ✅ |
| **17 syscalls** | EXIT, WRITE, YIELD, GETPID, IPC, SPAWN, KILL, WAIT, READ, SVC_REG/LOOKUP, SIGNAL, PIPE | ✅ |
| **Per-process page tables** | Unique L0 per process, TTBR0 switch + TLB flush on context switch | ✅ |
| **EL0 user space** | Unprivileged processes, SVC syscalls, timer preemption, mixed EL0/EL1 | ✅ |
| **Service registry** | 16-slot microkernel service table, 3 kernel services (UART, Timer, Memory) | ✅ |
| **Signals** | SIGTERM, SIGKILL, SIGCHLD with per-process handlers | ✅ |
| **Pipes** | 8 pipes × 4KB circular buffer, pipe_create/write/read/close | ✅ |
| **IPC v2** | 8-message circular queue per process, blocking recv, total_sent tracking | ✅ |
| **Memory protection** | Data/instruction abort handlers, kill faulting process with address display | ✅ |
| **Interactive shell** | Ctrl+C, up-arrow history, tab completion, `[PID] fjsh>` prompt | ✅ |
| **160 shell commands** | ps, top, spawn, kill, wait, nice, pmap, memstat, svclist, signal, pipe, write, append, net, display, ... | ✅ |
| **Idle process** | PID 15, WFI loop, IDLE priority — runs when no other process READY | ✅ |
| **Process names** | Stored in process table, shown in ps/kill/spawn/top output | ✅ |
| **CPU% per process** | Per-process tick counting, CPU% = ticks/total_irqs in `top` | ✅ |
| **GICv3 interrupts** | Distributor + Redistributor + CPU interface, Timer PPI 30 | ✅ |
| **MMU** | 4-level page tables, 2MB blocks, caches ON, identity-mapped | ✅ |
| **SPSR save/restore** | 272-byte exception frames (x0-x30 + SP + ELR + SPSR) | ✅ |
| **FAT filesystem** | VirtIO block device + FAT16/32 read/write/list | ✅ |
| **String literals** | `println("text")` in @kernel compiles to .rodata (no heap) | ✅ |

## Demo

```
FajarOS v3.0 Surya

[0] fjsh> ps
PID STATE  TICKS PRI NAME
--- -----  ----- --- ----
 0  *RUN*  0   1  kernel
15  READY  0   0  idle

[0] fjsh> demo
Starting demo...
1:A 2:B 3:C(svc)
AAABBBCCCAAABBB...  (preemptive round-robin)

[0] fjsh> ps
PID STATE  TICKS PRI NAME
--- -----  ----- --- ----
 0  *RUN*  79   1  kernel
 1   DEAD  4   1  A
 2   DEAD  3   1  B
 3   DEAD  1   1  C
15  READY  0   0  idle

[0] fjsh> spawn a
PID 1 [a]

[0] fjsh> spawn u
PID 2 [u] EL0
USER!                    ← EL0 process prints via SVC

[0] fjsh> top
PID ST  TICKS  CPU%
--- --  -----  ----
 0  *   189    96%
 1  D   3    1%
 2  D   0    0%
15  R   0    0%
Up: 1942ms  IRQs: 189

[0] fjsh> svclist
SVC  PID  PORT
---  ---  ----
 U    0    1       ← UART service
 T    0    2       ← Timer service
 M    0    3       ← Memory service

[0] fjsh> memstat
Page tables: 7 (28KB)
Processes: 5/16
Stack: 320KB
Kernel:  0x40000000-0x47FFFFFF (128MB)
MMIO:    0x00000000-0x0FFFFFFF (256MB)
Stacks:  0x48000000-0x48FFFFFF (16MB)
PT pool: 0x46000000-0x46FFFFFF (16MB)

[0] fjsh> pmap
PID  TTBR0        STACK
---  ----------   -----
 0   0x46000000   0x48010000
 1   0x46004000   0x48020000
 2   0x46005000   0x48030000
15   0x46000000   0x48100000

[0] fjsh> test
1.P
2.P
3.P
4.P
5.P
5/5 pass
```

## Architecture

```
EL0 (User, unprivileged)            EL1 (Kernel, privileged)
┌──────────────────┐               ┌──────────────────────────────┐
│ Process A (a)    │──SVC #0────→  │ Syscall Dispatch (17 calls)  │
│ Process B (b)    │──SVC #0────→  │ Priority Scheduler (16 PIDs) │
│ Process U (EL0)  │──SVC #0────→  │ Per-process Page Tables      │
└──────────────────┘  ←──eret──── │ Service Registry             │
         ↑ Timer IRQ (10ms)        │ Signal Delivery              │
         │ sched_switch()          │ IPC Message Queues           │
         │ TTBR0 switch            │ Pipe I/O                    │
         │ TLB flush               └──────────────────────────────┘
```

### Memory Map

```
0x00000000-0x0FFFFFFF  MMIO (Device-nGnRnE): GIC, UART, VirtIO
0x40000000-0x47FFFFFF  Kernel (Normal WB): code, data, process tables
0x46000000-0x46FFFFFF  Page Table Pool (bump allocator)
0x47000000-0x47001FFF  Process table (16 × 256 bytes)
0x47002000-0x47003FFF  IPC mailboxes (16 × 256 bytes)
0x4700A000-0x4700A1FF  Service registry (16 × 32 bytes)
0x4700B000-0x4700CFFF  Pipe buffers (8 × 4136 bytes)
0x4700E000-0x4700E0FF  Function table (process name → entry)
0x48000000-0x48FFFFFF  Process stacks (16 × 64KB)
```

## Syscalls

| # | Name | Args | Description |
|---|------|------|-------------|
| 0 | SYS_EXIT | — | Terminate current process |
| 1 | SYS_WRITE | char | Print character to UART |
| 2 | SYS_YIELD | — | Voluntary yield to scheduler |
| 3 | SYS_GETPID | — | Get current process ID |
| 4 | SYS_IPC_SEND | dest, value | Send message to process |
| 5 | SYS_IPC_RECV | — | Receive message (blocks if empty) |
| 6 | SYS_SPAWN | entry_addr | Create new process |
| 7 | SYS_KILL | pid | Terminate target process |
| 8 | SYS_WAIT | pid | Block until target exits |
| 9 | SYS_READ | — | Read UART char (non-blocking) |
| 10 | SYS_SVC_REGISTER | name, port | Register microkernel service |
| 11 | SYS_SVC_LOOKUP | name | Find service owner PID |
| 12 | SYS_SIGNAL | handler_addr | Register signal handler |
| 13 | SYS_KILL_SIG | pid, signal | Send signal to process |
| 14 | SYS_PIPE | — | Create pipe (returns pipe_id) |
| 15 | SYS_PIPE_WRITE | pipe_id, byte | Write byte to pipe |
| 16 | SYS_PIPE_READ | pipe_id | Read byte from pipe |

## Shell Commands (152 total)

```
[0] fjsh> help
Commands:
  help ps mem echo uptime clear
  peek poke dump ticks sleep version
  halt whoami date history reg bench
  env export kill free fill hex dmesg
  cmp test top about log colors calc
  reboot time seq xxd rand loops ascii
  bits prime swap wc head fib fact gcd
  pow sysinfo ls touch rm cat stat
  mkdir cp mount lsfat catfat df sync
  spawn wait demo ipc nice pmap memstat
  ipcstat svclist signal pipe
```

## Hardware Verified (Radxa Dragon Q6A)

| Test | Result |
|------|--------|
| Cranelift JIT | fib(30) in **8ms** (644× vs interpreter) |
| GPIO96 blink | 5 ON/OFF cycles on real hardware |
| **MNIST MLP** (CPU) | **10/10 correct**, 0.8ms/inference |
| **MNIST MLP** (GPU) | **10/10 correct**, 25.3ms/inference (Adreno 643) |
| **ResNet18** (CPU INT8) | 1000-class ImageNet, **37 img/s** batch |
| FajarOS boot (QEMU on Q6A) | All 152 commands work, 5/5 self-tests pass |
| EL0 user process | SVC + timer preemption from unprivileged level |
| Per-process page tables | TTBR0 switch verified with `pmap` command |

## Stats

```
Kernel LOC:     5,016 (100% Fajar Lang)
Shell commands: 160
Syscalls:       17
Max PIDs:       16
Priority levels: 4 (IDLE, NORMAL, HIGH, REALTIME)
Services:       3 (UART, Timer, Memory)
Pipes:          8 × 4KB
IPC queue:      8 messages per process
Signals:        SIGTERM(1), SIGKILL(9), SIGCHLD(17)
Page tables:    Per-process L0, shared kernel L1
Exception frame: 272 bytes (x0-x30 + SP + ELR + SPSR)
```

## Source

- **Compiler:** [github.com/fajarkraton/fajar-lang](https://github.com/fajarkraton/fajar-lang) (v3.2.0)
- **OS:** [github.com/fajarkraton/fajar-os](https://github.com/fajarkraton/fajar-os) (v3.2)
- **Hardware:** [Radxa Dragon Q6A](https://radxa.com/products/dragon/q6a/) (Qualcomm QCS6490)

## License

MIT

## Author

Fajar (PrimeCore.id) + Claude Opus 4.6
