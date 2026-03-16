# V30_RULES.md — FajarOS "Surya" OS-Specific Coding Rules

> These rules are **NON-NEGOTIABLE**. Every line of FajarOS kernel, driver, service, and AI code must comply.
> This is kernel code — bugs mean hardware lockups, data corruption, or security breaches.
> No exceptions without explicit sign-off from the project lead.
>
> **Priority:** CORRECTNESS > SAFETY > DETERMINISM > PERFORMANCE
> "If it compiles in Fajar Lang, it is safe to deploy on the QCS6490."

---

## 0. Applicability

| Layer | Context | This Document Applies? |
|-------|---------|----------------------|
| Layer 1: Microkernel | `@kernel` | **YES — Sections 1, 5, 6, 7, 8, 9, 10** |
| Layer 2: HAL Drivers | `@kernel` | **YES — Sections 1, 2, 5, 6, 7, 8, 9, 10** |
| Layer 3: OS Services | `@safe` / `@device` | **YES — Sections 3, 4, 5, 6, 7, 8, 9, 10** |
| Layer 4: Applications | `@safe` | **YES — Sections 4, 7, 8, 9, 10** |
| Compiler additions | Rust (host) | `docs/V1_RULES.md` applies; this document for OS-specific concerns only |

All rules in `docs/V1_RULES.md` (general Fajar Lang coding rules) STILL APPLY in addition to these OS-specific rules. Where conflict exists, V30_RULES.md takes precedence for FajarOS code.

---

## 1. Kernel Code Rules (`@kernel` context)

> Kernel code runs at EL1 with full hardware access. A single bug here can brick the board,
> corrupt memory, or create a security hole. Every rule exists because the alternative is a
> hardware lockup or silent data corruption.

### 1.1 No Heap Allocation

```fajar
// FORBIDDEN in @kernel
let s = String::new()              // ERROR KE001: heap allocation in kernel
let v = Vec::new()                 // ERROR KE001: heap allocation in kernel
let b = Box::new(42)               // ERROR KE001: heap allocation in kernel

// REQUIRED: use kernel_alloc with explicit size
let buf = kernel_alloc(4096)       // explicit 4KB allocation from kernel heap
let page = page_alloc(1)           // allocate 1 physical page (4KB)
```

- **Compiler enforces KE001** — any heap-allocating type in `@kernel` is a compile error.
- `kernel_alloc(size: usize)` allocates from the kernel's buddy allocator with explicit byte count.
- `page_alloc(count: usize)` allocates physically contiguous pages.
- Every `kernel_alloc` MUST have a corresponding `kernel_free` in the same scope or an explicit ownership transfer comment.
- Stack-allocated arrays and fixed-size buffers are preferred over any dynamic allocation.

### 1.2 No Panic in Kernel

```fajar
// FORBIDDEN in @kernel
panic("something went wrong")      // ERROR: kernel must never panic
assert(x > 0)                      // ERROR: assert panics on failure

// REQUIRED: return error codes
@kernel fn page_map(table: *mut PageTable, vaddr: u64, paddr: u64) -> Result<void, KernelError> {
    if vaddr % PAGE_SIZE != 0 {
        return Err(KernelError::UnalignedAddress(vaddr))  // KE102
    }
    // ...
    Ok(())
}

// ALLOWED: kernel_halt() for truly unrecoverable situations (triple fault, etc.)
// Every call to kernel_halt MUST have a preceding serial print explaining WHY
@kernel fn handle_triple_fault(esr: u64, elr: u64) {
    serial_print("FATAL: triple fault at ")
    serial_print_hex(elr)
    kernel_halt()  // wfe loop — requires physical power cycle
}
```

- Return `Result<T, KernelError>` from every kernel function.
- `assert()` and `panic()` are **compile errors** in `@kernel` context.
- `debug_assert()` is allowed but compiles to no-op in release builds.
- `kernel_halt()` is the ONLY permitted abort — used exclusively for unrecoverable hardware failures.
- Every `kernel_halt()` call site MUST print diagnostic info to serial BEFORE halting.

### 1.3 Bounded Loops Only

```fajar
// FORBIDDEN in @kernel
while condition {                  // ERROR: unbounded loop in kernel
    // ...
}

for item in unbounded_iterator {   // ERROR: unbounded iteration in kernel
    // ...
}

// REQUIRED: explicit maximum iteration count
@kernel fn poll_uart_ready(port: u32) -> Result<void, DriverError> {
    for _i in 0..10000 {           // bounded: max 10,000 iterations
        if volatile_read(port + UART_STATUS) & TX_READY != 0 {
            return Ok(())
        }
    }
    Err(DriverError::Timeout)      // DE101: hardware timeout
}

// REQUIRED: loop with explicit bound annotation
@kernel fn wait_for_irq() -> Result<void, KernelError> {
    let mut attempts: u32 = 0
    loop @bounded(100000) {        // compiler enforces max iterations
        if gic_pending() {
            return Ok(())
        }
        attempts += 1
        if attempts >= 100000 {
            return Err(KernelError::IrqTimeout)  // KE110
        }
    }
}
```

- Every `while`, `for`, and `loop` in `@kernel` MUST have a provable upper bound.
- The compiler emits **KE103 UnboundedLoop** if it cannot verify a finite iteration count.
- Use `loop @bounded(N)` annotation for loops where the bound is not syntactically obvious.
- Preferred pattern: `for _i in 0..MAX_ITERATIONS` with early return on success.
- Maximum bound for any single poll loop: **1,000,000 iterations** (approximately 1ms at 2.7GHz).

### 1.4 No Recursion in Kernel

```fajar
// FORBIDDEN in @kernel
@kernel fn traverse_page_table(table: *mut PageTable, level: u32) {
    // ERROR KE104: recursion in kernel (stack is 64KB)
    traverse_page_table(child, level + 1)
}

// REQUIRED: iterative with explicit stack
@kernel fn traverse_page_table(root: *mut PageTable) -> Result<void, KernelError> {
    let mut stack: [TraversalFrame; 4] = zero_init()  // max 4 levels, stack-allocated
    let mut depth: u32 = 0
    // ... iterative traversal using explicit stack array
}
```

- Kernel stack is **64KB** — recursion can overflow it with no guard page recovery at EL1.
- **ALL** kernel functions MUST be non-recursive. The compiler emits **KE104 RecursionInKernel** for any direct or indirect recursion detected in the call graph.
- Use iterative algorithms with explicit, fixed-size stack arrays.
- Maximum call depth in kernel: **32 frames** (enforced by static analysis).

### 1.5 Inline Assembly Safety

```fajar
// FORBIDDEN: asm without SAFETY comment
@kernel fn enable_mmu() {
    asm!("msr SCTLR_EL1, {0}", in(reg) sctlr)  // ERROR: missing SAFETY comment
}

// REQUIRED: every asm block has SAFETY comment
@kernel fn enable_mmu(sctlr: u64) {
    // SAFETY: SCTLR_EL1 controls MMU/cache enable bits. The caller MUST have:
    //   1. Set up valid page tables in TTBR0_EL1/TTBR1_EL1
    //   2. Configured MAIR_EL1 and TCR_EL1
    //   3. Ensured identity mapping covers current PC
    // Failure to meet these invariants will cause an immediate MMU fault (data abort).
    asm!("msr SCTLR_EL1, {0}", in(reg) sctlr)
    // SAFETY: ISB required after SCTLR write to ensure pipeline sees new MMU state.
    asm!("isb")
}
```

- Every `asm!()` block MUST have a `// SAFETY:` comment immediately preceding it.
- The SAFETY comment MUST describe:
  1. What hardware state is being modified
  2. What invariants the caller must uphold
  3. What happens if invariants are violated
- **No bare `asm!()` without register constraints** — every operand must use `in(reg)`, `out(reg)`, `inout(reg)`, or named registers like `in("x0")`.
- System register writes MUST be followed by `isb` if they affect instruction fetch or MMU.
- Memory barrier instructions (`dmb`, `dsb`) MUST specify the correct domain (`sy`, `ish`, `osh`).

### 1.6 Volatile I/O Memory Ordering

```fajar
// FORBIDDEN: volatile access without ordering specification
@kernel fn read_status(base: u64) -> u32 {
    volatile_read(base + 0x04)     // ERROR: no memory ordering specified
}

// REQUIRED: specify memory ordering for every volatile access
@kernel fn read_status(base: u64) -> u32 {
    // Read device status register. Device-nGnRnE memory type ensures
    // no reordering with other device accesses.
    volatile_read::<u32>(base + REG_STATUS)  // uses DeviceMemory ordering (MAIR index 0)
}

// REQUIRED: ordering barriers for multi-register sequences
@kernel fn send_command(base: u64, cmd: u32, data: u32) {
    volatile_write::<u32>(base + REG_DATA, data)
    // ORDERING: DSB ensures data write completes before command write.
    // Without this, the device may read stale data.
    dsb(sy)
    volatile_write::<u32>(base + REG_CMD, cmd)
}
```

- Every `volatile_read()` and `volatile_write()` MUST operate on memory mapped as Device-nGnRnE (MAIR index 0) or Device-nGnRE (MAIR index 1).
- Multi-register access sequences MUST use explicit ordering barriers:
  - `dsb(sy)` — full system data synchronization barrier
  - `dmb(sy)` — data memory barrier (lighter than DSB)
  - `isb` — instruction synchronization barrier (after system register writes)
- The ordering requirement MUST be documented in a comment at each barrier site.
- MMIO regions MUST be mapped with the correct memory type attribute. Mapping device memory as Normal (cacheable) is a **critical bug** — it causes silent data corruption.

### 1.7 IRQ Handler Reentrancy

```fajar
// RULE: IRQ handlers MUST be reentrant-safe
// RULE: IRQ handlers MUST NOT acquire locks that non-IRQ code holds

@kernel fn irq_uart_rx(port: u32) {
    // FORBIDDEN: calling functions that may block or allocate
    let s = String::new()          // ERROR: heap allocation in IRQ
    spinlock_acquire(global_lock)  // ERROR: may deadlock if interrupted thread holds it

    // REQUIRED: use IRQ-safe data structures only
    let byte = volatile_read::<u8>(port + UART_RX_FIFO)
    ring_buffer_push(uart_rx_buf, byte)  // lock-free ring buffer, no allocation
}
```

- IRQ handlers MUST NOT:
  - Allocate memory (heap or page)
  - Acquire any spinlock that non-IRQ code also acquires (unless IRQs are disabled first)
  - Call any function that may block or sleep
  - Access any data structure that is not lock-free or IRQ-safe
  - Perform floating-point operations (FP registers not saved by default)
- IRQ handlers MUST:
  - Complete in bounded time (see Section 1.8)
  - Acknowledge the interrupt source before returning (write EOI to GICv3)
  - Use only lock-free data structures (ring buffers, atomic counters)
  - Preserve all callee-saved registers (enforced by exception vector stub)
- Nested IRQ: permitted only with strict priority ordering. Lower-priority IRQs MUST NOT preempt higher-priority handlers.

### 1.8 Interrupt Disable Duration

```fajar
// RULE: interrupts may be disabled for at most 10 microseconds

@kernel fn critical_section() -> Result<void, KernelError> {
    let daif = irq_disable()       // save DAIF, mask IRQs

    // Critical section: < 10μs of work
    // At 2.7GHz (Kryo 670 prime core), 10μs ≈ 27,000 cycles
    volatile_write::<u32>(GICD_BASE + GICD_ISENABLER, 1 << irq)
    volatile_write::<u32>(GICD_BASE + GICD_IPRIORITYR, priority)

    irq_restore(daif)              // restore previous DAIF state
    Ok(())
}
```

- **Maximum interrupt-disabled duration: 10 microseconds (10μs).**
- At 2.7GHz (Kryo 670 prime core), 10μs = ~27,000 cycles.
- Any critical section exceeding 10μs MUST be refactored to use finer-grained locking or lock-free algorithms.
- The compiler SHOULD emit a warning (**KE105 LongCriticalSection**) if static analysis cannot prove the critical section is shorter than 10μs.
- `irq_disable()` MUST save the previous DAIF state; `irq_restore()` MUST restore it — never use raw `msr DAIFSet` without saving.
- Timer IRQ latency target: < 50μs from interrupt assertion to handler entry.

---

## 2. Driver Code Rules (`@kernel` context)

> Drivers interact with real hardware. A wrong register write can physically damage
> peripherals, corrupt flash storage, or cause electrical shorts on GPIO pins.

### 2.1 Volatile Register Access

```fajar
// FORBIDDEN: non-volatile access to hardware registers
@kernel fn read_gpio(base: u64) -> u32 {
    let ptr = base + 0x04 as *const u32
    *ptr                           // ERROR: non-volatile read of MMIO register
}

// REQUIRED: all register access through volatile_read/volatile_write
@kernel fn read_gpio(pin: u32) -> bool {
    let val = volatile_read::<u32>(TLMM_BASE + TLMM_IN_OUT(pin))
    (val & GPIO_IN_BIT) != 0
}
```

- **Every** hardware register access MUST use `volatile_read()` or `volatile_write()`.
- Direct pointer dereference of MMIO addresses is a compile error in driver code.
- The compiler SHOULD warn if a raw pointer to an MMIO region is dereferenced without volatile semantics.
- Rationale: without volatile, the compiler may optimize away reads, reorder writes, or cache register values — all catastrophic for hardware interaction.

### 2.2 MMIO Base Addresses as Constants

```fajar
// FORBIDDEN: computed or runtime MMIO addresses
@kernel fn uart_init(base: u64) {  // ERROR: base address not const
    volatile_write::<u32>(base + 0x24, 0x01)
}

// REQUIRED: all MMIO base addresses as const with descriptive names
const TLMM_BASE: u64 = 0x0F10_0000          // QCS6490 Top-Level Mode Mux
const QUP_BASE: u64 = 0x0A8C_0000           // QCS6490 Qualcomm Unified Peripheral
const GICD_BASE: u64 = 0x1780_0000          // GICv3 Distributor
const GICR_BASE: u64 = 0x17A0_0000          // GICv3 Redistributor
const PCIE_BASE: u64 = 0x0100_0000          // PCIe controller
const ADRENO_BASE: u64 = 0x3D00_0000        // Adreno 643 GPU
const CDSP_BASE: u64 = 0x0B00_0000          // Hexagon 770 compute DSP

// Register offsets as const within driver module
const REG_UART_TX: u64 = 0x00
const REG_UART_RX: u64 = 0x04
const REG_UART_STATUS: u64 = 0x08
const REG_UART_CTRL: u64 = 0x0C

@kernel fn uart_write_byte(byte: u8) -> Result<void, DriverError> {
    for _i in 0..10000 {
        if volatile_read::<u32>(QUP_BASE + REG_UART_STATUS) & TX_READY != 0 {
            volatile_write::<u32>(QUP_BASE + REG_UART_TX, byte as u32)
            return Ok(())
        }
    }
    Err(DriverError::UartTimeout)  // DE101
}
```

- All MMIO base addresses MUST be `const` values — never computed at runtime.
- Base address constants MUST include a comment naming the hardware block.
- Register offset constants MUST use `REG_` prefix and SCREAMING_CASE.
- The full register address is always `BASE + REG_OFFSET` — never a magic number.
- Rationale: a wrong MMIO address can write to arbitrary physical memory, potentially corrupting kernel state or damaging hardware.

### 2.3 DMA Buffer Cache Coherency

```fajar
// FORBIDDEN: DMA without cache management
@kernel fn dma_transfer(src: *mut u8, dst_phys: u64, len: usize) {
    dma_start(channel, src, dst_phys, len)  // ERROR: cache not flushed
    dma_wait(channel)
}

// REQUIRED: cache flush before DMA write, invalidate after DMA read
@kernel fn dma_write_to_device(buf: *mut u8, dev_addr: u64, len: usize) -> Result<void, DriverError> {
    // Flush CPU cache to ensure DMA controller sees latest data
    cache_flush(buf, len)
    dsb(sy)                        // ensure cache operations complete
    dma_start(channel, buf as u64, dev_addr, len, DMA_MEM_TO_DEV)
    let result = dma_wait_timeout(channel, 1000)  // 1000ms timeout
    result
}

@kernel fn dma_read_from_device(dev_addr: u64, buf: *mut u8, len: usize) -> Result<void, DriverError> {
    // Invalidate cache BEFORE DMA read to prevent stale reads
    cache_invalidate(buf, len)
    dsb(sy)
    dma_start(channel, dev_addr, buf as u64, len, DMA_DEV_TO_MEM)
    let result = dma_wait_timeout(channel, 1000)
    // Invalidate again AFTER DMA completes to ensure CPU sees new data
    cache_invalidate(buf, len)
    dsb(sy)
    result
}
```

- **Before DMA write** (memory → device): `cache_flush()` + `dsb(sy)` to push CPU data to RAM.
- **Before DMA read** (device → memory): `cache_invalidate()` to prevent stale cache hits.
- **After DMA read**: `cache_invalidate()` + `dsb(sy)` to ensure CPU sees DMA-written data.
- DMA buffers MUST be allocated from uncacheable memory regions when possible (MAIR Device-nGnRnE).
- DMA buffer addresses MUST be physically contiguous — use `dma_alloc_coherent()` which returns both virtual and physical addresses.
- DMA buffer alignment: minimum 64 bytes (cache line size on Kryo 670).

### 2.4 Error Return, Never Panic

```fajar
// FORBIDDEN: any form of panic/assert/unwrap in driver code
@kernel fn spi_transfer(port: u32, tx: *const u8, rx: *mut u8, len: usize) {
    assert(len <= 4096)            // ERROR: assert in driver
    // ...
}

// REQUIRED: all driver functions return Result
@kernel fn spi_transfer(port: u32, tx: *const u8, rx: *mut u8, len: usize) -> Result<usize, DriverError> {
    if len > SPI_MAX_TRANSFER {
        return Err(DriverError::TransferTooLarge(len, SPI_MAX_TRANSFER))  // DE102
    }
    if len == 0 {
        return Err(DriverError::ZeroLengthTransfer)  // DE103
    }
    // ...
    Ok(bytes_transferred)
}
```

- Every driver function MUST return `Result<T, DriverError>`.
- No `panic()`, `assert()`, `unwrap()`, or `expect()` in any driver code.
- Driver errors MUST include the specific hardware context (port number, register value, etc.).
- On unrecoverable hardware failure, the driver MUST:
  1. Log the failure to serial console
  2. Disable the failed peripheral (power gate if possible)
  3. Return an error to the caller — never halt

### 2.5 Timeout on All Hardware Waits

```fajar
// FORBIDDEN: infinite wait on hardware
@kernel fn wait_for_ready(base: u64) {
    while volatile_read::<u32>(base + REG_STATUS) & READY == 0 {
        // infinite loop if hardware is stuck
    }
}

// REQUIRED: bounded wait with timeout
const POLL_TIMEOUT_US: u64 = 100_000  // 100ms maximum hardware wait

@kernel fn wait_for_ready(base: u64) -> Result<void, DriverError> {
    let start = timer_get_ticks()
    let timeout_ticks = us_to_ticks(POLL_TIMEOUT_US)
    for _i in 0..1_000_000 {
        if volatile_read::<u32>(base + REG_STATUS) & READY != 0 {
            return Ok(())
        }
        if timer_get_ticks() - start > timeout_ticks {
            return Err(DriverError::HardwareTimeout {   // DE101
                register: base + REG_STATUS,
                expected: READY,
                actual: volatile_read::<u32>(base + REG_STATUS),
            })
        }
    }
    Err(DriverError::HardwareTimeout {
        register: base + REG_STATUS,
        expected: READY,
        actual: volatile_read::<u32>(base + REG_STATUS),
    })
}
```

- Every hardware poll loop MUST have a timeout.
- Default timeout: **100ms** for most peripherals, **1000ms** for storage devices, **5000ms** for network link-up.
- Timeout errors MUST include: register address, expected value, actual value read.
- After timeout, the driver MUST attempt to reset the peripheral before returning error.
- For interrupt-driven drivers, use `wait_for_irq_timeout(irq, timeout_ms)` instead of polling.

### 2.6 Register Layout Documentation

```fajar
// REQUIRED: document every register struct with bitfield layout

/// QCS6490 TLMM GPIO Configuration Register (per-pin)
/// Address: TLMM_BASE + 0x1000 * pin
///
/// Bit layout:
/// [31:17] Reserved
/// [16:14] DRV_STRENGTH  — Drive strength (2mA to 16mA)
/// [13]    OE            — Output enable (1=output, 0=input)
/// [12:10] FUNC_SEL      — Function select (0=GPIO, 1-7=alt functions)
/// [9]     Reserved
/// [8:6]   PULL          — Pull configuration (00=none, 01=down, 10=keeper, 11=up)
/// [5:0]   Reserved
const TLMM_GPIO_CFG_OE: u32 = 1 << 13
const TLMM_GPIO_CFG_FUNC_MASK: u32 = 0x7 << 10
const TLMM_GPIO_CFG_PULL_MASK: u32 = 0x7 << 6
const TLMM_GPIO_CFG_DRV_MASK: u32 = 0x7 << 14

/// GPIO In/Out Register
/// [1] GPIO_OUT — Write 1=high, 0=low
/// [0] GPIO_IN  — Read current pin level
const GPIO_OUT_BIT: u32 = 1 << 1
const GPIO_IN_BIT: u32 = 1 << 0
```

- Every register MUST have a doc comment showing the full bitfield layout.
- Bit positions MUST be documented with `[MSB:LSB]` notation.
- Register field constants MUST use shift-and-mask patterns: `value << SHIFT` and `MASK << SHIFT`.
- Struct-based register representations MUST match the hardware manual exactly — field names should correspond to the QCS6490 TRM (Technical Reference Manual).
- Reserved bits MUST be preserved when writing: read-modify-write, never blind-write.

### 2.7 Test on QEMU Before Hardware

```fajar
// RULE: Every driver has a QEMU test AND a hardware test path

// QEMU test (runs in CI, no hardware required)
@test fn test_uart_echo_qemu() {
    // Uses QEMU PL011 UART at 0x0900_0000
    let result = uart_init(QEMU_UART_BASE, 115200)
    assert_eq(result, Ok(()))
    uart_write_byte(0x41)  // 'A'
    // QEMU serial output verified by test harness
}

// Hardware test (runs on Q6A only, gated by #[cfg(target_board = "dragon-q6a")])
@test @hardware fn test_uart_echo_q6a() {
    // Uses QCS6490 QUP UART5 at pins 8/10
    let result = uart_init(QUP_UART5_BASE, 115200)
    assert_eq(result, Ok(()))
    uart_write_byte(0x41)
    let echo = uart_read_byte_timeout(1000)
    assert_eq(echo, Ok(0x41))
}
```

- Every driver MUST have tests that pass on QEMU `-M virt -cpu cortex-a76`.
- Hardware-specific tests MUST be gated with `@hardware` annotation and only run on the physical board.
- QEMU tests run in CI on every commit — hardware tests run on manual deploy.
- Driver development cycle: QEMU first, then hardware verification.

---

## 3. AI Code Rules (`@device` context)

> AI code handles tensor data, NPU inference, and GPU compute. Bugs here cause
> incorrect inference results, NPU firmware crashes, or GPU hangs.

### 3.1 No Raw Pointers

```fajar
// ENFORCED BY COMPILER: @device context forbids raw pointers
@device fn process_tensor(data: *mut f32) {  // ERROR DE001: raw pointer in @device
    // ...
}

// REQUIRED: use typed tensor references
@device fn process_tensor(data: Tensor<f32, [1, 3, 224, 224]>) -> Tensor<f32, [1, 1000]> {
    // typed, shape-checked, safe
    let result = model.forward(data)
    result
}
```

- The compiler enforces **DE001 RawPointerInDevice** — no `*mut T`, `*const T`, or `addr` in `@device` context.
- All data passing uses typed Tensor references, slices, or IPC messages.
- DMA buffer access from `@device` code goes through a safe abstraction: `DmaBuffer<T>` which handles cache coherency internally.
- Rationale: AI code should never be able to corrupt kernel memory or hardware state.

### 3.2 Compile-Time Tensor Shape Checking

```fajar
// REQUIRED: tensor shapes checked at compile time where possible
@device fn classify(image: Tensor<f32, [1, 3, 224, 224]>) -> Tensor<f32, [1, 1000]> {
    let features = conv2d(image, weights_conv1)    // shape: [1, 64, 112, 112]
    let flat = flatten(features)                   // shape: [1, 64*112*112]
    let logits = dense(flat, weights_fc)           // shape: [1, 1000]
    softmax(logits)                                // shape: [1, 1000]
}

// COMPILE ERROR: shape mismatch
@device fn bad_classify(image: Tensor<f32, [1, 3, 224, 224]>) -> Tensor<f32, [1, 10]> {
    let logits = dense(flatten(image), weights_fc) // ERROR TE001: output shape [1, 1000] != declared [1, 10]
}
```

- Tensor shapes MUST be declared in function signatures where dimensions are known.
- The compiler checks shape propagation through operations at compile time.
- For dynamic shapes (batch size varies), use `Tensor<f32, [?, 3, 224, 224]>` where `?` is a runtime dimension.
- Shape errors are **TE001-TE008** — caught at compile time, not runtime.

### 3.3 Fallible NPU Model Loading

```fajar
// FORBIDDEN: ignoring NPU load errors
@device fn init_model() {
    let model = npu_load("/opt/fj/models/resnet50.bin")  // ERROR: unhandled Result
}

// REQUIRED: handle all NPU errors explicitly
@device fn init_model() -> Result<NpuModel, NpuError> {
    let model = npu_load("/opt/fj/models/resnet50.bin")?  // NE101 if file missing
    // NE102 if model format invalid
    // NE103 if NPU firmware not loaded
    // NE104 if out of NPU memory
    npu_validate(model)?           // verify model compatibility with Hexagon V68
    Ok(model)
}
```

- `npu_load()`, `npu_infer()`, and `npu_unload()` ALL return `Result<T, NpuError>`.
- Every NPU call MUST handle the error case — the `?` operator or explicit `match`.
- NPU operations can fail for hardware reasons (DSP firmware crash, memory exhaustion, power throttling) — the caller MUST have a fallback strategy.
- Recommended pattern: NPU inference with CPU fallback.

### 3.4 GPU Dispatch Synchronization

```fajar
// FORBIDDEN: reading GPU results without sync
@device fn gpu_matmul(a: Tensor, b: Tensor) -> Tensor {
    gpu_dispatch(matmul_kernel, a, b)
    let result = gpu_read_output()   // ERROR: no sync point — data race
}

// REQUIRED: explicit sync before reading GPU results
@device fn gpu_matmul(a: Tensor, b: Tensor) -> Result<Tensor, GpuError> {
    let cmd = gpu_dispatch(matmul_kernel, a, b)?  // GE101 if dispatch fails
    gpu_sync(cmd)?                                 // GE102 if GPU hangs (5s timeout)
    let result = gpu_read_output(cmd)?             // GE103 if output buffer invalid
    Ok(result)
}
```

- Every `gpu_dispatch()` MUST be followed by `gpu_sync()` before reading results.
- `gpu_sync()` has a **5-second** default timeout — returns `GpuError::Timeout (GE102)` if GPU hangs.
- Multiple dispatches can be batched: dispatch N kernels, then sync once.
- GPU error recovery: on timeout, attempt GPU reset before returning error.

### 3.5 Camera DMA Buffers: Zero-Copy Only

```fajar
// FORBIDDEN: copying camera frame data
@device fn process_frame(cam: Camera) -> Result<Tensor, CamError> {
    let frame = cam.capture()?
    let copy = mem_copy(frame.data, frame.len)     // ERROR: memcpy of camera frame
    // ...
}

// REQUIRED: zero-copy pipeline
@device fn process_frame(cam: Camera) -> Result<Tensor, CamError> {
    let frame = cam.capture()?                      // DMA buffer, zero-copy
    let tensor = Tensor::from_dma_buffer(frame)?    // reinterpret buffer as tensor, no copy
    let result = npu_infer(model, tensor)?          // NPU reads directly from DMA buffer
    cam.release_buffer(frame)                       // return buffer to camera DMA pool
    Ok(result)
}
```

- Camera frames are delivered via DMA into pre-allocated buffers — **never copy** frame data.
- `Tensor::from_dma_buffer()` creates a tensor view over the DMA buffer without copying.
- After processing, the DMA buffer MUST be returned to the camera pool via `release_buffer()`.
- Camera DMA buffers are typically 1920x1080x3 = ~6MB each — copying wastes bandwidth and time.
- Maximum camera pipeline latency target: **33ms** (30 FPS) from capture to inference result.

---

## 4. Service Code Rules (`@safe` context)

> Services run in user space with no direct hardware access. They communicate with
> the kernel and drivers via syscalls and IPC.

### 4.1 Syscall Error Handling

```fajar
// FORBIDDEN: ignoring syscall errors
@safe fn write_to_file(fd: i32, data: str) {
    sys_write(fd, data.as_bytes(), data.len())     // ERROR: unhandled Result
}

// REQUIRED: all syscalls return Result<T, SysError>
@safe fn write_to_file(fd: i32, data: str) -> Result<usize, SysError> {
    let written = sys_write(fd, data.as_bytes(), data.len())?
    // SE101 BadFd, SE102 WouldBlock, SE103 Interrupted, SE104 NoSpace
    if written < data.len() {
        return Err(SysError::ShortWrite { expected: data.len(), actual: written })  // SE105
    }
    Ok(written)
}
```

- Every syscall returns `Result<T, SysError>` — there are no void syscalls.
- `SysError` includes the syscall number and a human-readable description.
- Partial success (short read/write) MUST be handled — never assume full transfer.
- Retry on `SE103 Interrupted` (signal interrupted syscall) is the caller's responsibility.

### 4.2 IPC Message Size Limits

```fajar
// RULE: IPC message payload is exactly 256 bytes maximum
const IPC_MAX_PAYLOAD: usize = 256

struct IpcMessage {
    sender: u32,       // 4 bytes — sender PID
    receiver: u32,     // 4 bytes — receiver PID
    msg_type: u32,     // 4 bytes — message type code
    payload_len: u32,  // 4 bytes — actual payload length (0..256)
    payload: [u8; 256] // 256 bytes — fixed-size payload buffer
}
// Total: 272 bytes per message

// For data larger than 256 bytes: use shared memory IPC
@safe fn send_large_data(dest: u32, data: *const u8, len: usize) -> Result<void, IpcError> {
    let shm = sys_shm_create(len)?                // allocate shared memory
    sys_shm_write(shm, data, len)?                // copy data to shared memory
    let msg = IpcMessage::new_shm_transfer(dest, shm, len)
    sys_ipc_send(dest, msg)?                      // send handle, not data
    Ok(())
}
```

- IPC inline payload: **256 bytes maximum**. Attempting to send more is `IpcError::PayloadTooLarge`.
- For data > 256 bytes: use shared memory IPC — send a shared memory handle in the message payload.
- IPC message queues: **64 messages** per process. Queue full returns `IpcError::QueueFull`.
- IPC timeout: `sys_ipc_recv_timeout(timeout_ms)` — never block indefinitely without timeout.

### 4.3 File Descriptor Discipline (RAII via Drop)

```fajar
// FORBIDDEN: manual fd management without cleanup
@safe fn read_config() -> Result<str, SysError> {
    let fd = sys_open("/etc/fajaros.conf", O_RDONLY)?
    let data = sys_read(fd, buffer, 4096)?
    // BUG: fd leaked if sys_read returns error!
    sys_close(fd)?
    Ok(data)
}

// REQUIRED: use File struct with Drop (RAII)
@safe fn read_config() -> Result<str, SysError> {
    let file = File::open("/etc/fajaros.conf")?  // fd acquired
    let data = file.read_all()?                   // if this errors, fd still closed
    Ok(data)
    // file goes out of scope here → Drop calls sys_close(fd) automatically
}
```

- File descriptors MUST be wrapped in a struct that implements `Drop` for automatic cleanup.
- Maximum open fds per process: **256**. Opening more returns `SysError::TooManyOpenFiles (SE106)`.
- Standard fds: 0 = stdin, 1 = stdout, 2 = stderr — always open, never close.
- Socket fds follow the same rules — wrapped in `Socket` struct with `Drop`.

### 4.4 Network Buffer Bounds

```fajar
// FORBIDDEN: unbounded network receive
@safe fn handle_request(sock: Socket) -> Result<str, NetError> {
    let data = sock.read_all()?    // ERROR: unbounded — could allocate gigabytes
}

// REQUIRED: bounded receive with explicit maximum
const MAX_REQUEST_SIZE: usize = 65536  // 64KB maximum request

@safe fn handle_request(sock: Socket) -> Result<str, NetError> {
    let data = sock.read_bounded(MAX_REQUEST_SIZE)?
    // Returns NetError::MessageTooLarge if sender exceeds limit
    Ok(data)
}
```

- All network reads MUST specify a maximum buffer size.
- `read_all()` is **forbidden** on network sockets — use `read_bounded(max)`.
- TCP receive buffer: maximum **64KB** per connection.
- UDP datagram: maximum **1500 bytes** (MTU-limited).
- Total network memory per process: limited by process memory quota.

---

## 5. Memory Safety Rules

> Memory bugs in an OS are not just crashes — they are security vulnerabilities.
> Every rule here prevents a class of exploits.

### 5.1 W^X (Write XOR Execute)

```fajar
// RULE: No memory page may be both writable and executable simultaneously

@kernel fn page_map(table: *mut PageTable, vaddr: u64, paddr: u64, attrs: PageAttrs) -> Result<void, KernelError> {
    // ENFORCED: W^X check
    if attrs.contains(PAGE_WRITE) && attrs.contains(PAGE_EXEC) {
        return Err(KernelError::WritableExecutable(vaddr))  // KE111
    }
    // ...
}
```

- **ABSOLUTE RULE:** No page may have both write (AP=01) and execute (PXN=0/UXN=0) permissions.
- Kernel .text: Read + Execute (never writable after boot).
- Kernel .data/.bss: Read + Write (never executable).
- User .text: Read + Execute (never writable).
- User .data/.bss/heap/stack: Read + Write (never executable).
- JIT compilation (if ever needed): write to buffer, then remap as read+execute with `dsb(ish)` + `isb`.
- Violation of W^X is **KE111** — a kernel panic-level error that halts the system.

### 5.2 Stack Guard Pages

```fajar
// RULE: every process stack has a guard page at its bottom

@kernel fn create_process_stack(size: usize) -> Result<StackInfo, KernelError> {
    let total = size + PAGE_SIZE                  // extra page for guard
    let base = page_alloc(total / PAGE_SIZE)?
    // Guard page: mapped with no permissions — any access triggers page fault
    page_map(table, base, base, PAGE_NONE)?       // guard page at bottom
    // Usable stack: above the guard page
    page_map(table, base + PAGE_SIZE, base + PAGE_SIZE, PAGE_READ | PAGE_WRITE)?
    Ok(StackInfo {
        guard: base,
        bottom: base + PAGE_SIZE,
        top: base + total,
    })
}
```

- Every process stack MUST have a **4KB guard page** at its lowest address.
- The guard page is mapped with **no permissions** (PTE invalid) — any access triggers a data abort.
- The kernel exception handler identifies stack overflow by checking if the faulting address falls in a guard page region.
- Kernel stack: 64KB with guard page. User stack: configurable, minimum 64KB, default 1MB, with guard page.
- Thread stacks: same rules apply — every thread gets its own guard page.

### 5.3 Kernel Heap: Buddy Allocator with Poison Patterns

```fajar
// RULE: kernel heap uses buddy allocator with debug poison patterns

const ALLOC_POISON: u8 = 0xAA    // freshly allocated memory filled with 0xAA
const FREE_POISON: u8 = 0xDD     // freed memory filled with 0xDD
const GUARD_PATTERN: u32 = 0xDEAD_BEEF  // guard word before/after each allocation

@kernel fn kernel_alloc(size: usize) -> Result<*mut u8, KernelError> {
    let actual_size = size + 8                    // 4-byte guard word before + after
    let ptr = buddy_alloc(actual_size)?
    // In debug builds: fill with alloc poison
    mem_set(ptr, ALLOC_POISON, actual_size)
    // Write guard words
    volatile_write::<u32>(ptr as u64, GUARD_PATTERN)
    volatile_write::<u32>((ptr as u64) + 4 + size as u64, GUARD_PATTERN)
    Ok(ptr + 4)                                   // return pointer past front guard
}

@kernel fn kernel_free(ptr: *mut u8, size: usize) {
    let actual_ptr = ptr - 4
    // Verify guard words (detects buffer overflow/underflow)
    let front = volatile_read::<u32>(actual_ptr as u64)
    let back = volatile_read::<u32>((ptr as u64) + size as u64)
    if front != GUARD_PATTERN || back != GUARD_PATTERN {
        serial_print("HEAP CORRUPTION DETECTED at ")
        serial_print_hex(ptr as u64)
        kernel_halt()
    }
    // In debug builds: fill with free poison (detects use-after-free)
    mem_set(actual_ptr, FREE_POISON, size + 8)
    buddy_free(actual_ptr, size + 8)
}
```

- Kernel heap: **64MB** starting at `0x4200_0000`, managed by buddy allocator.
- Minimum allocation: 64 bytes (cache-line aligned).
- Debug builds: allocated memory filled with `0xAA`, freed memory with `0xDD`.
- Guard words (`0xDEAD_BEEF`) before and after each allocation detect buffer overflow.
- Guard word corruption triggers immediate `kernel_halt()` with diagnostic output — heap corruption is unrecoverable.
- Release builds: poison patterns disabled for performance, guard words still active.

### 5.4 User/Kernel Heap Separation

```
Kernel Heap:  0x4200_0000 — 0x45FF_FFFF  (64MB, buddy allocator)
User Heap:    Per-process, in user VA space (0x8000_0000+), sbrk-style or mmap

RULE: kernel heap is NEVER accessible from user space (TTBR1 mapping, EL1 only)
RULE: user heap is NEVER accessible from kernel without explicit copy_from_user()
```

- Kernel and user heaps are in **separate address spaces** (TTBR0 for user, TTBR1 for kernel).
- Kernel code MUST NOT dereference user-space pointers directly — use `copy_from_user()` and `copy_to_user()` which validate the pointer range and handle page faults.
- `copy_from_user(kernel_buf, user_ptr, len)` returns `KernelError::BadUserPointer` if the user pointer is invalid.
- User processes have memory quotas — exceeding quota returns `SysError::OutOfMemory`.

### 5.5 DMA Buffer Safety

```fajar
// RULE: DMA buffers are mapped uncacheable and freed after use

@kernel fn dma_alloc_coherent(size: usize) -> Result<DmaBuffer, KernelError> {
    let phys = page_alloc_contiguous(size / PAGE_SIZE + 1)?
    let virt = kernel_va_alloc(size)?
    // Map as Device-nGnRnE (uncacheable, no reordering)
    page_map(kernel_table, virt, phys, PAGE_READ | PAGE_WRITE | PAGE_DEVICE)?
    Ok(DmaBuffer { virt, phys, size })
}

// RULE: DMA buffers implement Drop — freed automatically when scope ends
impl Drop for DmaBuffer {
    fn drop(self) {
        page_unmap(kernel_table, self.virt, self.size)
        page_free(self.phys, self.size / PAGE_SIZE + 1)
    }
}
```

- DMA buffers: mapped as Device memory (uncacheable) — no cache flush/invalidate needed.
- DMA buffers MUST be freed after use — enforced by `Drop` on `DmaBuffer` struct.
- DMA buffer lifetime MUST NOT exceed the driver operation that uses it.
- Maximum DMA buffer pool: **128MB** at `0x4800_0000` — exhaustion returns `KernelError::DmaPoolExhausted (KE112)`.
- DMA buffers for camera/NPU: allocated from a dedicated pool with IOMMU protection.

---

## 6. Concurrency Rules

> Concurrency bugs in a kernel are among the hardest to debug. A deadlock means
> the system freezes. A race condition means silent data corruption.

### 6.1 Spinlock Discipline

```fajar
// FORBIDDEN: spinlock without timeout
@kernel fn acquire_lock(lock: *mut Spinlock) {
    while !try_lock(lock) {}       // ERROR: unbounded spin
}

// FORBIDDEN: nested spinlocks
@kernel fn nested_locks() {
    spinlock_acquire(lock_a)
    spinlock_acquire(lock_b)       // ERROR: nested spinlock — deadlock risk
    spinlock_release(lock_b)
    spinlock_release(lock_a)
}

// REQUIRED: try_lock with timeout
@kernel fn acquire_lock(lock: *mut Spinlock) -> Result<SpinlockGuard, KernelError> {
    let start = timer_get_ticks()
    let timeout_ticks = us_to_ticks(100)  // 100μs max spin
    for _i in 0..100_000 {
        if try_lock(lock) {
            return Ok(SpinlockGuard::new(lock))
        }
        if timer_get_ticks() - start > timeout_ticks {
            return Err(KernelError::LockTimeout)  // KE113
        }
    }
    Err(KernelError::LockTimeout)
}
```

- **All spinlocks use `try_lock` with timeout** — never spin indefinitely.
- Maximum spin duration: **100 microseconds** (100μs). If the lock is not acquired within that time, return `KE113 LockTimeout`.
- **Spinlocks MUST NOT be nested.** If two locks are needed, redesign to use a single coarser lock or a lock-free algorithm. The compiler SHOULD emit **KE114 NestedSpinlock** if it detects nested acquisition.
- Spinlock acquisition order: if nesting is absolutely unavoidable (with explicit waiver), acquire in **alphabetical order by lock name** to prevent deadlocks.
- Spinlocks use `SpinlockGuard` with `Drop` for automatic release — never call `spinlock_release()` manually.

### 6.2 Interrupt-Safe Locks

```fajar
// FORBIDDEN: acquiring lock without disabling IRQs if IRQ handler uses same lock
@kernel fn update_process_list(proc: Process) {
    spinlock_acquire(process_lock)  // ERROR: IRQ could fire while held → deadlock
    // ...                          // if IRQ handler also acquires process_lock
}

// REQUIRED: disable IRQ before acquiring interrupt-shared lock
@kernel fn update_process_list(proc: Process) -> Result<void, KernelError> {
    let daif = irq_disable()       // save and disable interrupts
    let guard = spinlock_acquire_timeout(process_lock, 100)?
    // Critical section: IRQs disabled, lock held
    process_list_add(proc)
    drop(guard)                    // release lock
    irq_restore(daif)             // restore previous IRQ state
    Ok(())
}

// PREFERRED: use IrqSpinlock that automatically disables/restores IRQs
@kernel fn update_process_list(proc: Process) -> Result<void, KernelError> {
    let guard = irq_spinlock_acquire_timeout(process_lock, 100)?
    process_list_add(proc)
    Ok(())
    // guard drop: releases spinlock AND restores IRQ state
}
```

- If an IRQ handler and non-IRQ code share a lock, the non-IRQ code MUST disable interrupts before acquiring the lock.
- Use `IrqSpinlock` type which automatically saves DAIF, disables IRQs, acquires lock, and restores on drop.
- `irq_restore()` MUST restore the **previous** DAIF state, not blindly enable — handles nested disable/restore correctly.
- Total time with IRQs disabled (including lock-held time): **< 10μs** (see Section 1.8).

### 6.3 No Sleeping While Holding Spinlock

```fajar
// FORBIDDEN: any blocking operation while spinlock is held
@kernel fn bad_io_under_lock(lock: *mut Spinlock) {
    let guard = spinlock_acquire_timeout(lock, 100)?
    let data = uart_read_blocking()  // ERROR: may sleep — deadlock
    sys_ipc_recv(pid)                // ERROR: may block — deadlock
    sleep_ms(10)                     // ERROR: explicit sleep under spinlock
}
```

- While holding a spinlock, the following are **FORBIDDEN**:
  - Any blocking I/O (`uart_read_blocking`, `dma_wait`, etc.)
  - IPC operations (`ipc_send`, `ipc_recv`)
  - Sleep/yield (`sleep_ms`, `yield`)
  - Memory allocation (`kernel_alloc`, `page_alloc`)
  - Any function call that may internally acquire another spinlock
- The only operations permitted under a spinlock are: register reads/writes, memory copies of bounded size, atomic operations, and pure computation.
- If you need to do blocking I/O with mutual exclusion, use message-passing via IPC instead of shared-memory-with-lock.

### 6.4 IPC Over Shared Memory

```fajar
// PREFERRED: IPC message passing for inter-process communication
@safe fn request_sensor_data() -> Result<SensorData, IpcError> {
    let msg = IpcMessage::new(SENSOR_SERVICE_PID, MSG_READ_SENSOR)
    let reply = sys_ipc_call(SENSOR_SERVICE_PID, msg, 1000)?  // 1s timeout
    SensorData::from_bytes(reply.payload)
}

// DISCOURAGED: shared memory (only for large data like camera frames)
// When shared memory IS used, it MUST be:
//   1. Mapped read-only by the consumer
//   2. Protected by an IPC protocol (producer signals "ready", consumer signals "done")
//   3. Bounded in size (declared at creation time, never grown)
```

- Default inter-process data sharing: **IPC message passing** (syscall-based, kernel-mediated, safe).
- Shared memory: allowed ONLY for large data transfers (> 256 bytes) where copying is prohibitive (camera frames, GPU buffers, NPU model data).
- Shared memory regions MUST have explicit ownership: one writer, many readers.
- Shared memory access MUST be coordinated by IPC messages (never poll shared memory in a loop).
- Lock-free ring buffers in shared memory are allowed for high-throughput producer-consumer patterns, but MUST use atomic operations with appropriate memory ordering.

---

## 7. Naming Conventions (OS-Specific)

> Consistent naming prevents bugs. When you see `sys_`, you know it crosses the kernel boundary.
> When you see `irq_`, you know it runs in interrupt context with all its restrictions.

### 7.1 Function Prefixes

| Prefix | Context | Meaning | Example |
|--------|---------|---------|---------|
| `sys_` | `@safe` → kernel | Syscall wrapper | `sys_open()`, `sys_write()`, `sys_mmap()` |
| `irq_` | `@kernel` | IRQ handler / IRQ management | `irq_uart_rx()`, `irq_disable()`, `irq_restore()` |
| `*_init()` | `@kernel` | Driver initialization (suffix) | `uart_init()`, `gic_init()`, `mmu_init()` |
| `*_deinit()` | `@kernel` | Driver teardown (suffix) | `uart_deinit()`, `pcie_deinit()` |
| `dma_` | `@kernel` | DMA operations | `dma_alloc()`, `dma_start()`, `dma_wait()` |
| `page_` | `@kernel` | Page table operations | `page_map()`, `page_unmap()`, `page_alloc()` |
| `sched_` | `@kernel` | Scheduler operations | `sched_yield()`, `sched_add_task()` |
| `ipc_` | `@kernel`/`@safe` | IPC operations | `ipc_send()`, `ipc_recv()`, `ipc_call()` |
| `npu_` | `@device` | NPU operations | `npu_load()`, `npu_infer()`, `npu_unload()` |
| `gpu_` | `@device` | GPU compute operations | `gpu_dispatch()`, `gpu_sync()` |
| `vfs_` | `@safe` | Virtual filesystem | `vfs_open()`, `vfs_read()`, `vfs_stat()` |
| `net_` | `@safe` | Network stack | `net_tcp_connect()`, `net_udp_send()` |
| `cam_` | `@device` | Camera operations | `cam_capture()`, `cam_release_buffer()` |

### 7.2 Register and Hardware Constants

```fajar
// MMIO base addresses: BLOCK_BASE suffix
const TLMM_BASE: u64 = 0x0F10_0000
const QUP_BASE: u64 = 0x0A8C_0000
const GICD_BASE: u64 = 0x1780_0000
const GICR_BASE: u64 = 0x17A0_0000
const EMAC_BASE: u64 = 0x00A0_0000

// Register offsets: REG_ prefix
const REG_GICD_CTLR: u64 = 0x0000
const REG_GICD_TYPER: u64 = 0x0004
const REG_GICD_ISENABLER: u64 = 0x0100
const REG_GICD_IPRIORITYR: u64 = 0x0400
const REG_GICD_ITARGETSR: u64 = 0x0800

// Bit masks: FIELD_MASK suffix or FIELD_BIT suffix
const GICD_CTLR_ENABLE: u32 = 1 << 0
const SCTLR_MMU_BIT: u64 = 1 << 0
const SCTLR_CACHE_BIT: u64 = 1 << 2
const SCTLR_ICACHE_BIT: u64 = 1 << 12

// IRQ numbers: IRQ_ prefix
const IRQ_TIMER: u32 = 27         // aarch64 virtual timer (PPI)
const IRQ_UART5: u32 = 64         // QUP UART5 SPI (placeholder — verify from TRM)
const IRQ_EMAC: u32 = 96          // Ethernet MAC (placeholder — verify from TRM)

// Memory sizes: SIZE_ prefix or _SIZE suffix
const PAGE_SIZE: usize = 4096
const KERNEL_STACK_SIZE: usize = 65536  // 64KB
const USER_STACK_DEFAULT: usize = 1048576  // 1MB
const DMA_POOL_SIZE: usize = 134217728   // 128MB

// Syscall numbers: SYS_ prefix
const SYS_EXIT: u32 = 0x00
const SYS_WRITE: u32 = 0x01
const SYS_READ: u32 = 0x02
const SYS_OPEN: u32 = 0x03
```

### 7.3 Type Naming

```fajar
// Hardware types: descriptive, PascalCase
struct PageTableEntry { ... }
struct GicDistributor { ... }
struct QupUartEngine { ... }
struct DmaDescriptor { ... }
struct IpcMessage { ... }

// Enums: descriptive variants
enum ProcessState { Ready, Running, Blocked, Terminated }
enum PageFlags { Read, Write, Execute, User, Device }
enum DmaDirection { MemToDevice, DeviceToMem, MemToMem }
enum IrqTrigger { Edge, Level, RisingEdge, FallingEdge, BothEdges }
```

### 7.4 Source File Naming

```
fajaros/
  kernel/
    boot.fj            // UEFI boot and early init
    mmu.fj             // MMU and page table management
    exceptions.fj      // Exception vector table and handlers
    scheduler.fj       // Process scheduler
    ipc.fj             // Inter-process communication
    alloc.fj           // Kernel memory allocator (buddy)
    syscall.fj         // Syscall dispatch table
  drivers/
    gic.fj             // GICv3 interrupt controller driver
    tlmm.fj            // TLMM GPIO driver
    qup_uart.fj        // QUP UART driver
    qup_spi.fj         // QUP SPI driver
    qup_i2c.fj         // QUP I2C driver
    pcie.fj            // PCIe controller driver
    nvme.fj            // NVMe block device driver
    emac.fj            // Ethernet MAC driver
    timer.fj           // Architected timer driver
    dma.fj             // DMA engine driver
  services/
    init.fj            // Init process (PID 1)
    vfs.fj             // Virtual filesystem
    fat32.fj           // FAT32 filesystem
    ext4.fj            // ext4 filesystem
    tcpip.fj           // TCP/IP network stack
    display.fj         // Display compositor
    npu_daemon.fj      // NPU inference daemon
    gpu_service.fj     // GPU compute service
  apps/
    fjsh.fj            // FajarOS shell
    repl.fj            // Fajar Lang REPL
```

---

## 8. Error Code System (OS-Specific)

> Every OS error has a unique code. When a user reports "KE112", you know instantly
> it is a DMA pool exhaustion. No ambiguity, no guessing.

### 8.1 Kernel Errors (KE100-KE199)

| Code | Name | Description | Severity |
|------|------|-------------|----------|
| KE100 | `KernelPanic` | Unrecoverable kernel error (triple fault, heap corruption) | Fatal — system halt |
| KE101 | `OutOfKernelMemory` | Buddy allocator exhausted | Critical — OOM killer triggered |
| KE102 | `UnalignedAddress` | Address not aligned to required boundary (page, cache line) | Error — return to caller |
| KE103 | `UnboundedLoop` | Loop in @kernel has no provable upper bound (compile-time) | Compile error |
| KE104 | `RecursionInKernel` | Recursive call detected in @kernel call graph (compile-time) | Compile error |
| KE105 | `LongCriticalSection` | Interrupt-disabled section exceeds 10μs estimate (compile-time) | Warning |
| KE106 | `InvalidPageTableEntry` | PTE has invalid bit combination | Error — page fault handler |
| KE107 | `PageFault` | Access to unmapped or protected page | Error — may be recoverable (demand paging) |
| KE108 | `StackOverflow` | Stack pointer entered guard page | Fatal — process killed |
| KE109 | `InvalidSyscall` | Syscall number not in dispatch table | Error — return ENOSYS |
| KE110 | `IrqTimeout` | Waited for interrupt that never arrived | Error — hardware issue |
| KE111 | `WritableExecutable` | Page mapped with both W and X permissions (W^X violation) | Fatal — system halt |
| KE112 | `DmaPoolExhausted` | DMA buffer pool has no free space | Error — caller must retry |
| KE113 | `LockTimeout` | Spinlock not acquired within timeout | Error — contention issue |
| KE114 | `NestedSpinlock` | Detected nested spinlock acquisition (compile-time or runtime) | Error/compile error |
| KE115 | `HeapCorruption` | Guard word mismatch in kernel allocator | Fatal — system halt |
| KE116 | `BadUserPointer` | User-space pointer passed to kernel is invalid | Error — return EFAULT |
| KE117 | `ProcessLimitExceeded` | Maximum process count reached | Error — return to caller |
| KE118 | `SchedulerError` | Scheduler state machine in invalid state | Critical — restart scheduler |
| KE119 | `ContextSwitchFailed` | Unable to save/restore process context | Fatal — system halt |

### 8.2 Driver Errors (DE100-DE199)

| Code | Name | Description | Severity |
|------|------|-------------|----------|
| DE100 | `DriverInitFailed` | Driver initialization function returned error | Critical — peripheral unavailable |
| DE101 | `HardwareTimeout` | Polling hardware register exceeded timeout | Error — retry or reset |
| DE102 | `TransferTooLarge` | Requested transfer exceeds maximum size | Error — return to caller |
| DE103 | `ZeroLengthTransfer` | Transfer requested with length 0 | Error — return to caller |
| DE104 | `InvalidRegisterValue` | Register read returned unexpected/impossible value | Error — hardware fault |
| DE105 | `DmaError` | DMA transfer failed (bus error, address fault) | Error — retry |
| DE106 | `BusError` | I2C NACK, SPI overrun, or bus arbitration failure | Error — retry or reset |
| DE107 | `FifoOverrun` | Hardware FIFO overflowed (data lost) | Warning — increase IRQ priority |
| DE108 | `FifoUnderrun` | Hardware FIFO underflowed (stale data) | Warning — reduce transfer rate |
| DE109 | `PeripheralNotFound` | Expected peripheral not detected on bus | Error — wrong hardware config |
| DE110 | `ClockConfigFailed` | Unable to set peripheral clock to requested rate | Error — use fallback rate |
| DE111 | `PinConflict` | GPIO pin already configured for different function | Error — reconfigure or use alt pin |
| DE112 | `PhyLinkDown` | Ethernet PHY has no link (cable unplugged?) | Warning — retry periodically |
| DE113 | `NvmeError` | NVMe command failed (status from completion queue) | Error — includes NVMe status |
| DE114 | `PcieEnumFailed` | PCIe bus enumeration found no devices | Error — check physical connection |
| DE115 | `SdCardNotPresent` | SD card slot is empty | Error — check card insertion |
| DE116 | `InterruptNotAcked` | IRQ handler returned without acknowledging interrupt | Error — storm risk |
| DE117 | `RegisterReadback` | Register readback does not match written value | Error — hardware fault |
| DE118 | `CacheCoherencyError` | DMA data inconsistent with cache (missing flush/invalidate) | Error — data corruption |
| DE119 | `PowerDomainError` | Peripheral power domain not enabled | Error — enable power first |

### 8.3 Syscall Errors (SE100-SE199)

| Code | Name | Description | Severity |
|------|------|-------------|----------|
| SE100 | `SyscallFailed` | Generic syscall failure (fallback code) | Error |
| SE101 | `BadFd` | File descriptor is invalid or closed | Error |
| SE102 | `WouldBlock` | Non-blocking operation would block | Transient — retry |
| SE103 | `Interrupted` | Syscall interrupted by signal | Transient — retry |
| SE104 | `NoSpace` | Filesystem or buffer has no free space | Error |
| SE105 | `ShortWrite` | Fewer bytes written than requested | Warning — retry remaining |
| SE106 | `TooManyOpenFiles` | Process fd table is full (256 max) | Error — close unused fds |
| SE107 | `PermissionDenied` | Process lacks permission for operation | Error — check credentials |
| SE108 | `NotFound` | File, process, or resource not found | Error |
| SE109 | `AlreadyExists` | File or resource already exists | Error |
| SE110 | `InvalidArgument` | Syscall argument out of valid range | Error |
| SE111 | `OutOfMemory` | Process memory quota exceeded | Error — reduce usage |
| SE112 | `IpcQueueFull` | Destination process IPC queue is full | Transient — retry |
| SE113 | `IpcTimeout` | IPC recv/call timed out | Error — destination unresponsive |
| SE114 | `IpcPeerDied` | IPC destination process terminated | Error — restart peer |
| SE115 | `NetConnectionRefused` | TCP connection refused by remote | Error |
| SE116 | `NetTimeout` | Network operation timed out | Error — check connectivity |
| SE117 | `NetUnreachable` | Network destination unreachable | Error — check routing |
| SE118 | `MessageTooLarge` | IPC or network message exceeds size limit | Error |
| SE119 | `ProcessNotFound` | Target PID does not exist | Error |

### 8.4 NPU Errors (NE100-NE119)

| Code | Name | Description | Severity |
|------|------|-------------|----------|
| NE100 | `NpuNotAvailable` | Hexagon 770 DSP not detected or firmware not loaded | Error — use CPU fallback |
| NE101 | `ModelNotFound` | Model file path does not exist | Error |
| NE102 | `ModelInvalid` | Model binary is corrupt or incompatible format | Error |
| NE103 | `FirmwareNotLoaded` | Hexagon DSP firmware (skel) not loaded | Error — load firmware first |
| NE104 | `NpuOutOfMemory` | NPU shared memory exhausted | Error — unload unused models |
| NE105 | `InferenceFailed` | NPU inference execution returned error | Error — retry or CPU fallback |
| NE106 | `InputShapeMismatch` | Tensor shape does not match model input specification | Error — reshape input |
| NE107 | `OutputShapeMismatch` | Model produced unexpected output shape | Error — model version issue |
| NE108 | `QuantizationError` | f64→INT8 quantization produced out-of-range values | Warning — clamp applied |
| NE109 | `DequantizationError` | INT8→f64 dequantization scale/offset invalid | Error — recalibrate model |
| NE110 | `FastRpcError` | FastRPC communication with CDSP failed | Error — DSP may have crashed |
| NE111 | `ModelLoadTimeout` | Model loading took longer than 30 seconds | Error — model too large |
| NE112 | `ConcurrentModelLimit` | Maximum concurrent NPU models exceeded | Error — unload a model first |
| NE113 | `PowerThrottled` | NPU performance reduced due to thermal throttling | Warning — reduce workload |
| NE114 | `QnnSdkError` | Underlying QNN SDK returned error | Error — includes QNN code |

### 8.5 GPU Errors (GE100-GE119)

| Code | Name | Description | Severity |
|------|------|-------------|----------|
| GE100 | `GpuNotAvailable` | Adreno 643 not detected or driver not loaded | Error — use CPU fallback |
| GE101 | `DispatchFailed` | GPU kernel dispatch failed | Error — check shader |
| GE102 | `GpuTimeout` | GPU operation timed out (5s default) | Error — GPU may be hung |
| GE103 | `InvalidOutputBuffer` | GPU output buffer is invalid or unmapped | Error |
| GE104 | `ShaderCompileFailed` | Compute shader compilation failed | Error — fix shader source |
| GE105 | `BufferAllocationFailed` | GPU buffer allocation failed (VRAM exhausted) | Error — free buffers |
| GE106 | `InvalidBufferSize` | Buffer size exceeds GPU limits | Error — reduce size |
| GE107 | `GpuReset` | GPU was reset due to hang (TDR) | Warning — resubmit work |
| GE108 | `VulkanError` | Vulkan API returned error | Error — includes VkResult |
| GE109 | `OpenClError` | OpenCL API returned error | Error — includes cl_int |
| GE110 | `GpuThermalThrottle` | GPU frequency reduced due to temperature | Warning — reduce workload |
| GE111 | `SyncFailed` | GPU fence/sync wait failed | Error — possible GPU crash |

### 8.6 Error Code Format

```
Format: [PREFIX][NUMBER]  (same as Fajar Lang core errors)

Ranges:
  KE001-KE004   Original kernel context errors (from V1_RULES)
  KE100-KE199   FajarOS kernel errors
  DE001-DE003   Original device context errors (from V1_RULES)
  DE100-DE199   FajarOS driver errors
  SE001-SE012   Original semantic errors (from V1_RULES)
  SE100-SE199   FajarOS syscall errors
  NE100-NE119   FajarOS NPU errors
  GE100-GE119   FajarOS GPU errors
```

- Error codes in the 100+ range are FajarOS-specific; codes below 100 are Fajar Lang compiler errors.
- Every error MUST include:
  - Error code (e.g., KE112)
  - Human-readable message
  - Context data (address, register value, process ID, etc.)
  - Source location (file, line) for compile-time errors
- Error display uses `miette` formatting for kernel serial output where available, raw hex dump otherwise.

---

## 9. Testing Rules (OS-Specific)

### 9.1 Test Environments

| Environment | When | How | What |
|-------------|------|-----|------|
| Host unit tests | Every commit | `fj test` on x86_64 | Pure logic: data structures, algorithms, parsers |
| QEMU integration | Every sprint | `fj test --qemu aarch64` | Boot, MMU, exceptions, scheduler, IPC, UART |
| Hardware smoke | Every phase gate | Manual on Dragon Q6A | GPIO, SPI, I2C, NPU, GPU, Ethernet |
| Hardware stress | Before release | 24-hour run on Q6A | Memory leaks, lock contention, thermal, stability |

### 9.2 QEMU Test Requirements

- Every kernel function MUST have a QEMU test that verifies correct behavior on `qemu-system-aarch64 -M virt -cpu cortex-a76`.
- QEMU tests boot FajarOS, run the test, and verify serial output — fully automated, no human interaction.
- QEMU test timeout: **30 seconds** per test. If QEMU does not produce expected output in 30s, the test fails.
- QEMU memory: `-m 256M` minimum for kernel tests, `-m 1G` for NPU/GPU simulation tests.

### 9.3 Driver Test Pattern

```fajar
// Every driver has three test levels:

// 1. Unit test (no hardware, mocked registers)
@test fn test_uart_baud_rate_calculation() {
    let divisor = calculate_baud_divisor(115200, 19200000)
    assert_eq(divisor, 10)
}

// 2. QEMU test (virtual hardware)
@test @qemu fn test_uart_echo_qemu() {
    uart_init(PL011_BASE, 115200)?
    uart_write_byte('A')
    // verify serial output contains 'A'
}

// 3. Hardware test (real Q6A board, manual run)
@test @hardware fn test_uart_echo_q6a() {
    uart_init(QUP_UART5_BASE, 115200)?
    uart_write_byte('A')
    let echo = uart_read_byte_timeout(1000)?
    assert_eq(echo, 'A')
}
```

### 9.4 Kernel Invariant Tests

Every sprint gate MUST verify these invariants:

1. No memory leaks: total allocated == total freed (after full test suite)
2. No lock contention: all spinlocks released before test ends
3. No orphan processes: all spawned processes terminated or cleaned up
4. No IRQ storms: interrupt count is within expected bounds
5. W^X enforced: no page has both write and execute permission
6. Guard pages intact: no stack overflow went undetected
7. DMA pool intact: all DMA buffers freed after use

---

## 10. Invariants (Must Always Be True)

> These invariants are the contract between FajarOS and the hardware. If any invariant
> is violated, the system is in an undefined state and MUST halt.

### 10.1 Kernel Invariants

1. **W^X:** No page table entry has both AP=01 (write) and PXN=0 (execute) simultaneously.
2. **Guard pages:** Every stack (kernel and user) has a guard page at its lowest address, mapped with PTE invalid.
3. **IRQ latency:** Time from IRQ assertion to handler entry is < 50μs under all conditions.
4. **Spinlock timeout:** No spinlock is held for more than 100μs.
5. **Critical section:** No interrupt-disabled section exceeds 10μs.
6. **Heap integrity:** All kernel allocations have valid guard words (0xDEAD_BEEF).
7. **No recursion:** Kernel call graph has no cycles.
8. **Bounded loops:** Every loop in @kernel has a provable upper bound.
9. **DMA coherency:** Every DMA transfer has correct cache flush/invalidate bracketing.
10. **User pointer validation:** Every user-space pointer passed to kernel is validated via `copy_from_user()` / `copy_to_user()` before dereference.

### 10.2 Driver Invariants

11. **Volatile access:** Every hardware register access uses `volatile_read()` or `volatile_write()`.
12. **MMIO mapping:** All device memory is mapped as Device-nGnRnE or Device-nGnRE (never Normal/Cacheable).
13. **IRQ acknowledgment:** Every IRQ handler writes EOI to GICv3 before returning.
14. **Timeout on hardware:** No hardware poll loop runs without a timeout.
15. **Error return:** No driver function uses `panic()`, `assert()`, or `unwrap()`.

### 10.3 Service Invariants

16. **Fd cleanup:** Every opened file descriptor is closed (via Drop) before process exits.
17. **IPC bounded:** No IPC message payload exceeds 256 bytes.
18. **Network bounded:** No network read allocates more than 64KB.
19. **Syscall result:** Every syscall return value is checked — no ignored `Result`.

### 10.4 AI Invariants

20. **No raw pointers:** @device code contains zero raw pointer operations (compiler-enforced).
21. **GPU sync:** Every `gpu_dispatch()` has a matching `gpu_sync()` before reading output.
22. **NPU error handling:** Every `npu_load()` and `npu_infer()` call handles the `Result`.
23. **Camera zero-copy:** No `memcpy` of camera DMA buffers — tensor views only.

### 10.5 Invariant Enforcement

| Invariant | Enforcement Mechanism |
|-----------|----------------------|
| 1 (W^X) | `page_map()` rejects invalid flag combinations at call site |
| 2 (Guard pages) | `create_process_stack()` always allocates guard page |
| 3-5 (Timing) | Static analysis + runtime monitoring in debug builds |
| 6 (Heap integrity) | `kernel_free()` checks guard words, halts on mismatch |
| 7-8 (No recursion, bounded loops) | Compiler static analysis (KE103, KE104) |
| 9 (DMA coherency) | `DmaBuffer` API forces correct cache management |
| 10 (User pointers) | `copy_from_user()` / `copy_to_user()` validate before access |
| 11-15 (Drivers) | Compiler context checks + code review |
| 16-19 (Services) | RAII Drop, type system, bounded read APIs |
| 20-23 (AI) | Compiler context checks (DE001), type system |

---

## 11. Code Review Checklist (FajarOS)

Before marking any FajarOS task as DONE, verify ALL of the following:

### Kernel / Driver Code (`@kernel`)

- [ ] No heap allocation (no String, Vec, Box — use kernel_alloc or stack arrays)
- [ ] No panic, assert, or unwrap — all functions return Result
- [ ] All loops are bounded (explicit max iteration count)
- [ ] No recursion in call graph
- [ ] Every `asm!()` has SAFETY comment (what, why, failure mode)
- [ ] Every `volatile_read/write` operates on correctly-typed memory (Device-nGnRnE)
- [ ] Every multi-register MMIO sequence has correct memory barriers (dsb/dmb)
- [ ] IRQ handlers are reentrant-safe (no locks, no allocation, no blocking)
- [ ] Interrupt-disabled sections < 10μs
- [ ] All hardware waits have timeouts
- [ ] All MMIO base addresses are const
- [ ] Register bitfields documented with [MSB:LSB] notation
- [ ] DMA buffers have correct cache flush/invalidate
- [ ] W^X: no page mapped writable+executable
- [ ] Spinlocks use try_lock with timeout
- [ ] No sleeping/blocking while holding spinlock
- [ ] Test passes on QEMU

### Service Code (`@safe`)

- [ ] All syscalls check return Result
- [ ] IPC payloads <= 256 bytes
- [ ] File descriptors closed via RAII (Drop)
- [ ] Network reads bounded
- [ ] No unbounded allocation

### AI Code (`@device`)

- [ ] No raw pointers (compiler enforces DE001)
- [ ] Tensor shapes declared and checked
- [ ] NPU load/infer errors handled
- [ ] GPU dispatch has matching sync
- [ ] Camera frames: zero-copy only

### General (All Code)

- [ ] Correct naming convention (sys_*, irq_*, REG_*, *_BASE, *_init)
- [ ] Error codes from correct range (KE100+, DE100+, SE100+, NE100+, GE100+)
- [ ] Doc comments on all public functions
- [ ] At least 1 test per function
- [ ] No regressions in existing tests

---

## 12. Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     FajarOS "Surya" — Rules at a Glance                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  @kernel: No heap. No panic. No recursion. No unbounded loops.             │
│           All asm has SAFETY. All I/O is volatile. IRQ < 10μs.             │
│                                                                             │
│  @device: No raw pointers. Shapes checked. NPU/GPU errors handled.        │
│           Camera frames zero-copy. GPU sync before read.                    │
│                                                                             │
│  @safe:   Syscalls return Result. IPC <= 256B. Fds via RAII.               │
│           Network bounded. No unbounded alloc.                              │
│                                                                             │
│  Memory:  W^X enforced. Guard pages on all stacks. Buddy + poison.         │
│           User/kernel separate. DMA mapped uncacheable.                     │
│                                                                             │
│  Locks:   Spinlock try_lock + 100μs timeout. Never nest. No sleep.         │
│           IRQ-safe locks disable IRQ first. IPC over shared memory.         │
│                                                                             │
│  Names:   sys_* irq_* *_init REG_* *_BASE SYS_* IRQ_* PAGE_SIZE           │
│                                                                             │
│  Errors:  KE100-199 (kernel) DE100-199 (driver) SE100-199 (syscall)        │
│           NE100-119 (NPU) GE100-119 (GPU)                                  │
│                                                                             │
│  Test:    QEMU first → hardware second. Timeout 30s. Verify invariants.    │
│                                                                             │
│  Priority: CORRECTNESS > SAFETY > DETERMINISM > PERFORMANCE                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*V30_RULES.md v1.0 — FajarOS "Surya" OS-Specific Coding Rules*
*Target: Radxa Dragon Q6A (Qualcomm QCS6490)*
*Non-negotiable unless discussed explicitly with project lead*
