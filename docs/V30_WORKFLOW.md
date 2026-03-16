# Workflow — FajarOS V3.0 "Surya"

> Target: Production-ready operating system for Radxa Dragon Q6A (QCS6490)
> Architecture: Microkernel + HAL drivers + OS services + AI subsystem
> Language: 100% Fajar Lang (inline asm for hardware registers only)
> Timeline: 10 phases, 42 sprints, 420 tasks
> Baseline: Fajar Lang v1.0.0 (3,392 tests, ~194K LOC, 185 files)
> Created: 2026-03-12

---

## 1. Development Philosophy

```
CORRECTNESS > SAFETY > ISOLATION > PERFORMANCE

"If it compiles in FajarOS, kernel and AI code cannot interfere with each other —
the compiler guarantees what traditional OSes enforce with hardware rings alone."
```

### Core Principles (FajarOS)

1. **CORRECTNESS first** — no undefined behavior, no kernel panics from logic bugs
2. **CONTEXT ISOLATION always** — `@kernel` code cannot touch tensors; `@device` code cannot touch IRQs; the compiler enforces this at build time, not at runtime
3. **QEMU before hardware** — every feature works in QEMU aarch64 virt before touching real Dragon Q6A silicon
4. **NO LIBC** — the entire runtime is Fajar Lang; no glibc, no musl, no external C dependencies
5. **INLINE ASM is an island** — `asm!()` blocks are the only non-Fajar code; they must be minimized, documented, and wrapped in safe abstractions
6. **BOOT OR BUST** — every commit that changes kernel code must produce a QEMU-bootable ELF; broken boot = broken build
7. **HARDWARE IS TRUTH** — QEMU tests establish correctness; Dragon Q6A tests establish reality; both must pass

### Context Rules (Compiler-Enforced)

```
@kernel — CAN: asm!, volatile I/O, raw pointers, page tables, IRQ, kernel_alloc
          CANNOT: heap alloc (malloc), tensor ops, network I/O, filesystem
          COMPILED TO: EL1 privileged aarch64

@device — CAN: tensor ops, GPU dispatch, NPU inference, DMA buffers
          CANNOT: raw pointers, IRQ, syscall, volatile I/O, asm!
          COMPILED TO: EL0 aarch64 + compute dispatch

@safe   — CAN: standard ops, heap, strings, collections, syscalls
          CANNOT: raw pointers, IRQ, volatile I/O, direct tensor, asm!
          COMPILED TO: EL0 aarch64, syscall for privileged ops
```

---

## 2. Sprint Cycle (1 week per sprint)

```
┌──────────────────────────────────────────────────────────────────┐
│                    SPRINT CYCLE (1 week)                           │
│                                                                    │
│   ┌─ PLAN ──→ DESIGN ──→ TEST ──→ IMPL ──→ VERIFY ──→ MERGE ─┐  │
│   │                                                              │  │
│   │  1. Read V30_PLAN.md → find current sprint                   │  │
│   │  2. Read V30_SKILLS.md → get implementation patterns         │  │
│   │     (aarch64 boot, MMU, GIC, QUP, TLMM, FastRPC)            │  │
│   │  3. Design public interface (@kernel fn sigs, structs)       │  │
│   │  4. Write tests FIRST (RED phase)                            │  │
│   │     • Host unit test (logic correctness)                     │  │
│   │     • QEMU integration test (boot + serial verify)           │  │
│   │  5. Implement minimally (GREEN phase)                        │  │
│   │  6. Verify on host: cargo test + clippy + fmt                │  │
│   │  7. Verify on QEMU: boot ELF → check serial output          │  │
│   │  8. Verify on Q6A: (Phase 3+ only, hardware-in-loop)        │  │
│   │  9. Update V30_PLAN.md, mark tasks [x], commit              │  │
│   │                                                              │  │
│   └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│   End of Sprint:                                                   │
│   • ALL host tests pass (including 3,392 baseline)                │
│   • ALL QEMU tests pass (kernel boots, expected serial output)    │
│   • Clippy zero warnings                                          │
│   • Formatted (cargo fmt)                                         │
│   • Tasks marked [x] in V30_PLAN.md                               │
│   • At least 5 new tests per sprint                               │
│   • Feature-gated code compiles with AND without feature flag     │
│   • Kernel ELF boots in QEMU (if kernel code changed)             │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Sprint Step Details

#### Step 1: Plan

```bash
# Read the plan, find current sprint
cat docs/V30_PLAN.md | grep -A 20 "Sprint N:"

# Read implementation patterns for the sprint domain
cat docs/V30_SKILLS.md | grep -A 50 "<domain>"
```

#### Step 2: Design

Define the public interface before writing any code. For kernel code, this means:

```fajar
// Example: designing a GIC driver (Sprint 8)
@kernel fn gic_init()
@kernel fn gic_enable_irq(irq_num: u32, priority: u8)
@kernel fn gic_disable_irq(irq_num: u32)
@kernel fn gic_ack_irq() -> u32
@kernel fn gic_eoi(irq_num: u32)
```

Every function must specify its context annotation. If a function has no annotation, it defaults to `@safe`.

#### Step 3: Test (RED phase)

Write three categories of tests:

```
Host Unit Test    → Tests logic on x86_64 host (fast, no hardware needed)
QEMU Test         → Tests real aarch64 execution in emulated environment
Hardware Test     → Tests on real Dragon Q6A (Phase 3+ only)
```

#### Step 4: Implement (GREEN phase)

Write the minimum code to make tests pass. For kernel code:

- Use `asm!()` only for hardware register access
- Wrap every `asm!()` block in a safe `@kernel fn`
- Use `volatile_read`/`volatile_write` for MMIO
- Use `// SAFETY:` comments on every `asm!()` block

#### Step 5: Verify

```bash
# Host verification (MUST pass every commit)
cargo test && cargo clippy -- -D warnings && cargo fmt -- --check

# QEMU verification (MUST pass when kernel code changes)
fj build --target aarch64-none os/kernel.fj
qemu-system-aarch64 -M virt -cpu cortex-a76 -m 4G \
  -kernel fajaros.elf -nographic -serial mon:stdio \
  -d int,cpu_reset 2>&1 | head -50

# Hardware verification (Phase 3+, when available)
# See Section 6: Hardware Testing Protocol
```

#### Step 6: Merge

```bash
# Commit with conventional format
git commit -m "feat(kernel): implement GICv3 interrupt controller"

# Mark task complete
# Edit docs/V30_PLAN.md: change [ ] to [x]
```

---

## 3. TDD for OS Code

Operating system code requires a layered testing strategy because kernel code cannot run in a standard test harness. FajarOS uses three testing tiers:

### Tier 1: Host Unit Tests (x86_64)

**When:** Every task, every commit.
**What:** Test logic, data structures, algorithms — anything that does not require aarch64 hardware or MMIO.
**How:** Standard `cargo test` with mock hardware.

```
Target: x86_64 host
Speed: < 1 second per test
Coverage: All pure logic (page table math, scheduler queues, IPC message format)
Runner: cargo test
```

Examples of what to test on host:

- Page table entry bit manipulation (no actual MMU needed)
- Scheduler ready queue ordering (no actual context switch)
- IPC message serialization/deserialization
- FAT32 cluster chain parsing (against byte arrays)
- TCP state machine transitions (no actual network)
- ELF header generation

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn page_entry_sets_valid_bit() {
        let entry = PageTableEntry::new(0x4000_0000, PageFlags::VALID | PageFlags::AF);
        assert!(entry.is_valid());
        assert_eq!(entry.physical_addr(), 0x4000_0000);
    }

    #[test]
    fn scheduler_round_robin_cycles_three_processes() {
        let mut sched = Scheduler::new();
        sched.add(Process::new(1, Priority::Normal));
        sched.add(Process::new(2, Priority::Normal));
        sched.add(Process::new(3, Priority::Normal));
        assert_eq!(sched.next().unwrap().pid, 1);
        assert_eq!(sched.next().unwrap().pid, 2);
        assert_eq!(sched.next().unwrap().pid, 3);
        assert_eq!(sched.next().unwrap().pid, 1); // wraps around
    }
}
```

### Tier 2: QEMU Integration Tests (aarch64 emulated)

**When:** Every sprint, every kernel-touching commit.
**What:** Test that compiled kernel code actually boots, runs, and produces expected serial output.
**How:** Compile to aarch64 ELF, boot in QEMU, capture serial output, assert expected strings.

```
Target: QEMU aarch64 virt machine
Speed: 2-10 seconds per test (boot + run + check)
Coverage: Boot sequence, MMIO, interrupts, MMU, context switch, IPC
Runner: Custom test harness (see Section 4: QEMU Development Loop)
```

QEMU test pattern:

```bash
#!/bin/bash
# test_qemu_boot.sh — verify kernel boots and prints banner
set -e

# Compile kernel
fj build --target aarch64-none os/kernel.fj -o fajaros.elf

# Boot in QEMU with 5-second timeout, capture serial output
timeout 5 qemu-system-aarch64 \
  -M virt -cpu cortex-a76 -m 4G \
  -kernel fajaros.elf \
  -nographic -serial mon:stdio \
  2>/dev/null || true > /tmp/qemu_output.txt

# Assert expected output
grep -q "FajarOS v3.0 Surya" /tmp/qemu_output.txt || {
    echo "FAIL: boot banner not found"
    cat /tmp/qemu_output.txt
    exit 1
}

echo "PASS: kernel boots and prints banner"
```

### Tier 3: Hardware-in-Loop Tests (Dragon Q6A)

**When:** Phase 3+ (after HAL drivers exist), before phase gate sign-off.
**What:** Test that drivers work on real QCS6490 hardware.
**How:** Flash kernel to Q6A, connect USB serial, run test sequence, verify output.

```
Target: Radxa Dragon Q6A (real hardware)
Speed: 30-60 seconds per test (flash + boot + run + check)
Coverage: GPIO, UART, SPI, I2C, PCIe, Ethernet, camera, NPU, GPU
Runner: Serial-connected test script (see Section 6: Hardware Testing Protocol)
```

### Test Naming Convention

```
# Host tests
fn kernel_page_table_maps_4kb_page()
fn kernel_scheduler_preempts_on_timer()
fn kernel_ipc_send_blocks_until_received()
fn driver_uart_ring_buffer_wraps_correctly()
fn net_tcp_state_machine_handles_syn_ack()
fn fs_fat32_reads_cluster_chain()

# QEMU tests
fn qemu_kernel_boots_and_prints_banner()
fn qemu_mmu_identity_maps_kernel_region()
fn qemu_gic_timer_irq_fires()
fn qemu_two_processes_communicate_via_ipc()

# Hardware tests
fn hw_gpio96_toggles_at_1hz()
fn hw_uart5_echoes_bytes()
fn hw_spi12_loopback_matches()
fn hw_npu_mobilenet_infers_under_5ms()
```

---

## 4. QEMU Development Loop

The QEMU development loop is the primary iteration cycle for kernel development. The goal is to minimize the time between writing code and seeing it run on an aarch64 machine.

### Setup

```bash
# Required packages
sudo apt install qemu-system-aarch64 qemu-efi-aarch64

# OVMF firmware for EFI boot testing
sudo apt install ovmf
# or download: https://releases.linaro.org/components/kernel/uefi-linaro/
```

### Fast Loop (Bare ELF, 2-5 seconds)

```bash
# Compile → Boot → Verify — the core loop
fj build --target aarch64-none os/kernel.fj -o fajaros.elf && \
qemu-system-aarch64 \
  -M virt -cpu cortex-a76 -m 4G \
  -kernel fajaros.elf \
  -nographic -serial mon:stdio

# Expected: kernel serial output appears in terminal
# Press Ctrl+A then X to exit QEMU
```

### EFI Boot Loop (5-10 seconds)

```bash
# For testing UEFI boot path (Phase 1, Sprint 4+)
fj build --target aarch64-none --format efi os/kernel.fj -o BOOTAA64.EFI

# Create EFI System Partition image
dd if=/dev/zero of=esp.img bs=1M count=64
mkfs.fat -F 32 esp.img
mmd -i esp.img ::/EFI
mmd -i esp.img ::/EFI/BOOT
mcopy -i esp.img BOOTAA64.EFI ::/EFI/BOOT/

# Boot with OVMF
qemu-system-aarch64 \
  -M virt -cpu cortex-a76 -m 4G \
  -bios /usr/share/qemu-efi-aarch64/QEMU_EFI.fd \
  -drive file=esp.img,format=raw,if=virtio \
  -nographic -serial mon:stdio
```

### Full System Loop (10-30 seconds)

```bash
# For testing with rootfs, network, and block device (Phase 4+)
qemu-system-aarch64 \
  -M virt -cpu cortex-a76 -m 4G -smp 8 \
  -kernel fajaros.elf \
  -nographic -serial mon:stdio \
  -drive file=rootfs.img,format=raw,if=virtio \
  -netdev user,id=net0,hostfwd=tcp::2222-:22,hostfwd=tcp::8080-:80 \
  -device virtio-net,netdev=net0 \
  -d int 2>qemu_debug.log
```

### QEMU Debug Flags

| Flag | Purpose | When to Use |
|------|---------|-------------|
| `-d int` | Log interrupts | Debugging GIC, timer, IRQ routing |
| `-d cpu_reset` | Log CPU reset | Debugging boot failures |
| `-d mmu` | Log MMU faults | Debugging page table issues |
| `-d in_asm` | Log executed instructions | Debugging asm! correctness |
| `-d guest_errors` | Log guest errors | Debugging exception handlers |
| `-s -S` | GDB server, wait for attach | Step-through debugging |
| `-monitor telnet::4444,server,nowait` | QEMU monitor | Live hardware inspection |

### QEMU GDB Debugging

```bash
# Terminal 1: start QEMU paused
qemu-system-aarch64 \
  -M virt -cpu cortex-a76 -m 4G \
  -kernel fajaros.elf \
  -nographic -serial mon:stdio \
  -s -S

# Terminal 2: attach GDB
gdb-multiarch fajaros.elf \
  -ex "target remote :1234" \
  -ex "break kernel_main" \
  -ex "continue"
```

### Automated QEMU Test Script

```bash
#!/bin/bash
# scripts/qemu_test.sh — Run all QEMU tests
set -e

TIMEOUT=10
PASS=0
FAIL=0

run_qemu_test() {
    local name="$1"
    local expected="$2"
    local kernel="$3"

    echo -n "TEST: $name ... "

    OUTPUT=$(timeout $TIMEOUT qemu-system-aarch64 \
        -M virt -cpu cortex-a76 -m 4G \
        -kernel "$kernel" \
        -nographic -serial mon:stdio \
        2>/dev/null || true)

    if echo "$OUTPUT" | grep -q "$expected"; then
        echo "PASS"
        PASS=$((PASS + 1))
    else
        echo "FAIL"
        echo "  Expected: $expected"
        echo "  Got: $(echo "$OUTPUT" | head -5)"
        FAIL=$((FAIL + 1))
    fi
}

# Compile
fj build --target aarch64-none os/kernel.fj -o fajaros.elf

# Run tests
run_qemu_test "boot_banner" "FajarOS v3.0 Surya" "fajaros.elf"
run_qemu_test "cpu_detect" "Kryo 670" "fajaros.elf"
run_qemu_test "mmu_enabled" "MMU: enabled" "fajaros.elf"
run_qemu_test "gic_init" "GIC: initialized" "fajaros.elf"
run_qemu_test "scheduler" "Process 1 running" "fajaros.elf"

echo ""
echo "Results: $PASS passed, $FAIL failed"
[ $FAIL -eq 0 ] || exit 1
```

---

## 5. Hardware Testing Protocol

### 5.1 When to Test on Hardware

| Phase | Hardware Testing | Reason |
|-------|-----------------|--------|
| Phase 1 (Compiler) | NO | Pure compiler work; QEMU sufficient |
| Phase 2 (Microkernel) | NO | QEMU aarch64 virt is accurate enough |
| Phase 3 (HAL Drivers) | **YES** | GPIO, UART, SPI, I2C need real pins |
| Phase 4 (Storage) | **YES** | NVMe, eMMC, SD need real devices |
| Phase 5 (Network) | **YES** | Ethernet PHY, DHCP need real network |
| Phase 6 (Display) | **YES** | HDMI, USB keyboard need real peripherals |
| Phase 7 (AI) | **YES** | NPU (Hexagon 770), GPU (Adreno 643) are hardware-only |
| Phase 8 (Services) | PARTIAL | Some on QEMU, process isolation on hardware |
| Phase 9 (Shell/Apps) | **YES** | Full system integration |
| Phase 10 (Production) | **YES** | Stress tests, security, release validation |

### 5.2 Hardware Test Setup

```
Host PC (x86_64 Linux)
  │
  ├── USB Serial Adapter ───→ Dragon Q6A UART5 (pin 8 TX / pin 10 RX)
  │   └── /dev/ttyUSB0 @ 115200 baud
  │
  ├── Ethernet Cable ───────→ Dragon Q6A GbE RJ45
  │   └── 192.168.1.0/24 subnet
  │
  ├── HDMI Capture Card ────→ Dragon Q6A HDMI port
  │   └── /dev/video0 (for automated display verification)
  │
  └── USB-C (EDL mode) ────→ Dragon Q6A USB-C (for flashing)
      └── qdl / edl tool

Dragon Q6A
  ├── NVMe SSD: FajarOS root filesystem
  ├── MicroSD: test data / recovery image
  ├── GPIO test circuit:
  │   ├── GPIO96 (pin 7) → LED (with 330R resistor)
  │   ├── GPIO0 (pin 13) → button (pull-up)
  │   └── GPIO97 (pin 11) → logic analyzer
  ├── I2C6 (pin 3 SDA / pin 5 SCL) → BME280 sensor breakout
  ├── SPI12 (pin 19 MOSI / pin 21 MISO) → loopback wire
  ├── Camera: Radxa Camera 12M 577 (MIPI CSI)
  └── HDMI: test monitor (1920x1080)
```

### 5.3 Flashing FajarOS to Dragon Q6A

```bash
# Option A: SD card boot (development — fastest iteration)
fj build --target aarch64-none --format efi os/kernel.fj -o BOOTAA64.EFI
# Mount SD card EFI partition, copy BOOTAA64.EFI

# Option B: NVMe install (production)
fj build --target aarch64-none --format image os/ -o fajaros.img
# Flash via USB: boot Q6A into EDL mode (hold VOL- during power on)
qdl prog_firehose.elf rawprogram.xml patch.xml

# Option C: Network boot (CI — fastest for automated testing)
# Configure Q6A UEFI to PXE boot
# Host PC runs TFTP server serving fajaros.efi
```

### 5.4 Hardware Test Execution

```bash
#!/bin/bash
# scripts/hw_test.sh — Run hardware-in-loop tests on Dragon Q6A
set -e

SERIAL=/dev/ttyUSB0
BAUD=115200

# Open serial console
exec 3<>/dev/ttyUSB0
stty -F $SERIAL $BAUD cs8 -cstopb -parenb raw -echo

# Reset board (GPIO-controlled reset line, or manual power cycle)
echo "Power-cycling Dragon Q6A..."
# gpio_reset.sh or manual prompt

# Wait for boot
echo "Waiting for boot (30 seconds)..."
BOOT_LOG=""
for i in $(seq 1 30); do
    read -t 1 -r line <&3 2>/dev/null && BOOT_LOG+="$line\n" || true
done

# Verify boot
if echo -e "$BOOT_LOG" | grep -q "FajarOS v3.0 Surya"; then
    echo "PASS: FajarOS booted on Dragon Q6A"
else
    echo "FAIL: boot banner not found"
    echo -e "$BOOT_LOG"
    exit 1
fi

# Run test commands via serial shell
send_cmd() {
    echo "$1" >&3
    sleep 1
    read -t 2 -r response <&3 2>/dev/null
    echo "$response"
}

# GPIO test
send_cmd "gpio write 96 1"
send_cmd "gpio read 96"
# Verify with multimeter or logic analyzer

# UART echo test
send_cmd "uart echo on 5"
# Send test bytes, verify they come back

# I2C sensor test
SENSOR=$(send_cmd "i2c read 6 0x76 0xD0 1")
if echo "$SENSOR" | grep -q "0x60"; then
    echo "PASS: BME280 sensor detected on I2C6"
else
    echo "FAIL: BME280 not detected"
fi

exec 3>&-
```

### 5.5 Hardware Test Checklist (Per Phase Gate)

```
Phase 3 (HAL Drivers):
  [ ] GPIO96 toggles — LED blinks at 1Hz (verified by eye)
  [ ] GPIO0 reads button press (verified by logic analyzer)
  [ ] UART5 echo works at 115200 baud (verified by USB serial terminal)
  [ ] SPI12 loopback: TX bytes match RX bytes
  [ ] I2C6 reads BME280 chip ID (0x60) and temperature
  [ ] Timer delay of 1 second is accurate to ±10ms
  [ ] DMA copies 1MB without corruption

Phase 4 (Storage):
  [ ] NVMe SSD detected via PCIe enumeration
  [ ] NVMe read/write 512 bytes matches
  [ ] SD card partition table reads correctly
  [ ] FAT32 file read from SD card succeeds
  [ ] ext4 file read from NVMe partition succeeds

Phase 5 (Network):
  [ ] Ethernet link up at 1 Gbps
  [ ] DHCP obtains IP address from router
  [ ] Ping to gateway succeeds
  [ ] HTTP server serves page (curl from host PC)
  [ ] SSH login from host PC works

Phase 6 (Display):
  [ ] HDMI output shows boot console at 1920x1080
  [ ] USB keyboard types characters
  [ ] Shell window displays in compositor
  [ ] MIPI DSI display works (if Radxa display attached)

Phase 7 (AI):
  [ ] NPU detects Hexagon 770 CDSP
  [ ] MobileNetV2 inference < 5ms on NPU
  [ ] GPU matmul 512x512 produces correct result
  [ ] Camera captures 1080p frame
  [ ] Camera → NPU → Display pipeline at >= 30 FPS
```

---

## 6. Quality Gates

### 6.1 Per-Task Gate

Before marking any task `[x]` in V30_PLAN.md:

```
[ ] Host tests pass:         cargo test
[ ] Native codegen tests:    cargo test --features native
[ ] Clippy clean:            cargo clippy -- -D warnings
[ ] Formatted:               cargo fmt -- --check
[ ] No .unwrap() in src/:    Only allowed in tests
[ ] Documented:              All new pub items have /// doc comments
[ ] Context annotations:     Every kernel fn is @kernel, every AI fn is @device
[ ] asm!() documented:       Every asm!() block has // SAFETY: comment
[ ] Feature gated:           New deps behind --features fajaros flag
[ ] QEMU boots:              If kernel code changed, ELF boots in QEMU
```

### 6.2 Per-Sprint Gate

```
[ ] All per-task gates pass
[ ] Zero regressions (3,392 existing tests still pass)
[ ] New tests added (minimum 5 per sprint)
[ ] V30_PLAN.md updated with [x] marks
[ ] QEMU test suite passes (all kernel tests)
[ ] No feature gate leaks (code compiles without --features fajaros)
[ ] Kernel binary size reasonable (< 2MB for microkernel, < 16MB total)
```

### 6.3 Per-Phase Gate

```
Phase 1 — Compiler Bare-Metal Support:
  [ ] `fj build --target aarch64-none kernel.fj` produces valid aarch64 ELF
  [ ] ELF boots in QEMU, prints to serial console
  [ ] EFI binary boots in QEMU with OVMF firmware
  [ ] volatile_read/write and asm!() with constraints compile correctly
  [ ] All 40 tasks pass, 0 regressions

Phase 2 — Microkernel:
  [ ] FajarOS boots in QEMU with MMU enabled (4-level page table)
  [ ] Exception handling works (syscall via SVC, page fault, IRQ)
  [ ] GICv3 interrupts fire and are handled (timer IRQ verified)
  [ ] 3+ processes run concurrently with preemptive scheduling
  [ ] IPC message passing works (send → receive → reply)
  [ ] All 60 tasks pass, kernel serial output verified

Phase 3 — HAL Drivers:
  [ ] GPIO blink works on real Dragon Q6A hardware
  [ ] UART echo works with serial terminal at 115200 baud
  [ ] SPI/I2C communicate with external devices
  [ ] Timer provides accurate delays (1s ±10ms)
  [ ] DMA transfers data without CPU involvement
  [ ] All 50 tasks pass
  [ ] All drivers are @kernel context, compiler-verified

Phase 4 — Storage & Filesystem:
  [ ] NVMe SSD block read/write works on Q6A
  [ ] SD card block read/write works
  [ ] VFS layer mounts filesystems, POSIX-like file API
  [ ] FAT32 read/write works
  [ ] ext4 read works
  [ ] All 40 tasks pass

Phase 5 — Network Stack:
  [ ] Ethernet link up on Dragon Q6A GbE port
  [ ] DHCP obtains IP address
  [ ] Ping works (both send and respond)
  [ ] TCP connections work (HTTP server + client)
  [ ] SSH server allows remote shell access
  [ ] All 40 tasks pass

Phase 6 — Display & Input:
  [ ] HDMI displays text and graphics on real monitor
  [ ] USB keyboard input works
  [ ] Terminal shell runs in windowed compositor
  [ ] Status bar shows real CPU/memory info
  [ ] All 30 tasks pass

Phase 7 — AI Subsystem:
  [ ] NPU inference works on real Hexagon 770 (MobileNetV2 < 5ms)
  [ ] GPU compute works on real Adreno 643 (matmul >= 100 GFLOPS)
  [ ] Camera captures live video at 30 FPS
  [ ] Camera → NPU → Display pipeline runs at >= 30 FPS
  [ ] On-device training works (simple CNN)
  [ ] All 50 tasks pass
  [ ] @device context enforced — no raw pointers in AI code

Phase 8 — OS Services:
  [ ] Init system starts services in dependency order
  [ ] Process isolation prevents cross-process memory access
  [ ] User system with login and permissions works
  [ ] Device manager enumerates hardware at boot
  [ ] Power management controls CPU/GPU frequency
  [ ] All 40 tasks pass

Phase 9 — Shell & Applications:
  [ ] Shell provides full system control (ls, cat, ps, gpio, npu-info)
  [ ] Fajar Lang programs compile and run on-device
  [ ] REPL can control hardware in real-time
  [ ] 9 demo applications work on real hardware
  [ ] All 30 tasks pass

Phase 10 — Production:
  [ ] 72-hour stress test passes with 0 failures
  [ ] Security audit complete, no critical vulnerabilities
  [ ] Complete documentation (user, developer, kernel, AI, hardware)
  [ ] Flashable image builds and installs successfully
  [ ] OTA update works
  [ ] All 40 tasks pass
  [ ] FajarOS 3.0 "Surya" RELEASED
```

### 6.4 Release Gate (FajarOS 3.0.0)

```
[ ] All 420 tasks complete (10 phases, 42 sprints)
[ ] All host tests pass (3,392 baseline + ~850 new OS tests)
[ ] All QEMU tests pass
[ ] All hardware tests pass on Dragon Q6A
[ ] 72-hour stress test: 0 kernel panics, 0 memory leaks
[ ] Security: W^X, ASLR, stack canaries, capability enforcement
[ ] Documentation: user guide, developer guide, kernel reference, AI guide, hardware guide
[ ] 9 demo applications run correctly
[ ] Flashable image installs and boots from NVMe
[ ] OTA update mechanism works
[ ] CHANGELOG.md updated
[ ] CLAUDE.md updated
[ ] README.md updated
[ ] GitHub release with pre-built images
```

---

## 7. CI Pipeline

### 7.1 Build Matrix

```yaml
# .github/workflows/fajaros.yml
name: FajarOS CI

on: [push, pull_request]

jobs:
  # Tier 1: Host tests (every push)
  host-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: cargo build
      - name: Test (default)
        run: cargo test
      - name: Test (native codegen)
        run: cargo test --features native
      - name: Test (FajarOS)
        run: cargo test --features fajaros
      - name: Clippy
        run: cargo clippy -- -D warnings
      - name: Format
        run: cargo fmt -- --check

  # Tier 2: QEMU tests (every push that changes os/ or kernel code)
  qemu-tests:
    runs-on: ubuntu-latest
    needs: host-tests
    steps:
      - uses: actions/checkout@v4
      - name: Install QEMU
        run: sudo apt-get install -y qemu-system-aarch64 qemu-efi-aarch64
      - name: Build kernel
        run: fj build --target aarch64-none os/kernel.fj -o fajaros.elf
      - name: QEMU boot test
        run: scripts/qemu_test.sh
      - name: QEMU integration tests
        run: scripts/qemu_integration.sh

  # Tier 3: Hardware tests (manual trigger, or nightly)
  hardware-tests:
    runs-on: self-hosted  # Dragon Q6A runner
    if: github.event_name == 'workflow_dispatch' || github.event.schedule
    needs: qemu-tests
    steps:
      - uses: actions/checkout@v4
      - name: Build kernel for Q6A
        run: fj build --target aarch64-none --board dragon-q6a os/ -o fajaros.img
      - name: Flash to Q6A
        run: scripts/flash_q6a.sh fajaros.img
      - name: Wait for boot
        run: scripts/wait_boot.sh /dev/ttyUSB0 30
      - name: Run hardware tests
        run: scripts/hw_test.sh
      - name: Collect serial log
        if: always()
        run: scripts/collect_serial_log.sh
```

### 7.2 CI Stages

```
Stage 1: Host Build + Test          ~3 minutes
  ├── cargo build
  ├── cargo test
  ├── cargo test --features native
  ├── cargo test --features fajaros
  ├── cargo clippy -- -D warnings
  └── cargo fmt -- --check

Stage 2: QEMU Integration           ~5 minutes
  ├── fj build --target aarch64-none
  ├── QEMU boot test (banner check)
  ├── QEMU MMU test
  ├── QEMU interrupt test
  ├── QEMU scheduler test
  └── QEMU IPC test

Stage 3: Hardware-in-Loop            ~10 minutes (nightly / manual)
  ├── Flash kernel to Q6A
  ├── Verify boot via serial
  ├── GPIO test
  ├── UART test
  ├── SPI/I2C test
  ├── Network test
  ├── NPU inference test
  └── Collect logs + artifacts

Total CI time (Stage 1+2): < 10 minutes
Total CI time (all stages): < 20 minutes
```

### 7.3 CI Failure Policy

```
Stage 1 failure:  BLOCKS merge. Fix immediately.
Stage 2 failure:  BLOCKS merge if kernel code changed. Investigate QEMU log.
Stage 3 failure:  Does NOT block merge (hardware may be offline).
                  Creates GitHub issue tagged [hardware-regression].
                  Must be resolved before next phase gate.
```

---

## 8. Debug Workflow

### 8.1 Kernel Panic Debugging

When the kernel panics (exception, assertion failure, or explicit `panic!`), the panic handler prints a context dump to serial:

```
!!! KERNEL PANIC !!!
Message: page fault at 0xDEAD_BEEF
ESR_EL1: 0x96000007 (Data Abort, level 3)
ELR_EL1: 0x4000_1A34
FAR_EL1: 0xDEAD_BEEF
SPSR_EL1: 0x6000_03C5

Registers:
  x0=0x0000000000000001  x1=0x000000004000A000
  x2=0x00000000DEADBEEF  x3=0x0000000000000000
  ...
  SP=0x00000000470FFFF0  LR=0x000000004000192C

Stack trace:
  0x4000_1A34  page_map+0x84
  0x4000_192C  kernel_init+0x1C
  0x4000_0100  _start+0x40
```

**Debugging steps:**

1. **Read the panic message** — identifies the immediate cause
2. **Decode ESR_EL1** — exception class (bits [31:26]) tells you the exception type
3. **Check ELR_EL1** — this is the instruction that faulted; use `objdump` to find it:
   ```bash
   aarch64-linux-gnu-objdump -d fajaros.elf | grep -A 5 "4000_1A34"
   ```
4. **Check FAR_EL1** — the faulting address (for data/instruction aborts)
5. **Walk the stack trace** — identify the call chain leading to the fault
6. **Reproduce in QEMU with `-d` flags** — get detailed execution trace

### 8.2 Common Exception Classes (ESR_EL1.EC)

| EC (hex) | Exception | Common Cause |
|----------|-----------|--------------|
| 0x00 | Unknown | Invalid instruction encoding |
| 0x15 | SVC (syscall) | Expected — dispatched to syscall handler |
| 0x20 | Instruction abort (lower EL) | Process jumped to unmapped address |
| 0x21 | Instruction abort (same EL) | Kernel executed unmapped code |
| 0x24 | Data abort (lower EL) | Process read/wrote unmapped memory |
| 0x25 | Data abort (same EL) | Kernel read/wrote unmapped memory |
| 0x2C | Floating-point | FP operation without CPACR_EL1 enabled |

### 8.3 Driver Failure Debugging

When a hardware driver fails (no response, wrong data, timeout):

```
1. VERIFY WIRING
   → Check physical connections (GPIO pin, I2C address, SPI CS)
   → Check voltage levels (3.3V vs 1.8V — Q6A GPIO is 1.8V!)

2. VERIFY REGISTER ADDRESSES
   → Cross-reference with QCS6490 TRM (or Linux kernel dts)
   → Verify TLMM mux settings match required function
   → Check that MMIO region is mapped in page table

3. CHECK CLOCK AND POWER
   → Many QCS6490 peripherals require clock enable before use
   → Verify QUP clock is enabled for UART/SPI/I2C
   → Check GDSC (power domain) is ON for the peripheral

4. USE LOGIC ANALYZER
   → Connect Saleae or similar to the bus
   → Verify signal timing, clock frequency, data integrity
   → Compare against expected protocol waveform

5. COMPARE WITH LINUX
   → Boot Linux on Q6A
   → Use devmem2 to read the same registers
   → Compare values with FajarOS driver reads
```

### 8.4 Timing Issue Debugging

Timing issues (race conditions, interrupt latency, scheduler jitter):

```
1. ADD TIMESTAMPS
   → Read CNTVCT_EL0 (virtual timer counter) at key points
   → Print delta timestamps to serial
   → timer_get_ticks() before and after critical sections

2. DISABLE INTERRUPTS FOR MEASUREMENT
   → DAIFSET to mask IRQs during timing-critical measurement
   → Re-enable immediately after measurement
   → Report interrupt-off duration (must be < 100us)

3. USE QEMU TRACE EVENTS
   → qemu-system-aarch64 -trace "gicv3*" ...
   → qemu-system-aarch64 -trace "timer*" ...
   → Compare IRQ delivery timing with expected schedule

4. CHECK SCHEDULER QUANTUM
   → Default quantum: 10ms (configurable)
   → Timer IRQ must fire reliably at quantum interval
   → Verify with: serial print on every timer IRQ for 100 ticks

5. CHECK MEMORY BARRIERS
   → Missing DMB/DSB/ISB after register writes
   → Especially critical after: page table update, GIC config, cache ops
   → Pattern: volatile_write() + dsb() + isb()
```

### 8.5 Memory Debugging

Memory corruption, leaks, or invalid access:

```
1. KERNEL HEAP TRACKING
   → kernel_alloc() tracks total allocated bytes
   → Print heap usage periodically: "Heap: 1.2MB / 64MB used"
   → Detect leaks: usage should stabilize after boot

2. PAGE TABLE DUMPS
   → Implement page_table_dump(table) → serial output
   → Print mapped regions: VA range, PA range, flags
   → Verify no overlap, no missing mappings

3. GUARD PAGES
   → Map guard pages (unmapped) around kernel stacks
   → Stack overflow → immediate data abort → clear error message
   → Guard page at 0x0000_0000 catches null pointer deref

4. QEMU MEMORY WATCH
   → Use QEMU monitor: `info mtree` shows memory map
   → Use QEMU monitor: `xp /16gx 0x42000000` reads kernel heap
   → Set watchpoint: `watch *0xDEADBEEF` breaks on write
```

---

## 9. Code Review Checklist

### 9.1 General (All Code)

```
[ ] No .unwrap() in src/ — only in tests
[ ] No unsafe without // SAFETY: comment
[ ] All pub items have /// doc comments
[ ] cargo test passes
[ ] cargo clippy -- -D warnings passes
[ ] cargo fmt -- --check passes
[ ] New functions have at least 1 test
```

### 9.2 Kernel Code (@kernel context)

```
[ ] Every function is annotated @kernel
[ ] No heap allocation (no String, no Vec, no Box)
    → Use fixed-size arrays, static buffers, kernel_alloc()
[ ] No tensor operations (no zeros(), no matmul(), no relu())
[ ] No network I/O
[ ] No filesystem operations
[ ] Every asm!() block has // SAFETY: comment explaining:
    → What registers are used and why
    → What memory is accessed and why it is safe
    → What side effects occur (interrupt state, MMU state)
[ ] Every volatile_read/volatile_write has correct address and size
[ ] Memory barriers (dmb/dsb/isb) placed after register writes that
    require ordering (page table updates, GIC config, cache ops)
[ ] Interrupts disabled during critical sections (DAIF manipulation)
    → Re-enabled as soon as possible (< 100us max)
[ ] No unbounded loops — every loop has a timeout or bound
[ ] Stack usage per function estimated (no > 4KB per frame)
[ ] DMA buffers are physically contiguous and cache-coherent
```

### 9.3 Device Code (@device context)

```
[ ] Every function is annotated @device
[ ] No raw pointer access
[ ] No IRQ manipulation
[ ] No syscall invocation
[ ] No volatile I/O
[ ] No asm!() blocks
[ ] Tensor operations use proper shape checking
[ ] GPU/NPU resources are released (model unload, buffer free)
[ ] Zero-copy buffer sharing uses proper synchronization
[ ] NPU model loading handles errors gracefully (model not found, format wrong)
```

### 9.4 Safe Code (@safe context)

```
[ ] No raw pointers
[ ] No IRQ, no volatile I/O, no asm!()
[ ] All hardware access goes through syscalls
[ ] Error handling: every syscall result checked
[ ] Resources cleaned up (file descriptors closed, memory freed)
[ ] No blocking operations without timeout
```

### 9.5 Inline Assembly Review

Every `asm!()` block requires careful review:

```
[ ] Correct register constraints (in/out/inout/lateout)
[ ] Correct clobbers listed (memory, cc, specific registers)
[ ] Correct options (nomem, nostack, preserves_flags — only if truly applicable)
[ ] AT&T vs Intel syntax consistent (Fajar Lang uses GNU AS syntax)
[ ] aarch64-specific: correct system register names (MSR/MRS)
[ ] aarch64-specific: correct condition codes
[ ] No undefined behavior: no writing to SP without restoring it
[ ] No security holes: no disabling MMU from user context
```

---

## 10. Release Process

### 10.1 Version Numbering

```
FajarOS 3.0.0
│       │ │ │
│       │ │ └── Patch: bug fixes only (no new features, no API changes)
│       │ └──── Minor: new features, backward-compatible
│       └────── Major: breaking changes (new syscall ABI, etc.)
└────────────── V3.0 plan identifier

Pre-release tags:
  3.0.0-alpha.1  — Phases 1-2 complete (boots in QEMU)
  3.0.0-alpha.2  — Phases 1-4 complete (boots with filesystem)
  3.0.0-beta.1   — Phases 1-7 complete (full hardware support)
  3.0.0-beta.2   — Phases 1-9 complete (usable system)
  3.0.0-rc.1     — Phase 10 stress testing
  3.0.0           — Production release
```

### 10.2 Building Flashable Images

```bash
# Step 1: Build kernel ELF
fj build --target aarch64-none --board dragon-q6a \
  os/kernel.fj -o build/fajaros-kernel.elf

# Step 2: Build all services and applications
for app in os/services/*.fj os/apps/*.fj; do
    fj build --target aarch64-none "$app" -o "build/$(basename $app .fj)"
done

# Step 3: Create root filesystem image
# Layout:
#   /boot/fajaros-kernel.elf
#   /bin/ (shell, utilities)
#   /sbin/ (services, drivers)
#   /etc/fajaros/ (config files)
#   /lib/ (shared libraries, NPU models)
#   /home/ (user data)
scripts/mkrootfs.sh build/ rootfs.img

# Step 4: Create EFI System Partition
scripts/mkesp.sh build/fajaros-kernel.elf esp.img

# Step 5: Create combined disk image
# Partition table:
#   1: EFI System Partition (64MB, FAT32) — kernel
#   2: Root filesystem (remaining, ext4) — everything else
scripts/mkdisk.sh esp.img rootfs.img fajaros-3.0.0.img

# Step 6: Create SD card installer
scripts/mkinstaller.sh fajaros-3.0.0.img fajaros-installer.img
```

### 10.3 Release Checklist

```
Pre-release:
  [ ] All 420 tasks marked [x] in V30_PLAN.md
  [ ] All host tests pass (cargo test, cargo test --features native)
  [ ] All QEMU tests pass (scripts/qemu_test.sh)
  [ ] All hardware tests pass on Dragon Q6A (scripts/hw_test.sh)
  [ ] 72-hour stress test completed with 0 failures
  [ ] Security audit completed (Sprint 40)
  [ ] All documentation written (Sprint 41)

Image build:
  [ ] Kernel ELF built for aarch64-none
  [ ] All services and apps built
  [ ] Root filesystem image created
  [ ] Disk image created
  [ ] SD card installer created
  [ ] Image checksums (SHA256) computed

Validation:
  [ ] Fresh install from SD card → boots successfully
  [ ] All demo apps run on fresh install
  [ ] OTA update from previous version works
  [ ] Recovery mode boots from SD card
  [ ] HDMI display works
  [ ] Network (Ethernet) works
  [ ] Camera + NPU pipeline works

Release:
  [ ] Version bumped: FajarOS 3.0.0 in kernel banner
  [ ] CHANGELOG.md updated with all changes
  [ ] CLAUDE.md status updated
  [ ] README.md updated with FajarOS section
  [ ] Git tag: v3.0.0
  [ ] GitHub release created with:
      ├── fajaros-3.0.0.img (disk image)
      ├── fajaros-installer.img (SD card installer)
      ├── fajaros-kernel.elf (standalone kernel)
      ├── SHA256SUMS
      └── Release notes
  [ ] Blog post / announcement published
```

### 10.4 OTA Update Process

```
1. Build new image: fj build ... → fajaros-3.0.1.img
2. Generate delta: scripts/mkdelta.sh 3.0.0.img 3.0.1.img → update.delta
3. Sign delta: scripts/sign.sh update.delta → update.delta.sig
4. Upload: scp update.delta update.delta.sig update-server:/updates/
5. On device: fajaros-update check → "Update 3.0.1 available (12MB)"
6. On device: fajaros-update apply → download, verify sig, apply to B partition
7. On device: reboot → boots from B partition
8. If B partition fails: watchdog timeout → automatic rollback to A partition
```

---

## 11. Feature Gating

### 11.1 Required Feature Flags

| Feature Flag | What It Gates | Dependencies |
|-------------|---------------|--------------|
| `fajaros` | All FajarOS kernel/driver code | (bare-metal runtime) |
| `fajaros-net` | Network stack (TCP/IP, Ethernet) | (implies `fajaros`) |
| `fajaros-fs` | Filesystem (VFS, FAT32, ext4) | (implies `fajaros`) |
| `fajaros-display` | Display subsystem (HDMI, compositor) | (implies `fajaros`) |
| `fajaros-gpu` | Adreno 643 GPU compute | (implies `fajaros`) |
| `fajaros-npu` | Hexagon 770 NPU inference | (implies `fajaros`) |
| `fajaros-camera` | MIPI CSI camera driver | (implies `fajaros`) |
| `fajaros-usb` | USB host controller (keyboard) | (implies `fajaros`) |
| `fajaros-full` | All of the above | (implies all) |

### 11.2 Feature Gate Rules

```
1. ALL FajarOS code MUST be behind #[cfg(feature = "fajaros")] or sub-features.
   The main Fajar Lang compiler MUST build and pass ALL existing tests
   without any fajaros feature enabled.

2. Sub-features imply the base feature:
   fajaros-net → fajaros
   fajaros-gpu → fajaros
   This ensures kernel primitives are always available when drivers are enabled.

3. CI must test:
   • cargo test                              (no OS code, baseline passes)
   • cargo test --features native            (Cranelift codegen)
   • cargo test --features fajaros           (kernel + drivers, host tests)
   • cargo test --features fajaros-full      (everything)

4. Feature-gated modules use:
   #[cfg(feature = "fajaros")]
   pub mod os_kernel;

5. Feature-gated tests use:
   #[cfg(feature = "fajaros")]
   #[test]
   fn kernel_page_table_maps_4kb_page() { ... }

6. NEVER put feature gates on existing non-OS code.
   FajarOS is additive — it extends the compiler, not replaces it.
```

### 11.3 Conditional Compilation Pattern

```rust
// In Cargo.toml
[features]
fajaros = []
fajaros-net = ["fajaros"]
fajaros-fs = ["fajaros"]
fajaros-display = ["fajaros"]
fajaros-gpu = ["fajaros"]
fajaros-npu = ["fajaros"]
fajaros-camera = ["fajaros"]
fajaros-usb = ["fajaros"]
fajaros-full = ["fajaros-net", "fajaros-fs", "fajaros-display",
                "fajaros-gpu", "fajaros-npu", "fajaros-camera", "fajaros-usb"]
```

```rust
// In src/lib.rs
#[cfg(feature = "fajaros")]
pub mod fajaros;

// In src/fajaros/mod.rs
pub mod kernel;     // microkernel (always with fajaros)
pub mod hal;        // HAL drivers (always with fajaros)

#[cfg(feature = "fajaros-net")]
pub mod net;

#[cfg(feature = "fajaros-fs")]
pub mod fs;

#[cfg(feature = "fajaros-display")]
pub mod display;

#[cfg(feature = "fajaros-gpu")]
pub mod gpu;

#[cfg(feature = "fajaros-npu")]
pub mod npu;

#[cfg(feature = "fajaros-camera")]
pub mod camera;

#[cfg(feature = "fajaros-usb")]
pub mod usb;
```

### 11.4 Build Profiles

```bash
# Development: fast compile, debug symbols, QEMU target
fj build --target aarch64-none --profile dev os/kernel.fj

# Release: optimized, stripped, Dragon Q6A target
fj build --target aarch64-none --profile release --board dragon-q6a os/kernel.fj

# Minimal: microkernel only, no drivers, no services (for boot testing)
fj build --target aarch64-none --features fajaros os/kernel_minimal.fj

# Full: everything enabled (for production image)
fj build --target aarch64-none --features fajaros-full --board dragon-q6a os/
```

---

## 12. Session Protocol (Claude Code)

Every Claude Code session for FajarOS work MUST follow this order:

```
1. READ  → CLAUDE.md (auto-loaded)
2. READ  → docs/V30_PLAN.md (find current sprint / user request)
3. READ  → docs/V30_SKILLS.md (if implementing hardware-touching feature)
4. READ  → docs/RADXA_Q6A_HARDWARE.md (if touching Q6A-specific registers)
5. READ  → docs/Q6A_LOW_LEVEL_DEV.md (if touching boot chain or SPI firmware)
6. ORIENT → "What does the user want?"
7. ACT   → Execute per TDD workflow:
            a. Write host unit test (RED)
            b. Implement minimally (GREEN)
            c. Write QEMU integration test
            d. Verify: cargo test + clippy + fmt
            e. If kernel code: verify QEMU boots
8. UPDATE → Mark task [x] in V30_PLAN.md
```

### OS-Specific Session Rules

```
1. NEVER write kernel code without @kernel annotation.
2. NEVER use heap allocation (String, Vec, Box) in @kernel context.
3. NEVER write asm!() without a // SAFETY: comment.
4. ALWAYS wrap asm!() in a safe @kernel fn (never expose raw asm to callers).
5. ALWAYS test in QEMU before claiming kernel code works.
6. ALWAYS check register addresses against QCS6490 documentation.
7. ALWAYS use volatile_read/volatile_write for MMIO — never raw pointer deref.
8. ALWAYS place memory barriers (dsb/isb) after page table or GIC updates.
```

---

## 13. Directory Structure (FajarOS)

```
fajar-lang/
├── os/                                ← FajarOS source (100% Fajar Lang)
│   ├── kernel/
│   │   ├── boot.fj                    ← _start, UEFI handoff, early init
│   │   ├── mmu.fj                     ← 4-level page table, TLB
│   │   ├── exception.fj              ← Vector table, handlers
│   │   ├── gic.fj                     ← GICv3 interrupt controller
│   │   ├── scheduler.fj              ← Process management, context switch
│   │   ├── ipc.fj                     ← Message passing, shared memory
│   │   ├── memory.fj                  ← Kernel allocator (bump + freelist)
│   │   ├── syscall.fj                 ← Syscall dispatch table
│   │   └── panic.fj                   ← Panic handler, context dump
│   ├── hal/
│   │   ├── gpio.fj                    ← TLMM GPIO driver
│   │   ├── uart.fj                    ← QUP UART driver
│   │   ├── spi.fj                     ← QUP SPI driver
│   │   ├── i2c.fj                     ← QUP I2C driver
│   │   ├── timer.fj                   ← Architected timer
│   │   ├── dma.fj                     ← DMA engine
│   │   ├── pcie.fj                    ← PCIe controller + NVMe
│   │   ├── sdhci.fj                   ← SD/eMMC controller
│   │   ├── ethernet.fj               ← RGMII Ethernet + PHY
│   │   ├── usb.fj                     ← DWC3 USB host
│   │   ├── hdmi.fj                    ← MDP + HDMI output
│   │   ├── camera.fj                  ← MIPI CSI receiver
│   │   ├── gpu.fj                     ← Adreno 643 compute
│   │   └── npu.fj                     ← Hexagon 770 FastRPC
│   ├── services/
│   │   ├── init.fj                    ← Init system (PID 1)
│   │   ├── vfs.fj                     ← Virtual filesystem
│   │   ├── fat32.fj                   ← FAT32 driver
│   │   ├── ext4.fj                    ← ext4 driver
│   │   ├── tcp.fj                     ← TCP/IP stack
│   │   ├── dhcp.fj                    ← DHCP client
│   │   ├── http.fj                    ← HTTP server/client
│   │   ├── ssh.fj                     ← SSH server
│   │   ├── compositor.fj             ← Display compositor
│   │   ├── devmgr.fj                  ← Device manager
│   │   ├── users.fj                   ← User/permission system
│   │   └── power.fj                   ← Power management
│   ├── apps/
│   │   ├── fjsh.fj                    ← FajarOS shell
│   │   ├── blinky.fj                  ← GPIO LED blink demo
│   │   ├── sensor_logger.fj          ← I2C sensor logging
│   │   ├── camera_viewer.fj          ← Live camera display
│   │   ├── object_detector.fj        ← NPU object detection
│   │   ├── ai_trainer.fj             ← On-device training
│   │   ├── web_server.fj             ← HTTP sensor server
│   │   ├── mqtt_client.fj            ← MQTT sensor publisher
│   │   ├── benchmark.fj              ← CPU/GPU/NPU benchmark
│   │   └── system_monitor.fj         ← System dashboard
│   └── linker/
│       ├── kernel.ld                  ← Kernel linker script
│       └── user.ld                    ← Userspace linker script
├── scripts/
│   ├── qemu_test.sh                   ← QEMU boot tests
│   ├── qemu_integration.sh           ← QEMU integration tests
│   ├── hw_test.sh                     ← Hardware-in-loop tests
│   ├── flash_q6a.sh                   ← Flash image to Q6A
│   ├── wait_boot.sh                   ← Wait for serial boot banner
│   ├── collect_serial_log.sh         ← Collect serial output
│   ├── mkrootfs.sh                    ← Create root filesystem image
│   ├── mkesp.sh                       ← Create EFI partition image
│   ├── mkdisk.sh                      ← Create combined disk image
│   └── mkinstaller.sh                ← Create SD card installer
└── docs/
    ├── V30_PLAN.md                    ← Master plan (420 tasks)
    ├── V30_WORKFLOW.md                ← THIS FILE
    ├── V30_RULES.md                   ← OS-specific coding rules
    ├── V30_SKILLS.md                  ← Implementation patterns
    ├── RADXA_Q6A_HARDWARE.md          ← Hardware specification
    ├── Q6A_APP_DEV.md                 ← Application development
    ├── Q6A_LOW_LEVEL_DEV.md           ← Boot chain, EDL, firmware
    ├── Q6A_HARDWARE_USE.md            ← Power, storage, pinout
    └── Q6A_ACCESSORIES.md             ← Camera, display, storage
```

---

## 14. Parallel Development Tracks

FajarOS phases have dependencies. The following tracks can be parallelized:

```
Track A (Critical Path): Phase 1 → Phase 2 → Phase 3 → Phase 4 → Phase 5
  Compiler bare-metal → Microkernel → HAL drivers → Storage → Network
  (Each phase depends on the previous one)

Track B (Display):  Phase 6 (can start after Phase 3 — needs HAL)
Track C (AI):       Phase 7 (can start after Phase 3 — needs HAL + DMA)
Track D (Services): Phase 8 (can start after Phase 4 — needs filesystem)
Track E (Apps):     Phase 9 (depends on all above)
Track F (Release):  Phase 10 (depends on all above)
```

### Recommended Development Order

```
Phase 1 (Compiler Bare-Metal)    ← START HERE. Everything depends on this.
  ↓
Phase 2 (Microkernel)            ← Needs bare-metal compiler output
  ↓
Phase 3 (HAL Drivers)            ← Needs working microkernel + first hardware tests
  ↓
Phase 4 (Storage)      ┐
Phase 5 (Network)      ├── Can overlap (all need Phase 3 HAL)
Phase 6 (Display)      │
Phase 7 (AI Subsystem) ┘
  ↓
Phase 8 (OS Services)            ← Needs filesystem + network + display
  ↓
Phase 9 (Shell & Apps)           ← Needs all services
  ↓
Phase 10 (Production)            ← Final integration + release
```

---

## 15. Branching Strategy

```
main                          ← stable releases only (tagged v3.0.0-alpha.N)
develop/fajaros               ← integration branch for FajarOS work
fajaros/phase1-compiler       ← Phase 1: compiler bare-metal support
fajaros/phase2-kernel         ← Phase 2: microkernel
fajaros/phase3-hal            ← Phase 3: HAL drivers
fajaros/phase4-storage        ← Phase 4: storage & filesystem
fajaros/phase5-network        ← Phase 5: network stack
fajaros/phase6-display        ← Phase 6: display & input
fajaros/phase7-ai             ← Phase 7: AI subsystem
fajaros/phase8-services       ← Phase 8: OS services
fajaros/phase9-apps           ← Phase 9: shell & applications
fajaros/phase10-release       ← Phase 10: production & release
feat/S01-aarch64-target       ← Per-sprint feature branches
fix/XXX                       ← Bugfix branches
release/v3.0.0                ← Release preparation
```

### Commit Convention

```
Format: <type>(<scope>): <description>

Types: feat, fix, test, refactor, docs, perf, ci, chore
Scope: kernel, boot, mmu, gic, sched, ipc, gpio, uart, spi, i2c,
       timer, dma, pcie, nvme, sd, vfs, fat32, ext4, eth, tcp,
       ip, dhcp, http, ssh, hdmi, usb, compositor, camera, npu,
       gpu, init, devmgr, users, power, shell, apps

Examples:
  feat(kernel): implement 4-level aarch64 page table
  feat(boot): parse UEFI memory map and init kernel heap
  feat(gic): implement GICv3 distributor initialization
  feat(gpio): add TLMM GPIO driver for QCS6490
  feat(uart): implement QUP UART with interrupt-driven RX
  feat(npu): implement FastRPC session for Hexagon 770
  fix(mmu): correct page table entry attribute index
  test(sched): add round-robin scheduling QEMU test
  docs(kernel): document exception vector table layout
```

---

## 16. Troubleshooting Quick Reference

| Problem | Cause | Solution |
|---------|-------|----------|
| QEMU hangs at boot (no output) | _start does not reach kernel_main | Check linker script ENTRY(), verify SP setup, add early serial print |
| QEMU "Trying to execute code outside RAM" | ELF loaded at wrong address | Check `.text` address in linker script matches QEMU -M virt memory |
| Data abort at MMIO address | MMIO region not mapped in page table | Add identity map for peripheral region (0x0000_0000 - 0x0FFF_FFFF) |
| Timer IRQ never fires | GIC not configured or timer not enabled | Check GICD, GICR, ICC_PMR, CNTV_CTL_EL0 enable bit |
| MMU enable hangs | Invalid page table or MAIR/TCR mismatch | Verify MAIR_EL1 attr indices match PTE AttrIndx, check TCR T0SZ |
| Context switch crashes | Register save/restore incomplete | Verify all 31 GP regs + SP + ELR + SPSR saved, 16-byte SP alignment |
| GPIO does nothing on Q6A | TLMM not configured for GPIO function | Check TLMM CFG register function select, check GPIO OE bit |
| UART no output on Q6A | Wrong QUP engine or baud rate | Verify QUP base address, check clock divider, try 115200 baud |
| I2C NACK | Wrong address or device not powered | Check 7-bit address (not shifted), verify pull-ups, check Vcc |
| NPU inference fails | FastRPC session not established | Verify CDSP is powered on, check QNN library paths |
| GPU compute hangs | Adreno not clocked or wrong submission | Verify GPU GDSC is on, check ring buffer pointer, fence signaling |
| Kernel heap exhaustion | Memory leak in kernel allocator | Add alloc/free tracking, check for missing frees after DMA |
| Binary too large (> 16MB) | Debug symbols or unstripped | Use `--release` profile, strip debug info for production |

---

*V30_WORKFLOW.md v1.0 — Development workflow for FajarOS V3.0 "Surya"*
*Target: Radxa Dragon Q6A (QCS6490) | 10 phases, 42 sprints, 420 tasks*
*Author: Fajar (PrimeCore.id) + Claude Opus 4.6 | Created: 2026-03-12*
