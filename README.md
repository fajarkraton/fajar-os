# FajarOS v3.0 "Surya"

> The world's first operating system written 100% in [Fajar Lang](https://github.com/fajarkraton/fajar-lang) — where kernel safety, hardware drivers, and AI inference share one language, one type system, and one compiler.

**Target Hardware:** [Radxa Dragon Q6A](https://radxa.com/products/dragon/q6a/) (Qualcomm QCS6490)

## Quick Start

```bash
# Build compiler
cd ~/fajar-lang && cargo build --release --features native

# Build kernel
cd ~/fajar-os
fj build kernel/boot/start.fj --target aarch64 --no-std -o fajaros.elf

# Run on QEMU
qemu-system-aarch64 -M virt,gic-version=3 -cpu cortex-a76 -m 512M \
  -kernel fajaros.elf -device loader,file=fat.img,addr=0x44000000 -nographic
```

## Features

| Feature | Status |
|---------|--------|
| MMU (4-level page tables, 2MB blocks, caches ON) | ✅ |
| Preemptive scheduling (100 Hz timer, round-robin, 8 PIDs) | ✅ |
| 10 syscalls via `svc()` builtin | ✅ |
| IPC (4-message circular queue, blocking recv) | ✅ |
| Interactive shell (65+ commands) | ✅ |
| String literals in @kernel (`println("text")`) | ✅ |
| FAT filesystem + VirtIO block device | ✅ |
| Process lifecycle (spawn/wait/kill/ps) | ✅ |
| GICv3 interrupt controller | ✅ |
| SPSR_EL1 save/restore (272-byte frames) | ✅ |
| **EL0 user space** (unprivileged processes) | ✅ |
| MMU AP split (kernel EL1 / user EL0) | ✅ |

## Demo

```
FajarOS v3.0 Surya

fjsh> version
FajarOS v3.0 Surya
Kernel: Fajar Lang
Arch: aarch64 EL1

fjsh> spawn a        → AAAA...(200×) → DEAD
fjsh> spawn c        → CCCC...(100× via SVC) → DEAD
fjsh> spawn r; spawn s → ssssssssss STUV (IPC!)
fjsh> wait 1         → PID 1 done
fjsh> ticks          → 11.3s (1111 IRQs)
```

## Architecture

```
EL0 (User, unprivileged)            EL1 (Kernel, privileged)
┌──────────────────┐               ┌──────────────────┐
│ svc(1, 'U', 0)   │──SVC #0────→ │ SYS_WRITE: putc  │
│ svc(4, dest, msg) │──SVC #0────→ │ SYS_IPC_SEND     │
│ svc(0, 0, 0)     │──SVC #0────→ │ SYS_EXIT: term    │
└──────────────────┘  ←──eret──── └──────────────────┘
         ↑ Timer IRQ (EL0→EL1)
         │ sched_switch() → round-robin → context switch
```

## Shell Commands

```
fjsh> help
Commands:
  help ps mem echo uptime clear
  peek poke dump ticks sleep version
  halt whoami date history reg bench
  env export kill free fill hex dmesg
  spawn wait demo ipc ... (65+ total)
```

## Syscalls

| # | Name | Args | Description |
|---|------|------|-------------|
| 0 | EXIT | — | Terminate process |
| 1 | WRITE | char | Print character to UART |
| 2 | YIELD | — | Voluntary yield |
| 3 | GETPID | — | Get current PID |
| 4 | IPC_SEND | dest, value | Send message |
| 5 | IPC_RECV | — | Receive (blocks) |
| 6 | SPAWN | entry | Create process |
| 7 | KILL | pid | Terminate process |
| 8 | WAIT | pid | Wait for exit |
| 9 | READ | — | Read UART char |

## Hardware Verified (Radxa Dragon Q6A)

| Test | Result |
|------|--------|
| Cranelift JIT | fib(40) in **0.65s** (1,246× vs interpreter) |
| GPIO96 blink | 5 ON/OFF cycles on real hardware |
| QNN CPU inference | MNIST in **24ms** (42 inf/sec) |
| QNN GPU inference | MNIST in **262ms** (FP32 on Adreno 643) |
| FajarOS boot (QEMU) | MMU→GIC→Timer→Shell, all features |
| EL0 user process | SVC from unprivileged level |

## Source

- **Compiler:** [github.com/fajarkraton/fajar-lang](https://github.com/fajarkraton/fajar-lang)
- **OS:** [github.com/fajarkraton/fajar-os](https://github.com/fajarkraton/fajar-os)

## License

MIT

## Author

Fajar (PrimeCore.id) + Claude Opus 4.6
