# FajarOS "Surya"

> The world's first operating system written 100% in [Fajar Lang](https://github.com/fajarkraton/fajar-lang) — where kernel safety, hardware drivers, and AI inference share one language, one type system, and one compiler.

**Target Hardware:** [Radxa Dragon Q6A](https://radxa.com/products/dragon/q6a/) (Qualcomm QCS6490)

## Architecture

```
Applications (@safe)    — Shell, REPL, AI demos
OS Services  (@safe)    — Init, VFS, TCP/IP, display, NPU daemon
HAL Drivers  (@kernel)  — GPIO, UART, I2C, SPI, NVMe, GPU, NPU
Microkernel  (@kernel)  — Boot, MMU, IPC, syscalls, scheduler
─────────────────────────────────────────────────────────
Hardware: QCS6490 (8-core ARM64, Adreno 643, Hexagon 770 NPU)
```

## Key Innovation

Fajar Lang's `@kernel` / `@device` / `@safe` context annotations are **compiler-enforced**:
- `@kernel` code cannot allocate heap or use tensor ops
- `@device` code cannot access raw pointers or hardware
- `@safe` code must go through syscalls to reach hardware

If it compiles, the isolation is guaranteed.

## Status

**Phase 1: Bare Metal Boot** — Starting

## Build

Requires [Fajar Lang compiler](https://github.com/fajarkraton/fajar-lang):

```bash
fj build kernel/boot/start.fj --target aarch64 --no-std -o fjaros.elf
```

## License

MIT
