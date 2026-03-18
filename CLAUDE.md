# CLAUDE.md — FajarOS "Surya" Master Reference

> Auto-loaded by Claude Code. Single source of truth for FajarOS development.

---

## 1. Project Identity

- **Project:** FajarOS (`fjOS`) — Operating system written 100% in Fajar Lang
- **Codename:** "Surya" (Indonesian: sun — evolution from "Dawn")
- **Target:** Radxa Dragon Q6A (Qualcomm QCS6490, ARM64)
- **Language:** 100% Fajar Lang (`.fj` files), inline asm only for hardware registers
- **Compiler:** `fj` from [fajar-lang](https://github.com/fajarkraton/fajar-lang) repo
- **Author:** Fajar (TaxPrime / PrimeCore.id)
- **Model:** Claude Opus 4.6 exclusively

**Vision:** *"The world's first OS where kernel, drivers, and AI share one language, one type system, and one compiler — with `@kernel`/`@device`/`@safe` enforced at compile time."*

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────┐
│  Layer 4: Applications (@safe)            apps/     │
│  Shell (fjsh), REPL, AI demo apps                   │
├─────────────────────────────────────────────────────┤
│  Layer 3: OS Services (@safe + @device)   services/ │
│  Init, VFS, TCP/IP, display, NPU daemon             │
├─────────────────────────────────────────────────────┤
│  Layer 2: HAL Drivers (@kernel)           drivers/  │
│  GPIO, UART, I2C, SPI, NVMe, GPU, NPU, Display     │
├─────────────────────────────────────────────────────┤
│  Layer 1: Microkernel (@kernel)           kernel/   │
│  Boot, MMU, IPC, syscalls, scheduler, interrupts    │
├─────────────────────────────────────────────────────┤
│  Hardware: Radxa Dragon Q6A (QCS6490)               │
│  CPU: Kryo 670 (A78+A55), GPU: Adreno 643          │
│  NPU: Hexagon 770 12TOPS, RAM: 8GB LPDDR5          │
└─────────────────────────────────────────────────────┘
```

### Context Safety Model

| Context | Allowed | Forbidden |
|---------|---------|-----------|
| `@kernel` | MMU, IRQ, raw ptr, asm! | Heap alloc, tensor ops |
| `@device` | Tensor, GPU, NPU | Raw ptr, IRQ, asm! |
| `@safe` | Standard operations | Raw ptr, hardware |

**Compiler enforces:** A `@safe` service cannot call `@kernel` functions directly — must go through syscalls.

---

## 3. Development Setup

### Prerequisites

```bash
# Fajar Lang compiler (from fajar-lang repo)
cd ~/Documents/Fajar\ Lang
cargo build --release --features native
export PATH="$PWD/target/release:$PATH"
fj --version

# Cross-compilation target
rustup target add aarch64-unknown-linux-gnu

# QEMU for testing (before hardware)
sudo apt install qemu-system-aarch64
```

### Build & Test Cycle

```bash
# Compile FajarOS kernel
fj build kernel/boot/start.fj --target aarch64 --no-std -o fjaros.elf

# Run in QEMU
qemu-system-aarch64 -M virt -cpu cortex-a78 -m 1G \
  -kernel fjaros.elf -nographic

# Deploy to Q6A
scp fjaros.elf radxa@192.168.50.94:/tmp/
ssh radxa@192.168.50.94 "sudo /tmp/fjaros.elf"  # userspace test
```

---

## 4. Target Hardware

| Component | Detail |
|-----------|--------|
| **SoC** | Qualcomm QCS6490, TSMC 6nm |
| **CPU** | Kryo 670: 1xA78@2.71GHz + 3xA78@2.4GHz + 4xA55@1.96GHz |
| **GPU** | Adreno 643 @ 812MHz, Vulkan 1.3 (Mesa Turnip) |
| **NPU** | Hexagon 770 V68, 12 TOPS INT8 |
| **RAM** | 8GB LPDDR5 @ 3200MHz |
| **Storage** | 238GB Samsung NVMe (536 MB/s read) |
| **GPIO** | 40-pin, `/dev/gpiochip4`, 27 usable pins |
| **Network** | GbE, WiFi 6, BT 5.4 |
| **Display** | HDMI 4K@30, MIPI DSI |
| **Camera** | 3x MIPI CSI |
| **Boot** | UEFI → systemd-boot |
| **SSH** | `radxa@192.168.50.94` (WiFi ehome) |

---

## 5. Current Status (v3.1)

```
Kernel LOC:     4,805 (100% Fajar Lang)
Shell commands: 152
Syscalls:       17
Max PIDs:       16
Priority levels: 4 (IDLE, NORMAL, HIGH, REALTIME)
Services:       3 (UART, Timer, Memory)
Pipes:          8 × 4KB
Signals:        SIGTERM, SIGKILL, SIGCHLD
Page tables:    Per-process TTBR0 switch
EL0:            User processes at unprivileged level
Faults:         Data/instruction abort handlers
```

### Plans
- `docs/V30_PLAN.md` — original 10-phase plan (420 tasks)
- `docs/IMPLEMENTATION_PLAN_V31.md` — v3.1 plan (6 phases, 55 tasks)
- Fajar Lang `docs/IMPLEMENTATION_PLAN_V32.md` — v3.2 plan (8 phases, 320 tasks)

---

## 6. Coding Rules

1. **100% Fajar Lang** — no C, no Rust in the OS itself
2. **`@kernel` for all kernel/driver code** — compiler prevents heap/tensor
3. **`@safe` for all services** — compiler prevents raw hardware access
4. **Inline asm only for hardware registers** — MMIO, MSR/MRS, barriers
5. **No `unsafe` equivalent** — context annotations replace unsafe blocks
6. **All errors are values** — no panics in kernel (return error codes)
7. **Max 50 lines per function** — keep kernel code auditable
8. **Tests before implementation** — TDD always

---

## 7. Repository Structure

```
fajar-os/
├── CLAUDE.md              ← THIS FILE
├── docs/
│   ├── V30_PLAN.md        ← Master plan (420 tasks)
│   ├── V30_WORKFLOW.md    ← Sprint cycle
│   ├── V30_RULES.md       ← Kernel safety rules
│   └── V30_SKILLS.md      ← Technical patterns
├── kernel/
│   ├── boot/              ← Stage 1 boot, EL1 setup, stack init
│   ├── mm/                ← MMU, page tables, physical allocator
│   ├── ipc/               ← Message passing, shared memory
│   └── syscall/           ← Syscall table, trap handler
├── drivers/
│   ├── gpio/              ← TLMM GPIO controller
│   ├── uart/              ← GENI UART (QUP)
│   ├── i2c/               ← GENI I2C
│   ├── spi/               ← GENI SPI
│   ├── nvme/              ← PCIe NVMe driver
│   ├── gpu/               ← Adreno 643 compute
│   ├── npu/               ← Hexagon 770 via FastRPC
│   ├── display/           ← HDMI + DSI output
│   └── network/           ← WiFi + Ethernet
├── services/
│   ├── init/              ← PID 1, service manager
│   ├── vfs/               ← Virtual filesystem
│   ├── net/               ← TCP/IP stack
│   ├── display/           ← Compositor
│   └── ai/                ← NPU inference daemon
├── apps/
│   ├── shell/             ← fjsh (FajarOS shell)
│   ├── repl/              ← Fajar Lang REPL
│   └── demo/              ← Demo applications
├── stdlib/                ← OS-specific Fajar Lang stdlib
├── tests/                 ← QEMU + hardware-in-loop tests
└── examples/              ← Example programs
```
