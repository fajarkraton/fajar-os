# FajarOS v3.1 Implementation Plan — "Surya Rising"

> **Context:** FajarOS v3.0 has preemptive multitasking, 6 syscalls, IPC, 63+ shell commands.
> **Date:** 2026-03-18
> **Goal:** Transform FajarOS from "OS demo" to "usable OS" with interactive process management,
> memory isolation, improved IPC, and real hardware deployment.

---

## Execution Order

```
Phase 1: Shell Integration + More Syscalls     ← START HERE (immediate value)
Phase 2: Memory Isolation (MMU per-process)     ← real OS separation
Phase 3: Improved IPC + Service Registry        ← microkernel foundation
Phase 4: Compiler asm! in() Fix                 ← cleanup
Phase 5: Q6A Hardware Deployment                ← when board online
Phase 6: Documentation + Demo                   ← showcase
```

---

## Phase 1: Shell Integration + More Syscalls (Sprint G)

**Priority:** HIGH — immediate visible impact
**Repo:** fajar-os + fajar-lang (minor compiler additions)
**Estimated:** 4-6 hours
**Depends on:** Sprint C (syscalls) — DONE

### Background
Currently the shell (PID 0) runs in kernel context and processes are spawned
only via `demo` and `ipc` commands with hardcoded functions. Users should be
able to interactively spawn, list, and manage processes from the shell.

### Architecture
```
Shell (PID 0, @kernel)
  │
  ├── "spawn a" → SYS_SPAWN → create PID N with entry=process_a
  ├── "spawn b" → SYS_SPAWN → create PID N with entry=process_b
  ├── "spawn c" → SYS_SPAWN → create PID N with entry=process_c
  ├── "ps"      → show all PID states
  ├── "kill N"  → SYS_KILL → terminate PID N
  └── "wait N"  → block shell until PID N exits
```

### Tasks
| # | Task | Detail | Status |
|---|------|--------|--------|
| G.1 | **SYS_SPAWN syscall (6)** | `svc(6, entry_addr, 0) → pid`. Kernel finds free PID slot, calls `create_process(pid, entry)`, returns new PID. | [ ] |
| G.2 | **SYS_KILL syscall (7)** | `svc(7, pid, 0) → 0/-1`. Mark target process TERMINATED if exists. Cannot kill PID 0. | [ ] |
| G.3 | **SYS_WAIT syscall (8)** | `svc(8, pid, 0) → exit_code`. Block calling process until target PID terminates. Uses BLOCKED state. | [ ] |
| G.4 | **SYS_READ syscall (9)** | `svc(9, 0, 0) → char`. Read single character from UART (fd=0=stdin). Returns -1 if no char available. Non-blocking. | [ ] |
| G.5 | **Process name registry** | Store 8-char process name at proc_table[pid]+56 (reserved area). Used by `ps` to show names. | [ ] |
| G.6 | **`spawn` shell command** | `spawn a` → lookup process_a entry via function table → SYS_SPAWN. Print "Spawned PID N". | [ ] |
| G.7 | **`wait` shell command** | `wait 3` → block shell until PID 3 exits. Shell resumes with "PID 3 exited". | [ ] |
| G.8 | **Improve `ps` output** | Show PID, STATE, NAME, TICKS (time slices used). Format: `PID  STATE  TICKS  NAME`. | [ ] |
| G.9 | **Improve `kill` command** | `kill N` → use SYS_KILL syscall. Print state change. Prevent killing PID 0. | [ ] |
| G.10 | **Process function table** | Map names to entry addresses: `{"a": process_a, "b": process_b, "c": process_c, "sender": process_sender, "receiver": process_receiver}`. Store at fixed kernel address. | [ ] |
| G.11 | **Interactive demo** | User types: `spawn a`, `spawn b`, `ps`, `kill 1`, `spawn c`, `ps`, `wait 3` — all work correctly. | [ ] |
| G.12 | **UART RX interrupt** | Enable PL011 RX interrupt (IRQ 33 on QEMU virt) via GICv3. Shell wakes on keypress instead of polling in WFI loop. | [ ] |

### Technical Notes
- **Process function table:** Since `fn_addr()` requires compile-time function names, create a lookup table at 0x47003000 with entries: `[name_hash, entry_addr]`. Shell hashes the command argument and finds the matching entry.
- **SYS_READ:** PL011 UART RX is at `UARTDR (base+0x00)`. Check `UARTFR (base+0x18)` bit 4 (RXFE = RX FIFO empty). If empty, return -1 (non-blocking). Future: interrupt-driven with ring buffer.
- **SYS_WAIT:** Add `wait_pid` field to process table at offset 56. When SYS_WAIT is called, set current process state to BLOCKED and store target PID. Timer IRQ handler checks: if target PID is TERMINATED, wake the waiting process.

### Success Criteria
- `spawn a` creates a process that prints 'A', `ps` shows it RUNNING
- `kill N` terminates any process, `ps` shows DED
- `wait N` blocks shell until process exits
- Interactive workflow: spawn → ps → kill → ps works smoothly

---

## Phase 2: Memory Isolation — Per-Process Page Tables (Sprint H)

**Priority:** HIGH — transforms demo OS into real OS
**Repo:** fajar-os + fajar-lang (MMU builtins)
**Estimated:** 8-12 hours
**Depends on:** Phase 1

### Background
Currently all processes share the same address space. Any process can read/write
any memory, including kernel data structures. Real OS needs per-process page tables
with TTBR0 switching on context switch.

ARM64 provides dual translation tables:
- **TTBR0_EL1:** User-space (lower VA, 0x0000_0000 — 0x0000_FFFF_FFFF_FFFF)
- **TTBR1_EL1:** Kernel-space (upper VA, 0xFFFF_0000_0000_0000+)

For FajarOS (runs at EL1, no EL0 yet), we use TTBR0 only with separate page tables.

### Architecture
```
Process A (TTBR0 = PT_A)         Process B (TTBR0 = PT_B)
┌─────────────────────┐         ┌─────────────────────┐
│ 0x0000_0000: code   │         │ 0x0000_0000: code   │ (different physical)
│ 0x0010_0000: stack  │         │ 0x0010_0000: stack  │ (different physical)
│ 0x4000_0000: kernel │ ←shared→│ 0x4000_0000: kernel │ (same physical)
│ 0x0800_0000: MMIO   │ ←shared→│ 0x0800_0000: MMIO   │ (same physical)
└─────────────────────┘         └─────────────────────┘

Context Switch:
1. Save current TTBR0 in process table
2. Load new process TTBR0
3. TLBI VMALLE1 (invalidate TLB)
4. ISB (barrier)
5. Restore registers + eret
```

### Tasks
| # | Task | Detail | Status |
|---|------|--------|--------|
| H.1 | **Enable MMU in kernel_main** | Call `init_mmu()` from kernel/mm/mmu.fj. Map full MMIO (0x00-0x10), kernel (0x40-0x50), process stacks (0x48-0x4C). | [ ] |
| H.2 | **Per-process L0 table allocation** | `create_process()` allocates a new L0 page table (4KB) for each process via `pt_alloc()`. | [ ] |
| H.3 | **Map kernel region in all page tables** | Every process page table maps 0x40000000-0x4FFFFFFF (kernel) and 0x00000000-0x0FFFFFFF (MMIO) identically. | [ ] |
| H.4 | **Map unique stack region per process** | Each process gets its own stack mapping at 0x48000000 + pid*0x10000 (64KB). | [ ] |
| H.5 | **TTBR0 switch in context switch** | Compiler builtin `sched_set_ttbr0(table_addr)` that does `msr TTBR0_EL1, x0; isb`. | [ ] |
| H.6 | **TLB invalidation on switch** | After TTBR0 change: `TLBI VMALLE1IS; DSB ISH; ISB`. Add as assembly stub. | [ ] |
| H.7 | **Store TTBR0 in process table** | Process table field at offset 64: TTBR0 address. Save/restore during context switch. | [ ] |
| H.8 | **Update vector stub for TTBR0** | `__exc_irq_cur` saves TTBR0 before switch, loads new TTBR0 after switch. | [ ] |
| H.9 | **Test: two processes with separate page tables** | Process A writes to 0x100000, Process B writes to 0x100000 — different physical memory. Verify no corruption. | [ ] |
| H.10 | **Protection fault on kernel write** | User process writing to kernel address (0x47000000) should cause data abort, not corrupt kernel. | [ ] |

### Technical Notes (from research)
- **TTBR0 vs TTBR1:** ARM64 uses bit 55 to select: `0xxxx...` → TTBR0 (user), `1xxxx...` → TTBR1 (kernel). Since FajarOS runs at EL1 and addresses are below 0x50000000, everything uses TTBR0.
- **TLB flush:** `TLBI VMALLE1IS` invalidates all TLB entries for EL1. Must be followed by `DSB ISH; ISB`.
- **Page table format:** 4KB granule, 48-bit VA, 4-level walk. L0[47:39], L1[38:30], L2[29:21], L3[20:12]. Each entry is 8 bytes.
- **Compiler builtin needed:** `sched_set_ttbr0(addr)` → assembly stub: `msr TTBR0_EL1, x0; tlbi vmalle1is; dsb ish; isb; ret`.

### Success Criteria
- MMU enabled, kernel runs with virtual addresses
- Each process has its own page table
- Context switch includes TTBR0 swap + TLB flush
- One process cannot access another's stack memory

---

## Phase 3: Improved IPC + Service Registry (Sprint I)

**Priority:** MEDIUM — microkernel foundation
**Repo:** fajar-os
**Estimated:** 4-5 hours
**Depends on:** Phase 1

### Background
Current IPC has a single-slot mailbox per process. Real microkernel needs
multi-message queues, request-reply patterns, and named service discovery.

### Architecture
```
Service Registry (kernel)
┌──────────────────────────────────────────┐
│ Name          PID    Port                │
│ "uart"        10     1                   │
│ "fs"          11     2                   │
│ "net"         12     3                   │
└──────────────────────────────────────────┘

Client Process                 Server Process (e.g., UART driver)
┌─────────────┐               ┌───────────────────┐
│ svc(IPC_CALL,│──message───→ │ svc(IPC_RECV)     │
│   "uart",   │              │   handle request   │
│   data)     │←──reply────  │ svc(IPC_REPLY,data)│
└─────────────┘               └───────────────────┘
```

### Tasks
| # | Task | Detail | Status |
|---|------|--------|--------|
| I.1 | **Multi-message queue** | Replace single-slot mailbox with 8-message circular buffer per process (8 × 32 bytes = 256 bytes per queue). Head/tail pointers. | [ ] |
| I.2 | **SYS_IPC_REPLY syscall (10)** | Reply to the process that sent the last message. Sender blocks until reply arrives (synchronous RPC). | [ ] |
| I.3 | **SYS_IPC_CALL syscall (11)** | Send + block for reply in one syscall (synchronous IPC). `svc(11, dest_pid, msg) → reply`. | [ ] |
| I.4 | **Service registry** | Kernel table at 0x47004000: 16 entries × 32 bytes (name[16] + pid + port). | [ ] |
| I.5 | **SYS_SVC_REGISTER syscall (12)** | `svc(12, name_ptr, port) → 0/-1`. Register current process as a named service. | [ ] |
| I.6 | **SYS_SVC_LOOKUP syscall (13)** | `svc(13, name_ptr, 0) → pid`. Find PID of a named service. | [ ] |
| I.7 | **UART server process** | Move UART output from direct putc to a service: shell sends IPC to "uart" service, which does the actual write. | [ ] |
| I.8 | **Test: request-reply IPC** | Client sends request, server processes and replies, client receives reply. Verify synchronous flow. | [ ] |
| I.9 | **Test: service discovery** | Process registers as "echo" service, another process looks up "echo" by name and sends a message. | [ ] |

### Technical Notes
- **Circular buffer:** `struct MsgQueue { msgs: [Msg; 8], head: u32, tail: u32, count: u32 }`. `send` writes at tail, `recv` reads from head. Queue full → sender blocks.
- **Synchronous IPC:** IPC_CALL = send + block for reply. IPC_REPLY = wake caller + copy reply data. This is the L4/seL4 pattern.
- **Service registry:** Simple hash-based lookup. Name is 16 bytes max, hashed to 4 bytes for fast comparison.

### Success Criteria
- Multiple messages can be queued per process
- Request-reply IPC works synchronously
- Services register and are discoverable by name
- UART output works as a service (not direct kernel call)

---

## Phase 4: Compiler asm! in() Fix (Sprint J)

**Priority:** LOW — workaround exists (svc() builtin)
**Repo:** fajar-lang
**Estimated:** 3-4 hours
**Depends on:** None

### Background
`asm!("svc #0", in("x0") val)` generates incorrect code in AOT mode.
The `in()` register constraint doesn't properly load the value before
the instruction. Workaround: use `svc()` builtin which is an assembly stub.

### Tasks
| # | Task | Detail | Status |
|---|------|--------|--------|
| J.1 | **Reproduce with minimal test** | Create AOT test: `asm!("mov x1, x0", in("x0") 42, out("x1") result)`. Check if result == 42. | [ ] |
| J.2 | **Trace Cranelift IR for asm! in()** | Inspect how `compile_asm` in `asm.rs` generates IR for register constraints. | [ ] |
| J.3 | **Compare with working stubs** | Assembly stubs work by using `bl` (branch-and-link) which properly passes x0-x7. The `asm!` path skips the ABI. | [ ] |
| J.4 | **Fix asm codegen for in() constraint** | Ensure Cranelift IR loads the value into the specified register BEFORE the inline assembly block. | [ ] |
| J.5 | **Fix asm codegen for out() constraint** | Ensure the output register value is captured into a Cranelift variable AFTER the inline assembly block. | [ ] |
| J.6 | **Test: asm! with in/out works in AOT** | `asm!("add x0, x0, #1", inout("x0") val)` → val + 1. Verify in both JIT and AOT. | [ ] |
| J.7 | **Test: svc with in() constraint** | `asm!("svc #0", in("x0") 1, in("x1") 67)` properly prints 'C' via SYS_WRITE. | [ ] |

### Technical Notes
- The `asm!` codegen in `src/codegen/cranelift/compile/asm.rs` currently encodes ARM64 instructions as raw u32 values. For `in()` constraints, it needs to emit Cranelift IR that moves values into specific physical registers before the assembly block.
- Cranelift's `regalloc` can handle fixed-register constraints via `AbiParam` with specific register assignments.
- Reference: Cranelift's `asm!()` support is limited; many OS projects use wrapper functions (like our `svc()` builtin) as the standard approach.

### Success Criteria
- `asm!("svc #0", in("x0") 1, in("x1") 67)` works in AOT mode
- All existing asm! tests pass
- No regressions in JIT mode

---

## Phase 5: Q6A Hardware Deployment (Sprint K)

**Priority:** HIGH (when hardware available) — BLOCKED
**Repo:** fajar-os + fajar-lang
**Estimated:** 2-3 days
**Depends on:** Phase 1-3, Q6A board powered on

### Background
FajarOS currently runs on QEMU. Deploying to the real Radxa Dragon Q6A
(Qualcomm QCS6490) requires adjusting MMIO addresses, boot sequence,
and driver initialization for real hardware.

### Tasks
| # | Task | Detail | Status |
|---|------|--------|--------|
| K.1 | **Power on Q6A** | Connect power, verify SSH access via WiFi or Ethernet. | [ ] |
| K.2 | **Cross-compile FajarOS** | `fj build --target aarch64 --no-std` with Q6A MMIO addresses instead of QEMU virt. | [ ] |
| K.3 | **Hardware address adaptation** | UART: QUP GENI at 0x0A8C_0000 (not PL011 at 0x0900_0000). GIC: 0x1780_0000/0x17A0_0000 (not 0x0800_0000). | [ ] |
| K.4 | **UEFI boot on Q6A** | Build EFI binary, place on NVMe ESP partition, boot from UEFI menu. | [ ] |
| K.5 | **Serial console** | Connect USB-serial to UART5 (pin 8/10), verify kernel output. | [ ] |
| K.6 | **GPIO blink test** | Blink LED on GPIO96 (pin 7) from FajarOS shell. | [ ] |
| K.7 | **Timer on real SoC** | Verify CNTFRQ_EL0 on QCS6490 (likely 19.2 MHz Qualcomm timer). Adjust quantum ticks. | [ ] |
| K.8 | **Complete v2.0 "Dawn" tasks** | 18 remaining tasks that need real Q6A hardware. | [ ] |
| K.9 | **Run multitasking demo on Q6A** | `demo` command with 3 processes on real ARM64 hardware. | [ ] |
| K.10 | **NPU inference test** | Load MNIST model via QNN, run inference from FajarOS process. | [ ] |

### Technical Notes (from research)
- **QCS6490 timer:** Qualcomm SoCs typically use 19.2 MHz timer (vs QEMU 62.5 MHz). Quantum calculation: `19200000 / 100 = 192000` ticks for 10ms.
- **UEFI boot:** Q6A uses UEFI → SPI NOR → XBL → DDR → kernel. FajarOS ELF needs to be placed as EFI application on ESP partition.
- **QUP UART:** Qualcomm's QUP (GENI) serial engine is register-incompatible with PL011. Need conditional compilation or runtime detection.
- **GICv3 addresses:** QCS6490 uses 0x17800000 (GICD) and 0x17A00000 (GICR), vs QEMU's 0x08000000/0x080A0000.

### Success Criteria
- FajarOS boots on real Q6A hardware
- Shell works via serial console
- GPIO blink verified on real LED
- Multitasking demo runs on real ARM64 SoC

---

## Phase 6: Documentation + Demo (Sprint L)

**Priority:** LOW — showcase
**Repo:** fajar-os + fajar-lang
**Estimated:** 1-2 days
**Depends on:** Phase 1-3

### Tasks
| # | Task | Detail | Status |
|---|------|--------|--------|
| L.1 | **BLOG_FAJAROS.md** | Technical blog post: "FajarOS: The First OS Written 100% in Fajar Lang". Architecture, syscalls, IPC, timer, context switch. | [ ] |
| L.2 | **Update CLAUDE.md** | Add FajarOS v3.0 status: multitasking, syscalls, IPC, scheduler. | [ ] |
| L.3 | **README.md for fajar-os** | Quick start guide: build, run on QEMU, commands, architecture overview. | [ ] |
| L.4 | **Demo video script** | 5-minute walkthrough: boot → shell → spawn → ps → kill → ipc → demo. | [ ] |
| L.5 | **Architecture diagram** | SVG/ASCII diagram showing kernel layers, syscall flow, IPC, scheduler. | [ ] |
| L.6 | **V30_PLAN.md update** | Mark completed sprints in FajarOS plan. Update progress percentages. | [ ] |
| L.7 | **Update memory** | Update MEMORY.md with all FajarOS progress, known issues, architecture notes. | [ ] |

### Success Criteria
- Blog post ready for publication
- README allows new developer to build and run FajarOS
- Demo video script covers all features

---

## Summary

| Phase | Sprint | Tasks | Priority | Estimated | Depends On |
|-------|--------|-------|----------|-----------|------------|
| 1 | G: Shell + Syscalls | 12 | HIGH | 4-6h | Sprint C ✓ |
| 2 | H: Memory Isolation | 10 | HIGH | 8-12h | Phase 1 |
| 3 | I: IPC + Services | 9 | MEDIUM | 4-5h | Phase 1 |
| 4 | J: asm! in() Fix | 7 | LOW | 3-4h | None |
| 5 | K: Q6A Deployment | 10 | HIGH* | 2-3 days | Phases 1-3 + HW |
| 6 | L: Documentation | 7 | LOW | 1-2 days | Phases 1-3 |
| **Total** | **6 sprints** | **55 tasks** | | | |

*K is HIGH priority but BLOCKED by hardware availability.

---

## Research Sources

- [ARM64 System Calls (SVC)](https://duetorun.com/blog/20230604/a64-svc/) — SVC handler implementation
- [ARMv-8a EL0↔EL1 Switching](https://pyjamacafe.com/posts/arm64-day1-el0-el1-switching/) — Exception level transitions
- [AArch64 MMU Programming](https://lowenware.com/blog/aarch64-mmu-programming/) — Page table setup
- [Quick and Dirty AArch64 MMU](https://dannasman.github.io/aarch64-mmu.html) — Practical MMU guide
- [Microkernel (OSDev Wiki)](http://wiki.osdev.org/Microkernel) — IPC patterns
- [Microkernels: A Modern Guide (2026)](https://thelinuxcode.com/microkernels-in-operating-systems-a-practical-modern-guide-2026/) — Service registry
- [UART RX Interrupts in Bare-Metal ARM](https://medium.com/@hielelshadday/in-embedded-system-development-handling-input-output-via-uart-is-almost-essential-52540ce24898) — Ring buffer pattern
- [ARM Paging (OSDev Wiki)](https://wiki.osdev.org/ARM_Paging) — Page table format
- [NYCU OS Lab 8: Virtual Memory](https://grasslab.github.io/NYCU_Operating_System_Capstone/labs/lab8.html) — TTBR0 context switch

---

*Plan created 2026-03-18 by Claude Opus 4.6*
*Total: 6 phases, 55 tasks, covering shell integration through hardware deployment*
