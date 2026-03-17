# FajarOS — Roadmap to World-Class Edge AI Operating System

> **Vision:** The world's best operating system for edge AI —
> the only OS where kernel, drivers, networking, GPU, and neural networks
> share one language, one type system, one compiler, with safety guarantees
> that no existing OS provides.
>
> **Niche:** "Best OS for edge AI written in a single language with compiler-enforced safety"
> **No direct competitor exists.**

---

## Current Status (v3.0 "Surya" Phase 1-2 Complete)

```
✅ Bare-metal aarch64 boot (EL1)
✅ MMU (2MB block mappings)
✅ Exception handling (vector table, SVC, IRQ)
✅ GICv3 interrupt controller
✅ Timer API (sleep_ms, 1ms accuracy)
✅ Preemptive scheduler (3-process round-robin)
✅ IPC (mailbox send/recv/reply)
✅ Interactive shell (63 commands)
✅ RAM filesystem (ls, touch, rm, cp, mkdir, cat, stat)
✅ FAT16 filesystem (mount, lsfat, catfat, df, writefat, rmfat)
✅ Self-test suite (5/5 pass)
✅ Kernel log (dmesg)
✅ Environment variables
```

**Lines of Fajar Lang kernel code: ~3,450**
**Target hardware: Radxa Dragon Q6A (Qualcomm QCS6490)**

---

## Gap Analysis: What "World's Best" Requires

### Tier 1: Must-Have (Without these, not a real OS)

| # | Feature | Linux | FajarOS | Priority | Effort |
|---|---------|-------|---------|----------|--------|
| 1 | **Persistent storage** (NVMe/eMMC/SD) | ext4, btrfs, NVMe driver | RAM-only | CRITICAL | 4 sprints |
| 2 | **TCP/IP networking** | Full stack | None | CRITICAL | 6 sprints |
| 3 | **Multi-core SMP** | Up to 1000+ cores | Single core | HIGH | 3 sprints |
| 4 | **USB stack** | Full USB 2.0/3.0 | None | HIGH | 3 sprints |
| 5 | **Display output** | Wayland/X11, GPU accel | None | HIGH | 4 sprints |
| 6 | **Real-time clock** | hwclock, NTP | Boot-relative only | MEDIUM | 1 sprint |
| 7 | **Power management** | cpufreq, suspend | None | MEDIUM | 2 sprints |
| 8 | **Watchdog timer** | Hardware WDT | None | MEDIUM | 1 sprint |

### Tier 2: Competitive Edge (Makes FajarOS unique)

| # | Feature | Any Other OS? | Priority | Effort |
|---|---------|---------------|----------|--------|
| 9 | **NPU inference from kernel** | No | CRITICAL | 2 sprints |
| 10 | **GPU compute from kernel** | No (except CUDA) | HIGH | 2 sprints |
| 11 | **Camera → NPU → GPIO pipeline** | No (in kernel) | HIGH | 3 sprints |
| 12 | **Formal safety proof** (@kernel/@device/@safe) | seL4 (different approach) | HIGH | 4 sprints |
| 13 | **Hot-reload AI models** | No (in kernel) | MEDIUM | 1 sprint |
| 14 | **On-device Fajar Lang REPL** | No | MEDIUM | 2 sprints |
| 15 | **Zero-copy sensor → AI pipeline** | No | MEDIUM | 2 sprints |

### Tier 3: Production Quality

| # | Feature | Description | Priority | Effort |
|---|---------|-------------|----------|--------|
| 16 | **OTA updates** | Secure firmware update | HIGH | 2 sprints |
| 17 | **Secure boot** | Verified boot chain | HIGH | 2 sprints |
| 18 | **Sandboxing** | Process isolation via MMU | HIGH | 2 sprints |
| 19 | **Crash recovery** | Kernel panic → reboot + log | MEDIUM | 1 sprint |
| 20 | **Performance profiling** | CPU, memory, IRQ stats | MEDIUM | 1 sprint |
| 21 | **Documentation** | mdBook, API reference | MEDIUM | 2 sprints |
| 22 | **CI/CD** | Automated build + QEMU test | MEDIUM | 1 sprint |
| 23 | **Benchmarks** | Published, reproducible | MEDIUM | 1 sprint |

### Tier 4: Community & Ecosystem

| # | Feature | Description | Priority | Effort |
|---|---------|-------------|----------|--------|
| 24 | **Academic paper** | "Compiler-Enforced OS Safety for Edge AI" | HIGH | 1 month |
| 25 | **Demo video** | Boot → shell → AI inference on Q6A | HIGH | 1 week |
| 26 | **Website** | fajaros.dev with docs + downloads | MEDIUM | 1 week |
| 27 | **Package manager** | `fjpkg install sensor-driver` | MEDIUM | 3 sprints |
| 28 | **Community** | GitHub issues, Discord, contributors | MEDIUM | Ongoing |
| 29 | **Conference talk** | FOSDEM, Embedded World, OSDI | LOW | 1 year |
| 30 | **Hardware partners** | Pre-installed on SBCs | LOW | 1 year |

---

## Implementation Plan: 12 Milestones

### Milestone 1: Persistent Storage (v3.1)
**Goal:** Read/write files on NVMe SSD from FajarOS
**Duration:** 4 sprints (40 tasks)
**Depends on:** Phase 2 complete ✅

| Sprint | Tasks | Description |
|--------|-------|-------------|
| S16 | 10 | VirtIO-blk driver: MMIO probe, v1 legacy init, feature negotiate ✅ |
| S17 | 10 | VirtIO I/O: descriptor chains, avail/used rings, sector read/write, sync ✅ |
| S18 | 10 | FAT16 filesystem: read directories, read files, parse BPB ✅ |
| S19 | 10 | FAT16 write: create files, write data, update FAT table ✅ |

**Gate:** `mount` ✅ (virtio), `lsfat` ✅, `catfat` ✅, `writefat` ✅, `rmfat` ✅, `sync` ✅ (persistent!)
**ALL COMPLETE** — data persists to disk image across reboots

### Milestone 2: TCP/IP Networking (v3.2)
**Goal:** FajarOS responds to ping, serves HTTP
**Duration:** 6 sprints (60 tasks)

| Sprint | Tasks | Description |
|--------|-------|-------------|
| S20 | 10 | Ethernet RGMII driver (QCS6490 EMAC), DMA ring buffers |
| S21 | 10 | ARP, IPv4 headers, ICMP echo (ping) |
| S22 | 10 | UDP socket, DNS resolver, DHCP client |
| S23 | 10 | TCP state machine, 3-way handshake, data transfer |
| S24 | 10 | HTTP server (static pages), WebSocket |
| S25 | 10 | WiFi driver (QCA6490), WPA2, connection manager |

**Gate:** `ping 8.8.8.8`, `curl http://fajaros-q6a/`, SSH server

### Milestone 3: Multi-Core SMP (v3.3)
**Goal:** All 8 Kryo 670 cores running FajarOS
**Duration:** 3 sprints (30 tasks)

| Sprint | Tasks | Description |
|--------|-------|-------------|
| S26 | 10 | PSCI secondary core boot, per-CPU stack, GIC affinity routing |
| S27 | 10 | Spinlocks, per-CPU scheduler, load balancing |
| S28 | 10 | big.LITTLE awareness: A78 for compute, A55 for background |

**Gate:** `top` shows 8 cores, processes migrate between cores

### Milestone 4: AI Integration (v3.4) ← UNIQUE FEATURE
**Goal:** NPU inference from FajarOS kernel, camera → AI → GPIO
**Duration:** 5 sprints (50 tasks)

| Sprint | Tasks | Description |
|--------|-------|-------------|
| S29 | 10 | Hexagon 770 FastRPC driver, DSP communication channel |
| S30 | 10 | QNN model loading from NVMe, INT8 inference via HTP |
| S31 | 10 | Adreno 643 GPU compute driver, Vulkan minimal, tensor ops |
| S32 | 10 | MIPI CSI camera driver, V4L2-like frame capture |
| S33 | 10 | End-to-end: Camera frame → preprocess → NPU → GPIO actuator |

**Gate:** Live camera → object detection → LED blink on detection
**THIS IS THE KILLER FEATURE — no other OS can do this natively**

### Milestone 5: Display & GUI (v3.5)
**Goal:** HDMI output with framebuffer, basic window system
**Duration:** 4 sprints (40 tasks)

| Sprint | Tasks | Description |
|--------|-------|-------------|
| S34 | 10 | HDMI/DSI display driver, linear framebuffer |
| S35 | 10 | Font rendering (bitmap), text console on screen |
| S36 | 10 | Basic compositor: windows, cursor, input events |
| S37 | 10 | GPU-accelerated 2D (rectangles, lines, text via Adreno) |

**Gate:** FajarOS desktop with terminal window on HDMI monitor

### Milestone 6: USB Stack (v3.6)
**Goal:** USB keyboard, mouse, storage
**Duration:** 3 sprints (30 tasks)

| Sprint | Tasks | Description |
|--------|-------|-------------|
| S38 | 10 | USB host controller (XHCI/DWC3), enumeration, descriptors |
| S39 | 10 | USB HID: keyboard + mouse drivers |
| S40 | 10 | USB mass storage: mount USB drive, read/write files |

**Gate:** Type on USB keyboard → shell responds, USB drive mounted

### Milestone 7: Security & Safety (v3.7)
**Goal:** Verified boot, process sandboxing, formal safety claims
**Duration:** 4 sprints (40 tasks)

| Sprint | Tasks | Description |
|--------|-------|-------------|
| S41 | 10 | Per-process page tables, user/kernel address space split |
| S42 | 10 | Syscall interface: all kernel services via SVC trap |
| S43 | 10 | Capability-based security: process can only access granted resources |
| S44 | 10 | Formal proof sketch: @kernel/@device/@safe type-level isolation |

**Gate:** Malicious process cannot access other process memory or hardware

### Milestone 8: Production Quality (v3.8)
**Goal:** Stable enough for 24/7 edge deployment
**Duration:** 4 sprints (40 tasks)

| Sprint | Tasks | Description |
|--------|-------|-------------|
| S45 | 10 | Crash recovery: panic → log → watchdog reboot |
| S46 | 10 | OTA update: download firmware, verify signature, apply |
| S47 | 10 | Performance monitoring: CPU/memory/IRQ/network stats |
| S48 | 10 | Stress testing: 7-day continuous operation, memory leak detection |

**Gate:** FajarOS runs 7 days without crash on Q6A

### Milestone 9: Developer Experience (v3.9)
**Goal:** Developers can build apps for FajarOS
**Duration:** 3 sprints (30 tasks)

| Sprint | Tasks | Description |
|--------|-------|-------------|
| S49 | 10 | On-device Fajar Lang REPL with hardware builtins |
| S50 | 10 | Package manager: `fjpkg install`, dependency resolution |
| S51 | 10 | SDK: cross-compile apps on host, deploy to Q6A |

**Gate:** `fjpkg install sensor-dashboard && fj run sensor-dashboard.fj`

### Milestone 10: Documentation & Community (v3.10)
**Goal:** Others can understand, use, and contribute to FajarOS
**Duration:** 2 sprints (20 tasks)

| Sprint | Tasks | Description |
|--------|-------|-------------|
| S52 | 10 | mdBook documentation: architecture, API, tutorials |
| S53 | 10 | Academic paper, demo video, conference submission |

**Gate:** 10+ GitHub stars, 1+ external contributor, 1 paper submitted

### Milestone 11: Ecosystem (v4.0)
**Goal:** FajarOS as a platform, not just an OS
**Duration:** 4 sprints (40 tasks)

| Sprint | Tasks | Description |
|--------|-------|-------------|
| S54 | 10 | App store: curated Fajar Lang applications |
| S55 | 10 | Board support: Raspberry Pi 5, Jetson, other ARM64 SBCs |
| S56 | 10 | Cloud integration: telemetry, remote management, fleet OTA |
| S57 | 10 | AI model marketplace: pre-trained models for common tasks |

**Gate:** FajarOS running on 3+ different hardware platforms

### Milestone 12: World-Class (v5.0)
**Goal:** Recognized as the best edge AI OS
**Duration:** Ongoing

| Metric | Target |
|--------|--------|
| GitHub stars | 1,000+ |
| Contributors | 50+ |
| Supported boards | 10+ |
| Published papers | 3+ |
| Conference talks | 5+ |
| Production deployments | 100+ |
| Uptime record | 365 days |
| Benchmark: boot → first inference | < 100ms |
| Benchmark: camera → AI → GPIO latency | < 10ms |

---

## Timeline Estimate

| Quarter | Milestones | Key Deliverable |
|---------|-----------|-----------------|
| **Q2 2026** | M1-M2 (Storage + Network) | FajarOS online, reads NVMe, responds to ping |
| **Q3 2026** | M3-M4 (SMP + AI) | 8-core, camera → NPU → GPIO pipeline |
| **Q4 2026** | M5-M6 (Display + USB) | GUI desktop, USB keyboard |
| **Q1 2027** | M7-M8 (Security + Production) | 24/7 stable, verified boot |
| **Q2 2027** | M9-M10 (Dev + Docs) | SDK, paper, community launch |
| **Q3 2027** | M11 (Ecosystem) | Multi-platform, app store |
| **Q4 2027** | M12 (World-Class) | 1000 stars, conference talks |

---

## Competitive Positioning

```
                    Safety ──────────────────►
                    │
                    │   seL4 (formal proof,      FajarOS (compiler-enforced,
                    │    but C, no AI)            single language, AI built-in)
                    │         ●                          ★
   AI Integration   │
        │           │
        │           │
        ▼           │   Zephyr (C, RTOS,         Linux (C, everything,
                    │    no AI in kernel)          but unsafe, no AI in kernel)
                    │         ●                          ●
                    │
                    └─────────────────────────────────────
```

**FajarOS occupies a unique position: high safety + high AI integration.**
No other OS is in this quadrant.

---

## Key Differentiators (Marketing Messages)

1. **"One Language, One Kernel, One AI"** — From boot to neural network inference, everything is Fajar Lang
2. **"If it compiles, it's safe"** — @kernel/@device/@safe prevents entire categories of bugs at compile time
3. **"Boot to inference in 100ms"** — Fastest path from power-on to AI result
4. **"No unsafe keyword"** — Unlike Rust OS projects, FajarOS has no escape hatch for safety
5. **"Edge AI, not cloud AI"** — Private, low-latency, always-on intelligence

---

*FajarOS Roadmap v1.0 | March 2026 | PrimeCore.id*
*"The sun rises on a new kind of operating system"*
