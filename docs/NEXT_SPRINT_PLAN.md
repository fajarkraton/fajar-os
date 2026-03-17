# FajarOS Next Sprint Plan — Post-Multitasking

> **Context:** Preemptive multitasking working (3 processes, 10ms quantum, 100 ctx-switch/sec).
> **Date:** 2026-03-18
> **Goal:** Make FajarOS a usable, debuggable OS with proper process lifecycle.

---

## Sprint A: Fix AOT asm! out() Return Value Bug (Compiler)

**Priority:** HIGH — blocks uptime display, memory reads, debugging
**Repo:** fajar-lang
**Estimated:** 2-4 hours

### Root Cause
AOT-compiled builtins (e.g., `timer_count()`, `volatile_read()`) return stale/zero values
in the main function context. The Cranelift codegen doesn't properly capture x0 return
values from imported functions declared as `Linkage::Import`.

### Tasks
| # | Task | Status |
|---|------|--------|
| A.1 | Reproduce: write minimal AOT test that calls `timer_count()` and checks return | [x] |
| A.2 | Inspect Cranelift IR: verify `call` instruction has proper return value binding | [x] |
| A.3 | Compare JIT vs AOT codegen for `timer_count()` — JIT works, AOT doesn't | [x] |
| A.4 | Fix ObjectCompiler function signature for imported () -> i64 functions | [x] NOT A BUG |
| A.5 | Verify: `uptime` command shows correct time in QEMU | [x] 0→1.9→4.9s ✓ |
| A.6 | Verify: `peek` uses `volatile_read` correctly (not just `volatile_read_u32`) | [x] |

### Result: NOT A BUG
Disassembly confirmed all codegen is correct. The "stale return values" were actually
correct values — commands execute in microseconds so uptime appeared as 0.
Verified: `uptime` shows 0→1.9s→4.9s with proper delays.
**Sprint A: COMPLETE (no fix needed)**

---

## Sprint B: Process Lifecycle (FajarOS)

**Priority:** HIGH — makes demo interactive
**Repo:** fajar-os
**Estimated:** 3-4 hours

### Tasks
| # | Task | Status |
|---|------|--------|
| B.1 | Implement `exit` in process: mark TERMINATED, trigger reschedule | [ ] |
| B.2 | Update scheduler to skip TERMINATED processes | [ ] |
| B.3 | Implement `spawn <name>` shell command: create process from function name | [ ] |
| B.4 | Implement `kill <pid>` shell command: mark TERMINATED | [ ] |
| B.5 | Update `ps` command: show RUNNING/READY/TERMINATED states for PID 0-15 | [ ] |
| B.6 | Process cleanup: reclaim PID slot after TERMINATED | [ ] |
| B.7 | Add `wait <pid>` in shell: block until process terminates | [ ] |
| B.8 | Test: spawn 2 processes, kill one, verify other continues | [ ] |
| B.9 | Test: spawn, exit naturally, verify PID recycled | [ ] |
| B.10 | Demo: interactive `spawn A`, `spawn B`, `ps`, `kill 1`, `ps` | [ ] |

### Success Criteria
- `spawn A` creates process that prints 'A'
- `kill 1` terminates PID 1
- `ps` shows correct state for all processes
- Terminated PIDs can be reused

---

## Sprint C: Syscall Interface (FajarOS)

**Priority:** MEDIUM — real OS architecture
**Repo:** fajar-os
**Estimated:** 6-8 hours

### Tasks
| # | Task | Status |
|---|------|--------|
| C.1 | Define syscall table: SYS_EXIT(0), SYS_WRITE(1), SYS_YIELD(2), SYS_SPAWN(3), SYS_GETPID(4) | [ ] |
| C.2 | Implement SVC #0 handler in `fj_exception_sync` (ESR.EC = 0x15) | [ ] |
| C.3 | Syscall dispatch: x8 = syscall number, x0-x5 = arguments | [ ] |
| C.4 | SYS_EXIT: mark current process TERMINATED, reschedule | [ ] |
| C.5 | SYS_WRITE: write buffer to UART (fd=1) | [ ] |
| C.6 | SYS_YIELD: voluntarily yield to scheduler | [ ] |
| C.7 | SYS_SPAWN: create new process from entry address | [ ] |
| C.8 | SYS_GETPID: return current PID | [ ] |
| C.9 | Update process_a/b to use syscalls instead of direct UART access | [ ] |
| C.10 | Test: process makes SVC → kernel handles → returns to process | [ ] |

### Success Criteria
- Processes use SVC for I/O (not direct MMIO)
- Kernel handles syscalls correctly
- ELR_EL1 correctly returns to instruction after SVC

---

## Sprint D: IPC (Inter-Process Communication)

**Priority:** MEDIUM — microkernel foundation
**Repo:** fajar-os
**Estimated:** 4-5 hours

### Tasks
| # | Task | Status |
|---|------|--------|
| D.1 | Define IPC message struct at 0x47002000 (sender, receiver, type, 248-byte payload) | [ ] |
| D.2 | Implement `ipc_send(dest_pid, msg_ptr, len)`: copy to dest mailbox, wake if blocked | [ ] |
| D.3 | Implement `ipc_recv(buf_ptr, len)`: block until message arrives | [ ] |
| D.4 | Add SYS_IPC_SEND(5) and SYS_IPC_RECV(6) syscalls | [ ] |
| D.5 | Process state BLOCKED: skip in scheduler, wake on message | [ ] |
| D.6 | Test: process A sends "HELLO" to process B, B prints it | [ ] |
| D.7 | Test: recv blocks until send arrives (no busy-wait) | [ ] |

### Success Criteria
- Two processes exchange messages via IPC
- Blocked processes don't consume CPU
- Message delivery is reliable

---

## Sprint E: Parser Split (Compiler)

**Priority:** LOW — code quality
**Repo:** fajar-lang
**Estimated:** 1-2 hours

### Tasks
| # | Task | Status |
|---|------|--------|
| E.1 | Analyze parser/mod.rs (4,931 LOC) structure | [ ] |
| E.2 | Extract expression parsing → parser/expr.rs | [ ] |
| E.3 | Extract statement parsing → parser/stmt.rs | [ ] |
| E.4 | Keep parser/mod.rs as coordinator (parse entry, token handling) | [ ] |
| E.5 | Verify all 5,947 tests pass | [ ] |

### Success Criteria
- parser/mod.rs < 2,000 LOC
- All tests pass, clippy clean

---

## Sprint F: Q6A Hardware Testing

**Priority:** BLOCKED (hardware offline)
**Repo:** fajar-lang, fajar-os
**Estimated:** When Q6A is online

### Tasks
| # | Task | Status |
|---|------|--------|
| F.1 | Power on Q6A, verify SSH access | [ ] |
| F.2 | Cross-compile FajarOS for Q6A (aarch64-linux-gnu) | [ ] |
| F.3 | Run fajaros.elf on Q6A (userspace test) | [ ] |
| F.4 | Complete 18 remaining v2.0 "Dawn" tasks | [ ] |
| F.5 | Test GPIO blink on real hardware | [ ] |
| F.6 | Test NPU inference on real Hexagon 770 | [ ] |

---

## Execution Order

```
Sprint A (AOT bug fix)      ← START HERE
    ↓
Sprint B (process lifecycle) ← immediate value
    ↓
Sprint C (syscalls)          ← real OS architecture
    ↓
Sprint D (IPC)               ← microkernel
    ↓
Sprint E (parser split)      ← when convenient
    ↓
Sprint F (Q6A testing)       ← when hardware online
```

---

*Plan created 2026-03-18 by Claude Opus 4.6*
