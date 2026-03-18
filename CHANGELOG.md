# Changelog — FajarOS

## v3.1.0 "Surya Rising" (2026-03-19)

### New Features
- **16-PID priority scheduler** — 4 levels (IDLE/NORMAL/HIGH/REALTIME), round-robin within same priority
- **Per-process page tables** — unique L0 per process, TTBR0 switch + TLB flush on context switch
- **Service registry** — 16-slot microkernel service table, `svc_register()`/`svc_lookup()`, SYS_SVC_REGISTER(10)/SYS_SVC_LOOKUP(11)
- **Signals** — SIGTERM(1), SIGKILL(9), SIGCHLD(17), per-process handlers, SYS_SIGNAL(12)/SYS_KILL_SIG(13)
- **Pipes** — 8 slots × 4KB circular buffer, SYS_PIPE(14)/SYS_PIPE_WRITE(15)/SYS_PIPE_READ(16)
- **IPC v2** — 8-message queues (was 4), total_sent tracking
- **Idle process** — PID 15, WFI loop, PRIO_IDLE
- **Data/instruction abort handlers** — page faults kill process with address display
- **Process names** — stored in proc_table, displayed in ps/kill/spawn/top
- **Per-process tick counting** — CPU% in `top` command
- **Interactive shell** — Ctrl+C, up-arrow history, tab completion, `[PID] fjsh>` prompt

### New Commands (+14)
`nice`, `pmap`, `memstat`, `ipcstat`, `svclist`, `signal`, `pipe` + tab completion

### Syscalls: 10 → 17
SYS_SVC_REGISTER(10), SYS_SVC_LOOKUP(11), SYS_SIGNAL(12), SYS_KILL_SIG(13), SYS_PIPE(14), SYS_PIPE_WRITE(15), SYS_PIPE_READ(16)

### Improvements
- Scheduler: 8 → 16 PIDs
- Kill: supports PID 1-14 (was 1-2), validates state, sets TERMINATED
- putc calls: 342 → 137 (-60%), replaced with `println()`/`print()` strings
- Kernel priority: NORMAL (was HIGH) for fair CPU sharing

### Hardware Verified (Radxa Dragon Q6A)
- **MNIST MLP**: 10/10 correct, CPU 0.8ms, GPU 25.3ms per inference
- **ResNet18**: INT8 CPU 27ms/img batch (37 img/s), 1000-class ImageNet
- **FajarOS on Q6A**: boots in QEMU, all 152 commands work, 5/5 self-tests pass
- **fj JIT**: fib(30) = 8ms (644× speedup)

### Stats
```
LOC:        4,805 (was 4,022, +19%)
Commands:   152 (was 65)
Syscalls:   17 (was 10)
PIDs:       16 (was 8)
Pipes:      8 × 4KB
Services:   3 registered
```

---

## v3.0.0 "Surya" (2026-03-18)

### Major Features
- **EL0 User Space** — user processes at unprivileged ARM64 exception level
- **MMU** — 4-level page tables, 2MB blocks, data+instruction caches
- **10 Syscalls** — EXIT, WRITE, YIELD, GETPID, IPC_SEND, IPC_RECV, SPAWN, KILL, WAIT, READ
- **IPC** — 4-message circular queue per process, blocking recv
- **Preemptive Scheduler** — 100 Hz timer (GICv3 PPI 30), round-robin, 8 PIDs
- **Interactive Shell** — spawn/wait/kill/ps + 65 commands
- **String Literals** — `println("text")` in @kernel via .rodata

### Hardware Verified (Radxa Dragon Q6A)
- Cranelift JIT: fib(40) in 0.65s (1,246× speedup)
- GPIO96: real hardware toggle
- QNN CPU: 24ms/inference (MNIST INT8)
- QNN GPU: 262ms/inference (MNIST FP32, Adreno 643 OpenCL 3.0)
- FajarOS: boots and runs all features on QEMU-on-Q6A

### Bug Fixes
- Sequential SVC calls (direct ELR_EL1 advance)
- SPSR_EL1 save/restore (272-byte exception frames)
- Void return in bare-metal (return iconst(0))
- MMU AP bits (WXN clear, user code in separate 2MB block)
- GPU inference (use float32 DLC instead of INT8)

### Architecture
- 100% Fajar Lang (no C, no Rust in the OS)
- @kernel/@device/@safe compiler-enforced contexts
- ARM64 aarch64-none bare-metal target
- QEMU virt (Cortex-A76) + Radxa Dragon Q6A (QCS6490)
