# V3.0 "Surya" — Technical Skills & Implementation Patterns

> Reference patterns for FajarOS kernel, drivers, and AI subsystem.
> Read this BEFORE implementing any Phase 1-10 task.

---

## 1. aarch64 Bare-Metal Boot

### UEFI Handoff

The Dragon Q6A boots via UEFI. FajarOS is loaded as an EFI application.

```fajar
// EFI entry point — called by UEFI firmware
@kernel fn efi_main(image_handle: u64, system_table: *const u8) -> u64 {
    // Save system table pointer for boot services
    let boot_services = efi_get_boot_services(system_table)

    // Get memory map (needed for ExitBootServices)
    let mmap_size: u64 = 0
    let mmap_key: u64 = 0
    let desc_size: u64 = 0
    efi_get_memory_map(boot_services, &mut mmap_size, &mut mmap_key, &mut desc_size)

    // Exit boot services — UEFI hands full control to us
    // After this: no UEFI runtime, we own all hardware
    efi_exit_boot_services(image_handle, mmap_key)

    // Jump to kernel
    kernel_main()

    0  // EFI_SUCCESS
}
```

### Early Init Sequence

```fajar
@kernel fn kernel_main() {
    // 1. Disable all interrupts
    asm!("msr daifset, #0xf")

    // 2. Ensure we're at EL1 (UEFI may boot us at EL2)
    let current_el: u64 = 0
    asm!("mrs {}, CurrentEL", out(reg) current_el)
    let el = (current_el >> 2) & 0x3
    if el == 2 {
        drop_to_el1()  // HCR_EL2.RW=1, SPSR_EL2, ELR_EL2, eret
    }

    // 3. Set stack pointer (top of kernel stack region)
    asm!("ldr x0, ={}", "mov sp, x0", const KERNEL_STACK_TOP)

    // 4. Zero BSS section
    zero_bss()

    // 5. Initialize MMU
    mmu_init()

    // 6. Set exception vectors
    asm!("adr x0, exception_vectors", "msr vbar_el1, x0")

    // 7. Initialize GICv3
    gic_init()

    // 8. Initialize kernel heap
    heap_init(KERNEL_HEAP_BASE, KERNEL_HEAP_SIZE)

    // 9. Enable interrupts
    asm!("msr daifclr, #0xf")

    // 10. Start init process
    scheduler_init()
    spawn(init_main)
    schedule()  // never returns
}
```

### EL2 → EL1 Drop

```fajar
@kernel fn drop_to_el1() {
    // Enable aarch64 at EL1
    let hcr: u64 = (1 << 31)  // RW bit = aarch64
    asm!("msr hcr_el2, {}", in(reg) hcr)

    // Set return state: EL1h, all interrupts masked
    let spsr: u64 = 0x3c5  // M=EL1h, DAIF=1111
    asm!("msr spsr_el2, {}", in(reg) spsr)

    // Set return address to after eret
    asm!("adr x0, 1f", "msr elr_el2, x0", "eret", "1:")
}
```

---

## 2. aarch64 MMU Setup

### Page Table Format (4KB Granule)

```
Level 0: 512 entries, each covers 512GB (bits [47:39])
Level 1: 512 entries, each covers 1GB   (bits [38:30])
Level 2: 512 entries, each covers 2MB   (bits [29:21])
Level 3: 512 entries, each covers 4KB   (bits [20:12])

Page Table Entry (64 bits):
┌──────────────────────────────────────────────────────────────────┐
│ 63:59 │ 58:55 │ 54:53 │ 52   │ 51:48 │ 47:12        │ 11:2  │ 1 │ 0 │
│ rsvd  │ SW    │ XN,PXN│ cont │ rsvd  │ output addr  │ attrs │ T │ V │
└──────────────────────────────────────────────────────────────────┘

V (bit 0): Valid
T (bit 1): Table (1) or Block (0) — only for L0-L2
AP (bits 7:6): Access Permission
  00 = EL1 R/W, EL0 none
  01 = EL1 R/W, EL0 R/W
  10 = EL1 RO, EL0 none
  11 = EL1 RO, EL0 RO
SH (bits 9:8): Shareability
  00 = Non-shareable
  10 = Outer shareable
  11 = Inner shareable
AF (bit 10): Access Flag (must set to 1)
AttrIndx (bits 4:2): Index into MAIR_EL1
```

### MAIR Configuration

```fajar
@kernel fn mmu_configure_mair() {
    // Attr0: Normal memory, Write-Back, Write-Allocate
    // Attr1: Device memory, nGnRnE (strongly ordered)
    // Attr2: Normal memory, Non-cacheable
    let mair: u64 = 0x00_44_00_FF
    //                    ││  ││  ││
    //                    ││  ││  └└─ Attr0: 0xFF = Normal WB-WA
    //                    ││  └└──── Attr1: 0x00 = Device nGnRnE
    //                    └└──────── Attr2: 0x44 = Normal NC
    asm!("msr mair_el1, {}", in(reg) mair)
}
```

### TCR Configuration

```fajar
@kernel fn mmu_configure_tcr() {
    let tcr: u64 = 0
        | (16 << 0)    // T0SZ = 16 → 48-bit VA (256TB)
        | (0b00 << 6)  // — reserved
        | (0b00 << 8)  // IRGN0 = Normal WB-WA
        | (0b00 << 10) // ORGN0 = Normal WB-WA
        | (0b11 << 12) // SH0 = Inner Shareable
        | (0b00 << 14) // TG0 = 4KB granule
        | (1 << 23)    // EPD1 = disable TTBR1 walks
    asm!("msr tcr_el1, {}", in(reg) tcr)
}
```

### Enable MMU

```fajar
@kernel fn mmu_enable(ttbr0: u64) {
    // Set translation table base
    asm!("msr ttbr0_el1, {}", in(reg) ttbr0)

    // Invalidate all TLBs
    asm!("tlbi alle1")
    asm!("dsb ish")
    asm!("isb")

    // Enable MMU + caches
    let mut sctlr: u64 = 0
    asm!("mrs {}, sctlr_el1", out(reg) sctlr)
    sctlr = sctlr | (1 << 0)   // M = MMU enable
    sctlr = sctlr | (1 << 2)   // C = Data cache enable
    sctlr = sctlr | (1 << 12)  // I = Instruction cache enable
    sctlr = sctlr | (1 << 19)  // WXN = Write implies XN
    asm!("msr sctlr_el1, {}", in(reg) sctlr)
    asm!("isb")
}
```

### Page Map Implementation

```fajar
@kernel fn page_map(l0_table: *mut u64, vaddr: u64, paddr: u64,
                      size: u64, attrs: u64) {
    let mut offset: u64 = 0
    while offset < size {
        let va = vaddr + offset
        let pa = paddr + offset

        let l0_idx = (va >> 39) & 0x1FF
        let l1_idx = (va >> 30) & 0x1FF
        let l2_idx = (va >> 21) & 0x1FF
        let l3_idx = (va >> 12) & 0x1FF

        // Walk/create page tables at each level
        let l1_table = get_or_create_table(l0_table, l0_idx)
        let l2_table = get_or_create_table(l1_table, l1_idx)
        let l3_table = get_or_create_table(l2_table, l2_idx)

        // Set L3 page entry
        let entry = pa | attrs | (1 << 10) | 0x3  // AF=1, valid+page
        volatile_write(l3_table as u64 + l3_idx * 8, entry)

        offset = offset + 4096  // 4KB pages
    }
}
```

---

## 3. GICv3 Programming

### Register Map (QCS6490)

```
GICD (Distributor):       0x1780_0000
  GICD_CTLR              +0x0000  Control register
  GICD_TYPER             +0x0004  Interrupt count
  GICD_ISENABLER[n]      +0x0100  Set-enable (32 IRQs per register)
  GICD_ICENABLER[n]      +0x0180  Clear-enable
  GICD_IPRIORITYR[n]     +0x0400  Priority (8 bits per IRQ)
  GICD_ITARGETSR[n]      +0x0800  Target CPU (GICv2 compat)
  GICD_ICFGR[n]          +0x0C00  Edge/level config

GICR (Redistributor):    0x17A0_0000  (per-CPU, stride 0x20000)
  GICR_WAKER             +0x0014  Wake control
  GICR_ISENABLER0        +0x10100  SGI/PPI set-enable
  GICR_ICENABLER0        +0x10180  SGI/PPI clear-enable
  GICR_IPRIORITYR[n]     +0x10400  SGI/PPI priority

ICC System Registers (accessed via MSR/MRS):
  ICC_PMR_EL1            Priority Mask (set to 0xFF = all)
  ICC_IAR1_EL1           Interrupt Acknowledge (read = ack + get ID)
  ICC_EOIR1_EL1          End of Interrupt (write IRQ ID)
  ICC_SRE_EL1            System Register Enable
  ICC_IGRPEN1_EL1        Group 1 interrupt enable
```

### IRQ Numbers (QCS6490)

```
IRQ  Type   Source
───  ─────  ──────────────────────────
27   PPI    Virtual timer (EL1)
30   PPI    Physical timer (EL1)
32+  SPI    UART0 (QUP SE)
36+  SPI    UART5 (QUP SE)
48+  SPI    GPIO group interrupts
64+  SPI    PCIe MSI
80+  SPI    USB DWC3
96+  SPI    Ethernet EMAC
112+ SPI    Camera CSID
128+ SPI    DMA engine
```

### GIC Init Pattern

```fajar
const GICD_BASE: u64 = 0x1780_0000
const GICR_BASE: u64 = 0x17A0_0000

@kernel fn gic_init() {
    // 1. Disable distributor
    volatile_write(GICD_BASE + 0x0000, 0)

    // 2. Configure all SPIs: group 1, level-triggered, priority 0xA0
    let typer = volatile_read(GICD_BASE + 0x0004)
    let irq_count = ((typer & 0x1F) + 1) * 32

    let mut i: u64 = 32  // skip SGI/PPI (0-31)
    while i < irq_count {
        let reg = i / 32
        let byte = i / 4

        // Set group 1 (non-secure)
        volatile_write(GICD_BASE + 0x0080 + reg * 4, 0xFFFFFFFF)

        // Set priority 0xA0 (medium)
        volatile_write(GICD_BASE + 0x0400 + byte, 0xA0)

        i = i + 1
    }

    // 3. Enable distributor (ARE=1, group 1)
    volatile_write(GICD_BASE + 0x0000, 0x37)

    // 4. Wake redistributor
    let waker = volatile_read(GICR_BASE + 0x0014)
    volatile_write(GICR_BASE + 0x0014, waker & !(1 << 1))  // Clear ProcessorSleep
    // Wait for ChildrenAsleep to clear
    while volatile_read(GICR_BASE + 0x0014) & (1 << 2) != 0 {}

    // 5. Enable CPU interface via system registers
    asm!("msr ICC_SRE_EL1, {}", in(reg) 0x7_u64)     // SRE=1, DFB=1, DIB=1
    asm!("isb")
    asm!("msr ICC_PMR_EL1, {}", in(reg) 0xFF_u64)     // Priority mask: all
    asm!("msr ICC_IGRPEN1_EL1, {}", in(reg) 0x1_u64)  // Enable group 1
    asm!("isb")
}

@kernel fn gic_enable_irq(irq: u32, priority: u8) {
    if irq >= 32 {
        // SPI: configure in GICD
        let reg = irq / 32
        let bit = irq % 32
        volatile_write(GICD_BASE + 0x0100 + (reg as u64) * 4, 1 << bit)
        volatile_write(GICD_BASE + 0x0400 + (irq as u64), priority as u64)
    } else {
        // SGI/PPI: configure in GICR
        let bit = irq % 32
        volatile_write(GICR_BASE + 0x10100, 1 << bit)
        volatile_write(GICR_BASE + 0x10400 + (irq as u64), priority as u64)
    }
}

@kernel fn gic_ack() -> u32 {
    let iar: u64 = 0
    asm!("mrs {}, ICC_IAR1_EL1", out(reg) iar)
    iar as u32
}

@kernel fn gic_eoi(irq: u32) {
    asm!("msr ICC_EOIR1_EL1, {}", in(reg) irq as u64)
}
```

---

## 4. QCS6490 Peripheral Programming

### TLMM (GPIO)

```
TLMM base: 0x0F10_0000
Per-pin stride: 0x1000

GPIO_CFG (offset 0x00):
  [1:0]   GPIO_PULL (0=none, 1=down, 3=up)
  [5:2]   FUNC_SEL (0=GPIO, 1-15=alt function)
  [8:6]   DRV_STRENGTH (0=2mA, 7=16mA)
  [9]     GPIO_OE (0=input, 1=output)

GPIO_IN_OUT (offset 0x04):
  [0]     GPIO_IN (read: current pin state)
  [1]     GPIO_OUT (write: set output value)

GPIO_INTR_CFG (offset 0x08):
  [0]     INTR_ENABLE
  [1]     INTR_POL (0=active-low, 1=active-high)
  [2]     INTR_DET (0=level, 1=edge)
  [3]     INTR_RAW_STATUS_EN
```

```fajar
const TLMM_BASE: u64 = 0x0F10_0000

@kernel fn gpio_configure(pin: u32, func: u32, direction: bool, pull: u32) {
    let base = TLMM_BASE + (pin as u64) * 0x1000
    let cfg = (pull & 0x3)
        | ((func & 0xF) << 2)
        | (3 << 6)  // 8mA drive strength
        | (if direction { 1 << 9 } else { 0 })
    volatile_write(base + 0x00, cfg as u64)
    dsb()
}

@kernel fn gpio_set(pin: u32, high: bool) {
    let base = TLMM_BASE + (pin as u64) * 0x1000
    volatile_write(base + 0x04, if high { 2 } else { 0 })  // bit[1] = OUT
    dsb()
}

@kernel fn gpio_get(pin: u32) -> bool {
    let base = TLMM_BASE + (pin as u64) * 0x1000
    volatile_read(base + 0x04) & 1 != 0  // bit[0] = IN
}
```

### QUP (UART Mode)

```
QUP base varies by engine (SE = Serial Engine):
  QUP0_SE0: 0x0980_0000 (UART0)
  QUP0_SE4: 0x0984_0000 (UART5)  ← 40-pin header pins 8/10
  QUP1_SE0: 0x0A80_0000 (UART6)

Key registers (offset from SE base):
  SE_GENI_M_CMD0           +0x000  Main sequencer command
  SE_GENI_S_CMD0           +0x004  Secondary sequencer command
  SE_GENI_TX_FIFOn         +0x700  TX FIFO (write bytes here)
  SE_GENI_RX_FIFOn         +0x780  RX FIFO (read bytes here)
  SE_GENI_TX_FIFO_STATUS   +0x800  TX FIFO level
  SE_GENI_RX_FIFO_STATUS   +0x804  RX FIFO level
  SE_GENI_M_IRQ_STATUS     +0x010  Main IRQ status
  SE_GENI_M_IRQ_CLEAR      +0x018  Clear main IRQ
  SE_GENI_CFG_REG80        +0x240  Clock divider
  SE_GENI_SER_M_CLK_CFG    +0x048  Serial clock config
```

```fajar
const UART5_BASE: u64 = 0x0984_0000

@kernel fn uart_init(base: u64, baud: u32) {
    // Calculate clock divider for desired baud rate
    // QUP clock is typically 19.2 MHz
    let div = 19_200_000 / (baud * 16)

    // Set clock divider
    volatile_write(base + 0x240, div as u64)

    // Configure: 8 data bits, no parity, 1 stop bit
    volatile_write(base + 0x048, 0x21)  // UART mode, enable

    dsb()
}

@kernel fn uart_tx_byte(base: u64, byte: u8) {
    // Wait for TX FIFO not full
    while volatile_read(base + 0x800) & 0xF != 0 {}

    // Write byte to TX FIFO
    volatile_write(base + 0x700, byte as u64)
}

@kernel fn uart_rx_byte(base: u64) -> u8 {
    // Wait for RX FIFO not empty
    while volatile_read(base + 0x804) & 0x1F == 0 {}

    // Read byte from RX FIFO
    (volatile_read(base + 0x780) & 0xFF) as u8
}
```

### QUP (I2C Mode)

```fajar
@kernel fn i2c_write(base: u64, addr: u8, data: *const u8, len: u32) -> bool {
    // Set slave address
    volatile_write(base + 0x100, (addr as u64) << 1)

    // Set transfer length
    volatile_write(base + 0x104, len as u64)

    // Start write command
    volatile_write(base + 0x000, 0x08)  // I2C_WRITE command

    // Write data bytes to TX FIFO
    let mut i: u32 = 0
    while i < len {
        while volatile_read(base + 0x800) & 0xF == 0xF {}  // Wait FIFO not full
        volatile_write(base + 0x700, volatile_read(data as u64 + i as u64))
        i = i + 1
    }

    // Wait for completion
    let timeout: u32 = 100_000
    let mut t: u32 = 0
    while t < timeout {
        let status = volatile_read(base + 0x010)
        if status & (1 << 0) != 0 {  // M_CMD_DONE
            volatile_write(base + 0x018, status)  // Clear IRQ
            return true
        }
        if status & (1 << 4) != 0 {  // NACK
            volatile_write(base + 0x018, status)
            return false
        }
        t = t + 1
    }
    false  // Timeout
}
```

### ARM Architected Timer

```fajar
@kernel fn timer_init() {
    // Read timer frequency
    let freq: u64 = 0
    asm!("mrs {}, cntfrq_el0", out(reg) freq)
    // QCS6490: typically 19.2 MHz

    // Enable virtual timer, unmask IRQ
    asm!("msr cntv_ctl_el0, {}", in(reg) 1_u64)  // ENABLE=1, IMASK=0
}

@kernel fn timer_get_ticks() -> u64 {
    let ticks: u64 = 0
    asm!("mrs {}, cntvct_el0", out(reg) ticks)
    ticks
}

@kernel fn timer_set_deadline(ticks: u64) {
    asm!("msr cntv_cval_el0, {}", in(reg) ticks)
}

@kernel fn sleep_us(us: u64) {
    let freq: u64 = 0
    asm!("mrs {}, cntfrq_el0", out(reg) freq)
    let start = timer_get_ticks()
    let target = start + (freq * us / 1_000_000)
    while timer_get_ticks() < target {}
}
```

---

## 5. PCIe / NVMe

### PCIe Configuration Space

```
PCIe controller base: 0x0100_0000 (QCS6490)
Config space access: ECAM (Enhanced Configuration Access Mechanism)

Config header (Type 0):
  Offset  Size  Field
  0x00    2     Vendor ID
  0x02    2     Device ID
  0x04    2     Command (bit 1=Memory, bit 2=BusMaster)
  0x06    2     Status
  0x08    1     Revision ID
  0x09    3     Class Code (0x010802 = NVMe)
  0x10    4     BAR0 (NVMe registers base, 64-bit)
  0x14    4     BAR1 (BAR0 upper 32 bits)
```

### NVMe Protocol

```
Admin Queue (ASQ/ACQ):
  Submission Entry: 64 bytes (CDW0-CDW15)
  Completion Entry: 16 bytes (DW0-DW3)
  Doorbell: write to BAR0 + 0x1000 + (QID * 2 * stride)

Key Admin Commands:
  Opcode 0x06: Identify Controller → 4KB controller info
  Opcode 0x06: Identify Namespace → 4KB namespace info (CNS=0)
  Opcode 0x01: Create I/O Submission Queue
  Opcode 0x05: Create I/O Completion Queue

I/O Commands:
  Opcode 0x02: Read  (SLBA, NLB, PRP1, PRP2)
  Opcode 0x01: Write (SLBA, NLB, PRP1, PRP2)

PRP (Physical Region Page):
  PRP1: first page physical address
  PRP2: second page or PRP list for >8KB transfers
```

```fajar
@kernel fn nvme_read_blocks(nvme_base: u64, lba: u64, count: u16,
                             buffer: u64) -> bool {
    // Build submission entry
    let sqe_base = nvme_sq_tail_ptr(nvme_base, 1)  // I/O queue 1

    volatile_write(sqe_base + 0x00, 0x02)         // CDW0: opcode = Read
    volatile_write(sqe_base + 0x18, buffer)        // PRP1
    volatile_write(sqe_base + 0x20, buffer + 4096) // PRP2 (if >4KB)
    volatile_write(sqe_base + 0x28, lba)           // CDW10-11: SLBA
    volatile_write(sqe_base + 0x30, (count - 1) as u64)  // CDW12: NLB (0-based)

    // Ring doorbell
    nvme_ring_sq_doorbell(nvme_base, 1)

    // Wait for completion
    nvme_wait_completion(nvme_base, 1, 100_000)
}
```

---

## 6. Network Stack Patterns

### Ethernet Frame

```
┌──────┬──────┬──────┬──────────────┬─────┐
│ DST  │ SRC  │ Type │ Payload      │ FCS │
│ 6B   │ 6B   │ 2B   │ 46-1500B     │ 4B  │
└──────┴──────┴──────┴──────────────┴─────┘
EtherType: 0x0800=IPv4, 0x0806=ARP, 0x86DD=IPv6
```

### TCP State Machine

```
                    ┌──────────┐
        ┌──────────►│  CLOSED  │◄──────────┐
        │           └────┬─────┘           │
        │ close()        │ connect()       │ close()
        │                ▼                 │
   ┌────┴────┐     ┌──────────┐      ┌────┴────┐
   │LAST_ACK │     │ SYN_SENT │      │TIME_WAIT│
   └────▲────┘     └────┬─────┘      └────▲────┘
        │                │ SYN+ACK        │
   FIN  │                ▼                │ FIN
   ┌────┴────┐     ┌──────────┐      ┌────┴────┐
   │CLOSE_WAIT│    │ESTABLISHED│─────►│FIN_WAIT │
   └─────────┘     └──────────┘      └─────────┘
                   send()/recv()
```

### DHCP Sequence

```fajar
@safe fn dhcp_discover(socket: i32, mac: [u8; 6]) {
    // 1. DISCOVER: broadcast to 255.255.255.255:67
    let msg = dhcp_build_discover(mac, random_xid())
    udp_send(socket, BROADCAST_IP, 67, msg)

    // 2. Wait for OFFER
    let offer = udp_recv_timeout(socket, 5000)  // 5s timeout
    let offered_ip = dhcp_parse_offer(offer)

    // 3. REQUEST: accept the offered IP
    let req = dhcp_build_request(mac, offered_ip, offer.server_ip)
    udp_send(socket, BROADCAST_IP, 67, req)

    // 4. Wait for ACK
    let ack = udp_recv_timeout(socket, 5000)
    let config = dhcp_parse_ack(ack)

    // Apply: IP, netmask, gateway, DNS
    net_configure(config.ip, config.netmask, config.gateway, config.dns)
}
```

---

## 7. FastRPC / Hexagon NPU

### FastRPC Architecture

```
User Process (aarch64)        Hexagon DSP (CDSP)
┌─────────────┐               ┌─────────────┐
│ QNN SDK     │               │ HTP Skel    │
│ libQnnHtp.so│──FastRPC──────│ libQnnHtp   │
│             │  /dev/fastrpc │ V68Skel.so  │
│ Model .dlc  │  -cdsp        │             │
└─────────────┘               └─────────────┘
```

### QNN SDK Usage Pattern

```fajar
@device fn npu_load_and_infer(model_path: str, input: Tensor<f32>) -> Tensor<f32> {
    // 1. Open FastRPC session
    let session = fastrpc_open("/dev/fastrpc-cdsp")

    // 2. Create QNN context
    let ctx = qnn_context_create(session, QnnBackend::Htp)

    // 3. Load model graph
    let graph = qnn_graph_load(ctx, model_path)

    // 4. Create input tensor (zero-copy: map to DSP-visible memory)
    let qnn_input = qnn_tensor_create(ctx, input.shape(), input.data_ptr())

    // 5. Create output tensor
    let output_shape = qnn_graph_output_shape(graph, 0)
    let qnn_output = qnn_tensor_create_empty(ctx, output_shape)

    // 6. Execute inference on Hexagon 770
    qnn_graph_execute(graph, [qnn_input], [qnn_output])

    // 7. Read results
    let result = qnn_tensor_to_fajar(qnn_output)

    // 8. Cleanup
    qnn_tensor_free(qnn_input)
    qnn_tensor_free(qnn_output)
    qnn_graph_free(graph)
    qnn_context_free(ctx)
    fastrpc_close(session)

    result
}
```

### Model Pipeline (ONNX → NPU)

```bash
# Step 1: Convert ONNX to QNN
qairt-converter --input_network model.onnx \
  --output_path /tmp/model.cpp

# Step 2: Quantize to INT8 (for 12 TOPS performance)
qairt-quantizer --input_network model.onnx \
  --input_list calibration_data.txt \
  --act_bw 8 --weight_bw 8 \
  --output_path /tmp/model_quantized.cpp

# Step 3: Create context binary (optimized for Hexagon 770)
qnn-context-binary-generator \
  --model /tmp/model_quantized.cpp \
  --backend libQnnHtp.so \
  --output_dir /tmp/output/

# Output: /tmp/output/model.serialized.bin (deploy this to Q6A)
```

---

## 8. Adreno 643 GPU Compute

### Architecture

```
Adreno 643 @ 812 MHz
├── 1 Shader Processor (SP)
│   ├── ALU pipelines (FP32 + FP16 + INT)
│   ├── 773 GFLOPS FP32
│   └── 1581 GFLOPS FP16
├── Texture unit
├── L1/L2 cache
└── Command Processor (CP)
    └── Ring buffer for command submission

GPU base: 0x3D00_0000
Devfreq: /sys/class/devfreq/3d00000.gpu
```

### Compute Dispatch Pattern

```fajar
@device fn gpu_matmul(a: Tensor<f32>, b: Tensor<f32>) -> Tensor<f32> {
    let m = a.shape()[0]
    let k = a.shape()[1]
    let n = b.shape()[1]

    // Allocate GPU buffers
    let gpu_a = gpu_buffer_create(m * k * 4)  // f32 = 4 bytes
    let gpu_b = gpu_buffer_create(k * n * 4)
    let gpu_c = gpu_buffer_create(m * n * 4)

    // Upload data (CPU → GPU, zero-copy if possible)
    gpu_buffer_write(gpu_a, a.data_ptr(), m * k * 4)
    gpu_buffer_write(gpu_b, b.data_ptr(), k * n * 4)

    // Dispatch compute kernel
    gpu_dispatch(
        kernel: "matmul_f32",
        buffers: [gpu_a, gpu_b, gpu_c],
        workgroups: [m / 16, n / 16, 1],  // 16x16 tiles
        local_size: [16, 16, 1],
        uniforms: [m, k, n]
    )

    // Wait for completion
    gpu_sync()

    // Read result (GPU → CPU)
    let result = Tensor::zeros([m, n])
    gpu_buffer_read(gpu_c, result.data_ptr(), m * n * 4)

    // Cleanup
    gpu_buffer_free(gpu_a)
    gpu_buffer_free(gpu_b)
    gpu_buffer_free(gpu_c)

    result
}
```

---

## 9. MIPI CSI Camera

### CSI-2 Protocol

```
MIPI CSI-2 packet format:
┌──────┬──────┬──────┬────────────┬──────┐
│ SoF  │ DI   │ WC   │ Pixel Data │ CRC  │
│ sync │ 1B   │ 2B   │ variable   │ 2B   │
└──────┴──────┴──────┴────────────┴──────┘

Data Identifier (DI):
  [7:6] Virtual Channel (0-3)
  [5:0] Data Type:
    0x1E = YUV422 8-bit
    0x2A = RAW8
    0x2B = RAW10
    0x2C = RAW12
    0x24 = RGB888
```

### Camera Capture Pattern

```fajar
@kernel fn camera_capture_frame(csi_port: u32, width: u32, height: u32,
                                  buffer: u64) -> bool {
    let csid_base = match csi_port {
        0 => 0x0AC4_0000,  // CSID0 (4-lane)
        1 => 0x0AC4_4000,  // CSID1 (2-lane)
        2 => 0x0AC4_8000,  // CSID2 (2-lane)
        _ => return false,
    }

    // Configure CSI receiver
    volatile_write(csid_base + 0x00, 0x01)          // Enable
    volatile_write(csid_base + 0x04, width as u64)   // Frame width
    volatile_write(csid_base + 0x08, height as u64)  // Frame height
    volatile_write(csid_base + 0x0C, 0x1E)           // Data type: YUV422

    // Configure DMA output
    volatile_write(csid_base + 0x20, buffer)          // DMA destination
    volatile_write(csid_base + 0x24, (width * height * 2) as u64)  // Frame size
    volatile_write(csid_base + 0x28, 0x01)            // Start capture

    // Wait for frame complete interrupt
    let timeout: u32 = 1_000_000
    let mut t: u32 = 0
    while t < timeout {
        if volatile_read(csid_base + 0x30) & 0x01 != 0 {
            volatile_write(csid_base + 0x30, 0x01)  // Clear IRQ
            return true
        }
        t = t + 1
    }
    false  // Timeout
}
```

### IMX577 Sensor Init (via I2C)

```fajar
@kernel fn imx577_init(i2c_base: u64) {
    let addr: u8 = 0x1A  // IMX577 I2C address

    // Software reset
    i2c_write_reg16(i2c_base, addr, 0x0103, 0x01)
    sleep_us(10_000)  // 10ms reset delay

    // Set 1920x1080 @ 30fps
    i2c_write_reg16(i2c_base, addr, 0x0340, 0x08)  // Frame length MSB
    i2c_write_reg16(i2c_base, addr, 0x0341, 0xCA)  // Frame length LSB
    i2c_write_reg16(i2c_base, addr, 0x0342, 0x0D)  // Line length MSB
    i2c_write_reg16(i2c_base, addr, 0x0343, 0xA0)  // Line length LSB

    // Output format: RAW10
    i2c_write_reg16(i2c_base, addr, 0x0112, 0x0A)
    i2c_write_reg16(i2c_base, addr, 0x0113, 0x0A)

    // Start streaming
    i2c_write_reg16(i2c_base, addr, 0x0100, 0x01)
}
```

---

## 10. Cranelift Bare-Metal Codegen

### Target Configuration

```rust
// In src/codegen/cranelift/mod.rs — add bare-metal target
fn create_bare_metal_target() -> TargetIsa {
    let mut builder = settings::builder();
    builder.set("opt_level", "speed").unwrap();
    builder.set("is_pic", "false").unwrap();     // No PIC for kernel
    builder.set("use_colocated_libcalls", "true").unwrap();

    let flags = settings::Flags::new(builder);
    let target = target_lexicon::Triple::from_str("aarch64-unknown-none").unwrap();

    isa::lookup(target).unwrap().finish(flags).unwrap()
}
```

### Linker Script Template

```
/* FajarOS kernel linker script — aarch64 */
ENTRY(_start)

MEMORY {
    KERNEL (rwx) : ORIGIN = 0x40000000, LENGTH = 256M
}

SECTIONS {
    .text 0x40000000 : {
        KEEP(*(.vectors))    /* Exception vectors first */
        *(.text .text.*)
    } > KERNEL

    .rodata : ALIGN(4096) {
        *(.rodata .rodata.*)
    } > KERNEL

    .data : ALIGN(4096) {
        __data_start = .;
        *(.data .data.*)
        __data_end = .;
    } > KERNEL

    .bss : ALIGN(4096) {
        __bss_start = .;
        *(.bss .bss.*)
        *(COMMON)
        __bss_end = .;
    } > KERNEL

    __stack_top = ORIGIN(KERNEL) + LENGTH(KERNEL);
}
```

### No-Std Runtime Functions

```rust
// In src/codegen/cranelift/runtime_bare.rs
// These replace libc functions for bare-metal

#[no_mangle]
pub extern "C" fn fj_rt_memcpy(dst: *mut u8, src: *const u8, n: usize) -> *mut u8 {
    let mut i = 0;
    while i < n {
        unsafe { *dst.add(i) = *src.add(i); }
        i += 1;
    }
    dst
}

#[no_mangle]
pub extern "C" fn fj_rt_memset(dst: *mut u8, val: i32, n: usize) -> *mut u8 {
    let mut i = 0;
    while i < n {
        unsafe { *dst.add(i) = val as u8; }
        i += 1;
    }
    dst
}

#[no_mangle]
pub extern "C" fn fj_rt_print_bare(ptr: *const u8, len: usize) {
    // Output to UART (PL011 on QEMU, QUP on Q6A)
    const UART_BASE: usize = 0x09000000; // PL011 on QEMU virt
    let mut i = 0;
    while i < len {
        unsafe {
            let byte = *ptr.add(i);
            core::ptr::write_volatile(UART_BASE as *mut u8, byte);
        }
        i += 1;
    }
}
```

### QEMU Testing Commands

```bash
# Build bare-metal kernel
fj build --target aarch64-none --kernel kernel.fj -o fajaros.elf

# Boot in QEMU (bare ELF)
qemu-system-aarch64 -M virt -cpu cortex-a76 -m 4G \
  -kernel fajaros.elf -nographic -serial mon:stdio

# Boot with UEFI (EFI binary)
qemu-system-aarch64 -M virt -cpu cortex-a76 -m 4G \
  -bios /usr/share/qemu/OVMF.fd \
  -drive file=esp.img,format=raw,if=virtio \
  -nographic -serial mon:stdio

# Debug with GDB
qemu-system-aarch64 -M virt -cpu cortex-a76 -m 4G \
  -kernel fajaros.elf -nographic -serial mon:stdio \
  -s -S  # Wait for GDB on port 1234

# In another terminal:
gdb-multiarch fajaros.elf -ex "target remote :1234" -ex "break kernel_main"

# With GIC (v3)
qemu-system-aarch64 -M virt,gic-version=3 -cpu cortex-a76 -m 4G \
  -kernel fajaros.elf -nographic

# With networking
qemu-system-aarch64 -M virt,gic-version=3 -cpu cortex-a76 -m 4G \
  -kernel fajaros.elf -nographic \
  -netdev user,id=net0,hostfwd=tcp::2222-:22 \
  -device virtio-net,netdev=net0

# With NVMe drive
qemu-system-aarch64 -M virt,gic-version=3 -cpu cortex-a76 -m 4G \
  -kernel fajaros.elf -nographic \
  -drive file=rootfs.img,if=none,id=nvme0 \
  -device nvme,serial=fajaros,drive=nvme0
```

---

## 11. Exception Vector Table

### aarch64 Vector Layout

```
Offset    Source          Type
──────    ──────          ──────
0x000     Current EL SP0  Synchronous
0x080     Current EL SP0  IRQ
0x100     Current EL SP0  FIQ
0x180     Current EL SP0  SError

0x200     Current EL SPx  Synchronous  ← kernel uses these
0x280     Current EL SPx  IRQ
0x300     Current EL SPx  FIQ
0x380     Current EL SPx  SError

0x400     Lower EL A64    Synchronous  ← userspace syscalls
0x480     Lower EL A64    IRQ
0x500     Lower EL A64    FIQ
0x580     Lower EL A64    SError

0x600     Lower EL A32    Synchronous
0x680     Lower EL A32    IRQ
0x700     Lower EL A32    FIQ
0x780     Lower EL A32    SError
```

### Vector Stub Pattern

```fajar
// Each vector entry is 128 bytes (32 instructions max)
@kernel fn exception_vector_stub() {
    // Save all registers to stack
    asm!("sub sp, sp, #272")           // 34 * 8 = 272 bytes
    asm!("stp x0, x1, [sp, #0]")
    asm!("stp x2, x3, [sp, #16]")
    asm!("stp x4, x5, [sp, #32]")
    // ... save x6-x29
    asm!("stp x28, x29, [sp, #224]")
    asm!("mrs x0, elr_el1")
    asm!("mrs x1, spsr_el1")
    asm!("stp x30, x0, [sp, #240]")   // LR + ELR
    asm!("str x1, [sp, #256]")         // SPSR

    // Read exception info
    asm!("mrs x0, esr_el1")            // Exception syndrome
    asm!("mrs x1, elr_el1")            // Return address
    asm!("mrs x2, far_el1")            // Fault address
    asm!("mov x3, sp")                 // Register context pointer

    // Call handler
    asm!("bl exception_dispatch")

    // Restore all registers
    asm!("ldr x1, [sp, #256]")
    asm!("ldp x30, x0, [sp, #240]")
    asm!("msr spsr_el1, x1")
    asm!("msr elr_el1, x0")
    asm!("ldp x0, x1, [sp, #0]")
    asm!("ldp x2, x3, [sp, #16]")
    // ... restore x4-x29
    asm!("add sp, sp, #272")
    asm!("eret")
}
```

### ESR_EL1 Exception Class Decoding

```
ESR_EL1.EC (bits [31:26]):
  0x15 = SVC (syscall from AArch64)
  0x18 = MSR/MRS trap
  0x20 = Instruction abort (lower EL)
  0x21 = Instruction abort (same EL)
  0x24 = Data abort (lower EL)
  0x25 = Data abort (same EL)
  0x2C = FP/SIMD access trap
```

---

## 12. Context Switch

```fajar
@kernel fn context_switch(old: *mut ProcessContext, new: *const ProcessContext) {
    // Save callee-saved registers (x19-x30, sp)
    asm!("stp x19, x20, [{}, #0]",  in(reg) old)
    asm!("stp x21, x22, [{}, #16]", in(reg) old)
    asm!("stp x23, x24, [{}, #32]", in(reg) old)
    asm!("stp x25, x26, [{}, #48]", in(reg) old)
    asm!("stp x27, x28, [{}, #64]", in(reg) old)
    asm!("stp x29, x30, [{}, #80]", in(reg) old)
    asm!("mov x0, sp")
    asm!("str x0, [{}, #96]",       in(reg) old)

    // Save TTBR0 (page table)
    asm!("mrs x0, ttbr0_el1")
    asm!("str x0, [{}, #104]",      in(reg) old)

    // Restore new process
    asm!("ldr x0, [{}, #104]",      in(reg) new)
    asm!("msr ttbr0_el1, x0")       // Switch address space
    asm!("tlbi vmalle1")             // Invalidate TLB
    asm!("dsb ish")
    asm!("isb")

    asm!("ldr x0, [{}, #96]",       in(reg) new)
    asm!("mov sp, x0")
    asm!("ldp x29, x30, [{}, #80]", in(reg) new)
    asm!("ldp x27, x28, [{}, #64]", in(reg) new)
    asm!("ldp x25, x26, [{}, #48]", in(reg) new)
    asm!("ldp x23, x24, [{}, #32]", in(reg) new)
    asm!("ldp x21, x22, [{}, #16]", in(reg) new)
    asm!("ldp x19, x20, [{}, #0]",  in(reg) new)

    asm!("ret")  // Return to new process (via x30)
}
```

---

*V3.0 "Surya" Skills — FajarOS Technical Reference*
*Read the relevant section BEFORE implementing each sprint.*
