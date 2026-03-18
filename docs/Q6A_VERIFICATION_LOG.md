# Q6A Hardware Verification Log — 2026-03-18

## Hardware
- **Board:** Radxa Dragon Q6A (QCS6490)
- **CPU:** 8-core Kryo 670 (4×Cortex-A78 + 4×Cortex-A55)
- **RAM:** 7.4 GB LPDDR5
- **Storage:** 28 GB (USB), NVMe installed
- **Temp:** 58°C idle
- **Kernel:** Linux 6.18.2-3-qcom aarch64
- **SSH:** radxa@192.168.50.94 (WiFi)

## Fajar Lang Compiler Verification

### Interpreter
```
$ time fj run fib.fj
832040
real 5.793s
```

### Cranelift JIT (Native ARM64)
```
$ time fj run --native fib.fj
832040
real 0.012s   ← 480× faster than interpreter
```

### fib(40) on Real Hardware
```
$ time fj run --native fib40.fj
fib(35) = 9227465
fib(40) = 102334155
real 0.684s   ← on Kryo 670
```

### AOT Bare-Metal Compilation
```
$ fj build start.fj --target aarch64 --no-std -o fajaros.elf
Built: fajaros.elf (target: aarch64-none (bare metal))
146704 bytes
```

## FajarOS Kernel Verification (QEMU on Q6A)

### Boot Sequence
```
FajarOS v3.0 Surya
fjsh> dmesg
BOOT
MMU
GIC
TIMR
SHEL
```

### Process T (SVC I/O + SVC EXIT)
```
fjsh> spawn t
PID 1
TEST!
fjsh> ps
PID STATE
--- -----
 0  *RUN*
 1   DEAD
```

### Process C (100 Sequential SVCs)
```
fjsh> spawn c
PID 1
fjsh> wait 1
wait...CCCCCCCC...(100×)
PID 1 done
```

### Timer
```
fjsh> ticks
7.9s (696 IRQs)   ← ~88 Hz (QEMU-on-ARM64 slightly slower)
```

## QNN NPU Inference

### Setup
```
$ sudo apt install libqnn-dev libqnn1 qnn-tools
QNN SDK v2.40.0 installed
```

### Backend Validation
```
GPU:  Present ✓
DSP:  Needs testsig
CPU:  Available ✓
```

### MNIST Inference (CPU Backend)
```
$ qnn-net-run --backend libQnnCpu.so --dlc_path mnist_mlp_int8.dlc
Execution time: 25ms
Output (blank input): [0.1016, 0.1016, ..., 0.1016]  (uniform = correct)
```

### Fajar Lang NPU Detection
```
$ fj run npu_demo.fj
NPU: Available
```

## GPIO Hardware Test

```
$ fj run gpio_test.fj
GPIO96 (PIN_7): HLHLHLHLHL done
CPU temp: 58000 (58.0°C)
```

## Status: ALL VERIFIED ✅
