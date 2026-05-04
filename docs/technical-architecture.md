# x86ochs — Technical Architecture

> Deep-dive reference for the internal data flows, component interfaces, and subsystem designs of the Bochs-based x86ochs emulator, with Mermaid diagrams at each level.

---

## Table of Contents

1. [Instruction Execution Pipeline](#1-instruction-execution-pipeline)
2. [CPU Class Hierarchy](#2-cpu-class-hierarchy)
3. [Register File Layout](#3-register-file-layout)
4. [Memory Subsystem](#4-memory-subsystem)
5. [Paging & Address Translation](#5-paging--address-translation)
6. [I/O Device Bus Architecture](#6-io-device-bus-architecture)
7. [Interrupt & Exception Delivery](#7-interrupt--exception-delivery)
8. [Timer & Scheduling Model](#8-timer--scheduling-model)
9. [Instruction Cache (iCache)](#9-instruction-cache-icache)
10. [Decoder Pipeline](#10-decoder-pipeline)
11. [SIMD / Vector Extension Architecture](#11-simd--vector-extension-architecture)
12. [Virtual Machine Extensions (VMX/SVM)](#12-virtual-machine-extensions-vmxsvm)
13. [Neural-Tensor Architecture Counterpart](#13-neural-tensor-architecture-counterpart)

---

## 1. Instruction Execution Pipeline

Bochs implements a **fetch–decode–execute** interpreter loop. There is no branch prediction, no out-of-order execution, no pipeline stall — every instruction completes atomically from the emulator's perspective.

```mermaid
sequenceDiagram
    participant Host as Host Thread
    participant PC  as pc_system
    participant CPU as BX_CPU_C
    participant IC  as iCache
    participant DEC as Decoder
    participant EXE as Exec Handler
    participant MEM as Memory / TLB
    participant IO  as I/O Devices

    Host->>PC: bx_pc_system.tickn(n)
    PC->>PC: advance timers
    PC-->>CPU: service_HRQ() if DMA pending
    PC->>CPU: cpu_loop()

    loop per instruction
        CPU->>IC: iCache.get(CS:EIP)
        alt cache miss
            IC->>DEC: fetchDecode(CS:EIP, ptr, remain)
            DEC-->>IC: bxInstruction_c
        end
        IC-->>CPU: bxInstruction_c i

        CPU->>EXE: i.execute(this)

        alt memory operand
            EXE->>MEM: read_virtual / write_virtual
            MEM->>MEM: TLB lookup
            alt TLB miss
                MEM->>MEM: page_walk(CR3, lin_addr)
            end
            MEM-->>EXE: data
        end

        alt I/O instruction (IN/OUT)
            EXE->>IO: inp/outp(port, len)
            IO-->>EXE: data
        end

        EXE->>CPU: update RIP, flags, regs
        CPU->>CPU: check_async_event()
    end
```

---

## 2. CPU Class Hierarchy

```mermaid
classDiagram
    class BX_CPU_C {
        +Bit32u gen_reg[BX_GENERAL_REGISTERS]
        +bx_segment_reg_t sregs[6]
        +bx_eflags_t eflags
        +Bit64u rip
        +bx_cr0_t cr0
        +Bit64u cr2, cr3, cr4
        +XMM_REG xmm[BX_XMM_REGISTERS]
        +bx_fpu_t the_i387
        +cpu_loop()
        +fetchDecode32()
        +fetchDecode64()
        +handleAsyncEvent()
        +deliver_INIT()
        +deliver_NMI()
        +exception(vector, error)
        +pagingEnabled()
        +read_virtual_*()
        +write_virtual_*()
    }

    class bxInstruction_c {
        +Bit16u execute_ptr
        +Bit8u  ilen
        +Bit8u  modrm, sib, rex
        +bx_operand_16  src, dst
        +executePtr execute
    }

    class bx_segment_reg_t {
        +Bit16u selector
        +bx_descriptor_t cache
        +bool valid
    }

    class bx_descriptor_t {
        +Bit32u base
        +Bit32u limit
        +Bit8u  ar_byte
        +bool   segment
        +bool   valid
    }

    class BxExecutePtr_t {
        <<typedef>>
        +void (*fn)(BX_CPU_C*, bxInstruction_c*)
    }

    BX_CPU_C "1" --> "many" bxInstruction_c : fetches
    BX_CPU_C "1" --> "6"   bx_segment_reg_t : contains
    bx_segment_reg_t --> bx_descriptor_t : cache
    bxInstruction_c --> BxExecutePtr_t    : dispatch
```

---

## 3. Register File Layout

```mermaid
graph TB
    subgraph GPR["General-Purpose Registers (16 × 64-bit in 64-bit mode)"]
        RAX["RAX / EAX / AX / AH,AL"]
        RCX["RCX / ECX / CX"]
        RDX["RDX / EDX / DX"]
        RBX["RBX / EBX / BX"]
        RSP["RSP / ESP / SP  (stack pointer)"]
        RBP["RBP / EBP / BP  (base pointer)"]
        RSI["RSI / ESI / SI"]
        RDI["RDI / EDI / DI"]
        R8_15["R8 – R15  (64-bit mode only)"]
    end

    subgraph SIMD_REG["SIMD Registers"]
        XMM["XMM0–XMM15  (128-bit SSE)"]
        YMM["YMM0–YMM15  (256-bit AVX)  — upper 128 of XMM"]
        ZMM["ZMM0–ZMM31  (512-bit AVX-512)  — upper 384 of YMM"]
        KMASK["k0–k7  (AVX-512 opmask registers)"]
    end

    subgraph FPU_REG["x87 FPU Register Stack"]
        ST0["ST(0)  — top of stack"]
        ST1["ST(1)"]
        ST2_7["ST(2) – ST(7)"]
        CTRL["FPU Control Word / Status Word / Tag Word"]
    end

    subgraph SPEC_REG["Special Registers"]
        RIP["RIP  (instruction pointer)"]
        RFLAGS["RFLAGS  (CF, PF, AF, ZF, SF, TF, IF, DF, OF, …)"]
        CR["CR0 / CR2 / CR3 / CR4 / CR8"]
        DR["DR0–DR7  (debug registers)"]
        EFER["EFER  (extended feature enable: LME, LMA, NX, …)"]
        MTRR["MTRRs  (memory type range registers)"]
    end

    GPR --- SIMD_REG
    SIMD_REG --- FPU_REG
    FPU_REG --- SPEC_REG
```

---

## 4. Memory Subsystem

```mermaid
graph TD
    GUEST["Guest Virtual Address\n(linear address after segmentation)"]
    SEG["Segment Base + Offset\nsregs[CS/DS/ES/FS/GS/SS].cache.base"]
    LIN["Linear Address\n(32-bit or 64-bit)"]
    TLB["TLB lookup\ntlb.h  –  4096-entry direct-mapped"]
    PTREE["Page Table Walk\nCR3 → PML4 → PDPT → PD → PT → PTE"]
    PHYS["Physical Address\n(≤52 bits)"]
    MTRR_LU["MTRR / PAT lookup\n(memory type: WB / WC / UC / WT)"]
    PHYSMEM["BX Physical Memory\nmemory.cc  –  host std::vector<Bit8u>"]
    MMIO["MMIO region\n(device registers)"]
    ROM["ROM / BIOS shadow\nmisc_mem.cc"]

    GUEST --> SEG --> LIN --> TLB
    TLB -- hit --> PHYS
    TLB -- miss --> PTREE --> PHYS
    PHYS --> MTRR_LU
    MTRR_LU --> PHYSMEM
    MTRR_LU --> MMIO
    MTRR_LU --> ROM
```

### Memory Map (typical PC layout)

| Range | Contents |
|---|---|
| `0x00000000 – 0x0009FFFF` | Conventional RAM (640 KB) |
| `0x000A0000 – 0x000BFFFF` | VGA frame buffer (MMIO) |
| `0x000C0000 – 0x000FFFFF` | Option ROMs, BIOS shadow |
| `0x00100000 – RAM_end` | Extended RAM |
| `0xFEC00000` | I/O APIC MMIO |
| `0xFEE00000` | Local APIC MMIO |
| `0xFFFF0000 – 0xFFFFFFFF` | BIOS ROM (reset vector at 0xFFFFFFF0) |

---

## 5. Paging & Address Translation

```mermaid
flowchart LR
    LA["Linear Address\n[63:0]"]

    subgraph PML4["PML4 (CR3-based, 64-bit)"]
        PML4E["PML4E\nbits [47:39]"]
    end
    subgraph PDPT["PDPT"]
        PDPTE["PDPTE\nbits [38:30]"]
    end
    subgraph PD["Page Directory"]
        PDE["PDE\nbits [29:21]"]
    end
    subgraph PT["Page Table"]
        PTE["PTE\nbits [20:12]"]
    end

    PAGE["Page Frame\nbits [11:0] = offset"]
    PA["Physical Address"]

    LA --> PML4E --> PDPTE --> PDE --> PTE --> PAGE --> PA

    CR3(("CR3\nPML4 base")) --> PML4E
```

**Fast path via TLB:**  
`tlb.h` maintains a direct-mapped software TLB. On a hit, the linear→physical translation is resolved in a single array lookup, bypassing the four-level walk entirely.

---

## 6. I/O Device Bus Architecture

```mermaid
graph TB
    CPU_IO["CPU  IN/OUT instruction\nBX_CPU_C::inp/outp"]
    IOPORT["I/O Port Address Space\n64 KB × (read/write handler)"]

    PCI_HOST["PCI Host Bridge\npci.cc  –  Config Space 0xCF8/0xCFC"]
    ISA_BRIDGE["PCI-to-ISA Bridge\npci2isa.cc  –  IRQ routing"]

    subgraph ISA_DEV["Legacy ISA Devices"]
        PIC_ISA["PIC  8259A\npic.cc"]
        PIT_ISA["PIT  8254\npit.cc"]
        CMOS["CMOS / RTC\ncmos.cc"]
        FDD["Floppy Controller\nfloppy.cc"]
        KBD["PS/2 Keyboard\nkeyboard.cc"]
        SER["Serial (16550)\nserial.cc"]
        PAR["Parallel Port\nparallel.cc"]
        DMA_ISA["DMA 8237\ndma.cc"]
    end

    subgraph PCI_DEV["PCI Devices"]
        IDE["IDE / PATA\npci_ide.cc"]
        NIC["Network  (NE2000 / E1000)\nnetwork/"]
        VGA_PCI["VGA / SVGA\ndisplay/vga.cc"]
        USB_HC["USB Host Controllers\nusb/"]
        SND["Sound (ES1370 / SB16)\nsound/"]
        ACPI["ACPI Controller\nacpi.cc"]
        HPET["HPET Timer\nhpet.cc"]
    end

    CPU_IO --> IOPORT
    IOPORT --> PCI_HOST
    IOPORT --> ISA_DEV
    PCI_HOST --> ISA_BRIDGE
    ISA_BRIDGE --> ISA_DEV
    PCI_HOST --> PCI_DEV
```

---

## 7. Interrupt & Exception Delivery

```mermaid
stateDiagram-v2
    [*] --> Running : cpu_loop()

    Running --> CheckAsync : end of each insn\ncheck_async_event()

    CheckAsync --> SMI   : SMI pending
    CheckAsync --> NMI   : NMI pending
    CheckAsync --> INTR  : INTR & IF=1
    CheckAsync --> Running : nothing pending

    SMI   --> SMM_Handler : RSM saves state → SMBASE
    NMI   --> NMI_Handler : IDT[2]
    INTR  --> INTA_Cycle  : PIC/APIC INTA
    INTA_Cycle --> IRQ_Handler : IDT[n]

    SMM_Handler --> Running : RSM restores
    NMI_Handler --> Running
    IRQ_Handler --> Running

    Running --> Fault : #GP / #PF / #UD / …
    Fault --> ExceptionDispatch : exception(vector, error_code)
    ExceptionDispatch --> IDT_Lookup : IDT[vector]
    IDT_Lookup --> Handler_Frame : push CS:RIP:RFLAGS\n(+ error code)
    Handler_Frame --> Running : guest ISR executes
```

**Exception classes:**

| Class | Behaviour | Examples |
|---|---|---|
| Fault | RIP points to faulting instruction | #GP, #PF, #UD, #NP |
| Trap | RIP points to *next* instruction | #DB (step), #OF |
| Abort | No reliable restart | #MC, #DF |

---

## 8. Timer & Scheduling Model

```mermaid
graph LR
    MAIN["bx_pc_system.tickn(n)"]

    subgraph TIMER_SLOTS["Timer Slots  (BX_MAX_TIMERS = 64)"]
        T0["Slot 0  PIT channel 0\n1.19 MHz → IRQ0"]
        T1["Slot 1  PIT channel 1\nRAM refresh (legacy)"]
        T2["Slot 2  PIT channel 2\nPC speaker"]
        T3["Slot 3  RTC alarm\n32.768 kHz → IRQ8"]
        T4["Slot 4  APIC timer\nper-CPU countdown"]
        TN["Slot N  USB frame timer\nNIC poll, sound DMA…"]
    end

    MAIN --> T0
    MAIN --> T1
    MAIN --> T2
    MAIN --> T3
    MAIN --> T4
    MAIN --> TN

    T0 -- "fires" --> PIC_ISA2["PIC → IRQ0 → CPU"]
    T3 -- "fires" --> RTC2["RTC → IRQ8 → CPU"]
    T4 -- "fires" --> LAPIC2["LAPIC timer interrupt"]
```

Timer resolution is **one CPU tick** (1 / `cpu_ips`). Devices request a period in ticks; `pc_system` fires the callback when `tickcount >= timeToFire`.

---

## 9. Instruction Cache (iCache)

The iCache is a software-only optimization that avoids re-decoding the same bytes repeatedly:

```mermaid
graph TD
    KEY["Cache Key\n= hash(CS.base + EIP)"]
    ENTRY["bxICacheEntry_t\n  – tag: CS.base + EIP\n  – bxInstruction_c[BX_MAX_TRACE_LENGTH]\n  – ilen_remaining"]

    LOOKUP["iCache::get(CS.base, EIP, entry)"]
    VALID["tag match AND\nbyte-hash match\n(detect SMC)"]
    DECODE["Decoder runs\nfetchDecode32/64()"]
    STORE["iCache::set(entry)"]

    LOOKUP --> VALID
    VALID -- hit --> ENTRY
    VALID -- miss / SMC --> DECODE --> STORE --> ENTRY
```

Self-modifying code (SMC) is detected by hashing the instruction bytes at cache-store time and re-validating on each use. This preserves correctness at the cost of a small per-fetch overhead.

---

## 10. Decoder Pipeline

```mermaid
flowchart TD
    BYTES["Raw bytes from\nphysical memory\n(up to 15 bytes per insn)"]

    REX["REX prefix scan\n(64-bit mode)"]
    VEX["VEX/EVEX prefix scan\n(AVX/AVX-512)"]
    SEG_PFX["Segment override\nOperand/address size prefixes"]
    LOCK_REP["LOCK / REP / REPNE prefixes"]

    OPCODE["Primary opcode byte\n(1 or 2 bytes: 0F xx)"]
    MODRM["ModRM byte\n+ optional SIB + displacement"]
    IMM["Immediate operand\n(1/2/4/8 bytes)"]

    DISPATCH["Dispatch table lookup\nbxOpcode_t → execute fn ptr"]
    RESULT["bxInstruction_c\n(execute ptr, operand descriptors,\nilen, prefixes)"]

    BYTES --> REX --> VEX --> SEG_PFX --> LOCK_REP
    LOCK_REP --> OPCODE --> MODRM --> IMM --> DISPATCH --> RESULT
```

The dispatch table is a two-level array: `BxOpcodeInfo32[256]` (primary) and `BxOpcodeInfo32_2B[256]` (escape `0F`). Each entry holds the execute function pointer and an operand descriptor that drives the ModRM decoder.

---

## 11. SIMD / Vector Extension Architecture

```mermaid
graph TB
    subgraph REG_WIDTH["Register width progression"]
        MM["MMX  mm0–mm7\n64-bit integer\n(aliased to x87 ST regs)"]
        XMM["SSE  xmm0–xmm15\n128-bit float/int"]
        YMM["AVX  ymm0–ymm15\n256-bit float/int\n(upper 128 of XMM)"]
        ZMM["AVX-512  zmm0–zmm31\n512-bit float/int\n(upper 384 of YMM)"]
    end

    subgraph INSN_FAMILIES["Instruction families"]
        PACKED_FP["Packed float:\nADDPS/PD  MULPS/PD\nCMPPS/PD  SQRTPS/PD"]
        PACKED_INT["Packed integer:\nPADD* PSUB* PMUL*\nPCMP* PSHUF*"]
        CONVERT["Type convert:\nCVTSI2SS  CVTPS2PD\nVCVTPH2PS"]
        MASK_OPS["AVX-512 masking:\n{k1}  {k1}{z}\nVPANDQ  VCOMPRESSPS"]
        GATHER["Gather/scatter:\nVGATHERDPS\nVSCATTERQPD"]
    end

    MM --> XMM --> YMM --> ZMM
    XMM --- PACKED_FP
    XMM --- PACKED_INT
    YMM --- CONVERT
    ZMM --- MASK_OPS
    ZMM --- GATHER
```

In Bochs, all SIMD state is stored in `BX_CPU_C::xmm[]` (128-bit slots). AVX upper halves are stored in `ymm_hi128[]`. AVX-512 upper halves use `vmm[]`. The `avx/` subdirectory contains one `.cc` file per logical instruction group.

---

## 12. Virtual Machine Extensions (VMX/SVM)

```mermaid
stateDiagram-v2
    [*]         --> VMX_OFF : power-on
    VMX_OFF     --> VMX_ROOT : VMXON
    VMX_ROOT    --> VMX_NON_ROOT : VMLAUNCH / VMRESUME
    VMX_NON_ROOT --> VMX_ROOT : VM-exit\n(exception, I/O, CPUID, …)
    VMX_ROOT    --> VMX_OFF : VMXOFF

    VMX_NON_ROOT : Guest OS runs here
    VMX_ROOT     : Hypervisor (VMM) runs here
```

Bochs implements Intel VMX (`vmx.cc`) and AMD SVM (`svm.cc`). The **VMCS** (Virtual Machine Control Structure) holds per-VM state including:
- Guest / host register save areas
- VM-execution control fields (exit conditions)
- VM-exit / VM-entry information fields

On each VM-exit, Bochs copies guest state from the simulated VMCS into `BX_CPU_C` and begins executing the VMM's exit handler. This enables running nested hypervisors inside the emulator.

---

## 13. Neural-Tensor Architecture Counterpart

The table below maps every Bochs architectural concept to its LLM/neural-network analog, providing a precise structural correspondence:

```mermaid
graph LR
    subgraph Bochs["Bochs Architecture"]
        B1["Instruction bytes\n(raw token stream)"]
        B2["Decoder\n(byte → bxInstruction_c)"]
        B3["Dispatch table\n(opcode → handler fn)"]
        B4["Execute handler\n(transforms CPU state)"]
        B5["CPU state\n(registers + memory)"]
        B6["iCache\n(avoid re-decoding)"]
        B7["TLB\n(fast addr translation)"]
        B8["pc_system timers\n(scheduler)"]
    end

    subgraph Neural["Neural / LLM Architecture"]
        N1["Token ids\n(vocabulary indices)"]
        N2["Embedding layer\n(id → float vector)"]
        N3["Attention mechanism\n(token → relevant context)"]
        N4["MLP / FFN layer\n(transforms hidden state)"]
        N5["Weight tensors W\n(learned parameters)"]
        N6["KV-cache\n(avoid re-computing attention)"]
        N7["Sparse attention\n(fast context lookup)"]
        N8["Activation functions\n(ReLU/Softmax = policy gate)"]
    end

    B1 -.->|"≅"| N1
    B2 -.->|"≅"| N2
    B3 -.->|"≅"| N3
    B4 -.->|"≅"| N4
    B5 -.->|"≅"| N5
    B6 -.->|"≅"| N6
    B7 -.->|"≅"| N7
    B8 -.->|"≅"| N8
```

### Key insight: determinism vs. stochasticity

| Property | Bochs | LLM |
|---|---|---|
| State transitions | Deterministic (given input bytes) | Stochastic (temperature-sampled) |
| "Parameters" | Fixed silicon / ISA definition | Gradient-trained float tensors |
| "Program" | x86 machine code | Natural language prompt |
| Optimization | None at runtime (compile-time perf tuning) | Loss minimization during training |
| Correctness | Binary: matches silicon or not | Probabilistic: cross-entropy loss |

The bridge between these two worlds runs through the **activation function**: just as Bochs's `if (modrm.mod == 3)` selects a register vs. memory operand (a hard binary gate = ReLU), a neural network's softmax decides which "next computation" to invoke — a soft, differentiable version of the same dispatch mechanism.

---

## Further Reading

- [`overview.md`](./overview.md) — high-level system overview
- [`architecture-evolution-and-llm-insights.md`](./architecture-evolution-and-llm-insights.md) — conceptual evolution narrative
- [`formal-spec-z-plus-plus.md`](./formal-spec-z-plus-plus.md) — mathematical formal specification
- Bochs source: `bochs/cpu/cpu.h`, `bochs/cpu/decoder/`, `bochs/memory/memory.cc`
- Intel SDM Vol. 3A: *System Programming Guide*
