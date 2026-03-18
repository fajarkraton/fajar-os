# Changelog — FajarOS

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
