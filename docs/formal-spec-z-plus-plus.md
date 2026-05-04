# x86ochs — Formal Specification in Z++

> A mathematically rigorous specification of the x86ochs / Bochs emulator core and its neural-tensor counterpart, written in **Z++** — the object-oriented extension of the Z formal notation (ISO/IEC 13568).
>
> Z++ augments standard Z schemas with **class**, **state**, **op**, and **inherit** constructs. All schemas follow the convention:
> - Decorated names (e.g., `state'`) denote post-state values.
> - `Δ` prefix denotes an operation that may change state.
> - `Ξ` prefix denotes a query that leaves state unchanged.
> - `ℕ`, `ℤ`, `ℝ`, `𝔹` denote naturals, integers, reals, and booleans respectively.

---

## Table of Contents

1. [Basic Types & Domains](#1-basic-types--domains)
2. [Memory Model](#2-memory-model)
3. [CPU Register File](#3-cpu-register-file)
4. [Instruction Representation](#4-instruction-representation)
5. [CPU Execution State Machine](#5-cpu-execution-state-machine)
6. [Decode Operation](#6-decode-operation)
7. [Execute Operation](#7-execute-operation)
8. [Paging & Address Translation](#8-paging--address-translation)
9. [Interrupt & Exception Model](#9-interrupt--exception-model)
10. [Timer Scheduling Model](#10-timer-scheduling-model)
11. [Neural Tensor Formal Model](#11-neural-tensor-formal-model)
12. [Structural Correspondence Theorem](#12-structural-correspondence-theorem)

---

## 1. Basic Types & Domains

```z++
-- Primitive width types
[BYTE]   == {b : ℕ | 0 ≤ b < 256}
[WORD]   == {w : ℕ | 0 ≤ w < 65536}
[DWORD]  == {d : ℕ | 0 ≤ d < 2^32}
[QWORD]  == {q : ℕ | 0 ≤ q < 2^64}

-- Address spaces
PHYS_ADDR == {a : ℕ | 0 ≤ a < 2^52}  -- Physical (52-bit max)
LIN_ADDR  == {a : ℕ | 0 ≤ a < 2^64}  -- Linear (virtual after segmentation)
IO_PORT   == {p : ℕ | 0 ≤ p < 2^16}  -- 64 K I/O ports

-- Opcode space
OPCODE    == {o : ℕ | 0 ≤ o < 2^24}  -- primary + escape byte + ModRM
VECTOR    == {v : ℕ | 0 ≤ v < 256}   -- IDT vector index

-- ISA mode
ISA_MODE  ::= REAL_16 | PROTECTED_32 | LONG_64

-- Memory type
MTYPE     ::= WB | WT | WC | UC | WP  -- Write-Back, Write-Through, …
```

---

## 2. Memory Model

```z++
class PhysicalMemory
  state
    ram      : PHYS_ADDR ⇸ BYTE        -- partial function: only mapped addrs
    rom      : PHYS_ADDR ⇸ BYTE        -- read-only; disjoint from ram
    mmio     : PHYS_ADDR ⇸ BYTE        -- memory-mapped I/O
    mtrr_map : PHYS_ADDR ⇸ MTYPE       -- memory type per page frame

  invariant
    dom ram ∩ dom rom  = ∅
    dom ram ∩ dom mmio = ∅
    dom rom ∩ dom mmio = ∅

  op ΔReadByte
    addr?   : PHYS_ADDR
    data!   : BYTE
  where
    addr? ∈ dom ram  ⟹ data! = ram(addr?)
    addr? ∈ dom rom  ⟹ data! = rom(addr?)
    addr? ∈ dom mmio ⟹ data! = mmio(addr?)  -- triggers device callback

  op ΔWriteByte
    addr?   : PHYS_ADDR
    data?   : BYTE
  where
    addr? ∈ dom ram  ⟹ ram'  = ram  ⊕ {addr? ↦ data?}
    addr? ∈ dom mmio ⟹ mmio' = mmio ⊕ {addr? ↦ data?}  -- device callback
    addr? ∈ dom rom  ⟹ ram'  = ram   -- write to ROM: silently ignored
```

---

## 3. CPU Register File

```z++
class RegisterFile
  state
    -- General-purpose registers (64-bit mode)
    gpr     : {0..15} → QWORD          -- R0=RAX … R15

    -- Instruction pointer
    rip     : QWORD

    -- EFLAGS
    cf, pf, af, zf, sf, tf, if_flag, df, of : 𝔹
    iopl    : {0..3}

    -- Segment registers (selector + cached descriptor)
    cs, ds, es, fs, gs, ss : SegRegister

    -- Control registers
    cr0     : DWORD
    cr2     : QWORD   -- page-fault linear address
    cr3     : QWORD   -- page-table base
    cr4     : DWORD

    -- SIMD registers
    zmm     : {0..31} → (seq₅₁₂ 𝔹)   -- 512-bit ZMM (AVX-512)
    kmask   : {0..7}  → QWORD          -- opmask registers

    -- x87 FPU stack
    fpu_st  : {0..7}  → FloatExt80     -- 80-bit extended precision
    fpu_top : {0..7}                    -- stack top pointer
    fpu_sw  : WORD                      -- status word
    fpu_cw  : WORD                      -- control word
    fpu_tw  : WORD                      -- tag word

  invariant
    rip < 2^64
    fpu_top ∈ {0..7}

class SegRegister
  state
    selector : WORD
    base     : QWORD
    limit    : DWORD
    ar       : BYTE       -- access rights
    valid    : 𝔹

class FloatExt80
  state
    sign     : 𝔹
    exponent : {0..32767}
    mantissa : {0..2^63 - 1}
```

---

## 4. Instruction Representation

```z++
-- Addressing mode for an operand
AddrMode ::= REG_DIRECT    -- operand is a register
           | MEM_DIRECT    -- operand is a memory address (no base/index)
           | MEM_BASE_DISP -- operand is base_reg + displacement
           | MEM_SIB       -- operand is base + index*scale + disp
           | IMMEDIATE     -- operand is an inline constant

class Operand
  state
    mode    : AddrMode
    regnum  : {0..31}     -- valid when mode = REG_DIRECT
    lin_addr: LIN_ADDR    -- valid when mode ∈ {MEM_DIRECT, …}
    imm_val : QWORD       -- valid when mode = IMMEDIATE
    width   : {8,16,32,64,128,256,512}  -- operand width in bits

class DecodedInsn
  state
    opcode  : OPCODE
    src     : Operand
    dst     : Operand
    ilen    : {1..15}     -- instruction length in bytes
    rep     : 𝔹           -- REP/REPE/REPNE prefix
    lock    : 𝔹           -- LOCK prefix
    seg_ovr : {0..5} ∪ {⊥}  -- segment override (⊥ = none)
    -- execute function pointer represented as opcode index
    exec_id : OPCODE
```

---

## 5. CPU Execution State Machine

```z++
CPUMode ::= HALTED | RUNNING | SMM | VMX_ROOT | VMX_NON_ROOT

class CPU
  inherit RegisterFile
  inherit PhysicalMemory

  state
    mode        : CPUMode
    tick_count  : ℕ           -- monotonically increasing CPU tick counter
    pending_irq : 𝔹
    pending_nmi : 𝔹
    pending_smi : 𝔹

  invariant
    mode = HALTED ⟹ ¬pending_irq ∨ ¬if_flag

  -- Single-step transition relation
  op ΔStep
  where
    mode = RUNNING
    ∧ (pending_smi  ⟹ ΔEnterSMM)
    ∧ (¬pending_smi ∧ pending_nmi ⟹ ΔDeliverNMI)
    ∧ (¬pending_smi ∧ ¬pending_nmi
       ∧ pending_irq ∧ if_flag ⟹ ΔDeliverIRQ)
    ∧ (¬pending_smi ∧ ¬pending_nmi
       ∧ ¬(pending_irq ∧ if_flag) ⟹ ΔFetchDecodeExecute)
    tick_count' = tick_count + 1

  -- Normal instruction execution
  op ΔFetchDecodeExecute
    insn! : DecodedInsn
  where
    ΔFetch(insn!)
    ΔExecute(insn!)
    rip' = rip + insn!.ilen

  -- Fetch: read instruction bytes from memory
  op ΔFetch
    insn! : DecodedInsn
  where
    ∃ bytes : seq BYTE •
        #bytes ≤ 15
        ∧ (∀ i : dom bytes • bytes(i) = ram(lin_to_phys(cs.base + rip + i)))
        ∧ ΔDecode(bytes, insn!)

  -- Halt: CPU stops until interrupt
  op ΔHalt
  where
    mode' = HALTED

  op ΔWakeFromHalt
  where
    mode = HALTED
    ∧ (pending_irq ∨ pending_nmi ∨ pending_smi)
    mode' = RUNNING
```

---

## 6. Decode Operation

```z++
class Decoder

  -- Top-level decode: bytes → DecodedInsn
  op ΔDecode
    raw     : seq BYTE
    insn!   : DecodedInsn
  where
    #raw ≥ 1  ∧  #raw ≤ 15
    ∧ ∃ pfx_len  : ℕ •
        pfx_len = CountPrefixes(raw)
        ∧ ∃ op_len : {1,2,3} •
            op_len = OpcodeLength(raw, pfx_len)
            ∧ insn!.opcode  = ExtractOpcode(raw, pfx_len, op_len)
            ∧ insn!.ilen    = pfx_len + op_len + ModRMLength(insn!.opcode) + ImmLength(insn!.opcode)
            ∧ ∀ field ∈ {src, dst} •
                DecodeOperand(insn!.opcode, raw, pfx_len + op_len, field, insn!.field)

  -- Prefix counting: REX, VEX, EVEX, segment overrides, …
  CountPrefixes : seq BYTE → ℕ
  CountPrefixes raw == #{ i : dom raw | raw(i) ∈ PREFIX_BYTES }

  -- Opcode byte length: 1 (normal), 2 (0F xx), 3 (0F 38/3A xx)
  OpcodeLength : seq BYTE × ℕ → {1,2,3}
  OpcodeLength(raw, start) ==
    if raw(start) = 0x0F
    then if raw(start+1) ∈ {0x38, 0x3A}
         then 3
         else 2
    else 1
```

---

## 7. Execute Operation

The execute operation is a *family* of partial functions, one per opcode family:

```z++
-- Execution result type
ExecResult ::= OK | FAULT(vector : VECTOR, error : DWORD)

-- Generic execute dispatch
op ΔExecute
  insn? : DecodedInsn
  result! : ExecResult
where
  result! = ExecuteTable(insn?.opcode)(cpu_state, insn?)

-- Example: ADD r/m32, r32   (opcode 0x01)
op ΔExecADD32
  insn? : DecodedInsn
  pre:  insn?.opcode = 0x01
where
  let a == ReadOperand32(insn?.src)
      b == ReadOperand32(insn?.dst)
      sum == (a + b) mod 2^32
  in  WriteOperand32(insn?.dst, sum)
      ∧ cf'   = (a + b ≥ 2^32)
      ∧ of'   = ((a[31] = b[31]) ∧ (a[31] ≠ sum[31]))
      ∧ sf'   = sum[31]
      ∧ zf'   = (sum = 0)
      ∧ pf'   = ParityBit(sum[7:0])
      ∧ af'   = ((a[3:0] + b[3:0]) ≥ 16)

-- Example: VADDPS ymm, ymm, ymm   (AVX packed single-precision add)
op ΔExecVADDPS
  insn? : DecodedInsn
  pre:  insn?.opcode ∈ VADDPS_OPCODES
where
  let src1 == zmm(insn?.src.regnum)[255:0]   -- lower 256 bits
      src2 == zmm(insn?.dst.regnum)[255:0]
      result == [FloatAdd32(src1[32i+31 : 32i], src2[32i+31 : 32i]) | i ∈ {0..7}]
  in  zmm'(insn?.dst.regnum) = ZeroExtend512(result)
```

---

## 8. Paging & Address Translation

```z++
-- Page Table Entry (64-bit long mode)
class PTE
  state
    present  : 𝔹
    writable : 𝔹
    user     : 𝔹
    nx       : 𝔹         -- execute-disable (NX bit, requires EFER.NXE)
    pfn      : PHYS_ADDR  -- page frame number (bits 51:12)
    pat      : {0..7}     -- page attribute table index

-- Four-level page walk (IA-32e / long mode)
PageWalk : LIN_ADDR × PHYS_ADDR → PHYS_ADDR ∪ {PAGE_FAULT}
PageWalk(la, cr3_base) ==
  let pml4_idx == la[47:39]
      pdpt_idx == la[38:30]
      pd_idx   == la[29:21]
      pt_idx   == la[20:12]
      offset   == la[11:0]
      pml4e    == ReadPhys64(cr3_base + pml4_idx × 8)
      pdpte    == ReadPhys64(PFN(pml4e) + pdpt_idx × 8)
      pde      == ReadPhys64(PFN(pdpte) + pd_idx   × 8)
      pte      == ReadPhys64(PFN(pde)   + pt_idx   × 8)
  in  if ¬P(pml4e) ∨ ¬P(pdpte) ∨ ¬P(pde) ∨ ¬P(pte)
      then PAGE_FAULT
      else PFN(pte) × 4096 + offset

-- TLB as a partial function from linear to physical address
class TLB
  state
    entries : LIN_ADDR ⇸ PHYS_ADDR   -- 4096-entry direct-mapped (modelled as ⇸)
    global  : LIN_ADDR ⇸ PHYS_ADDR   -- global pages (not flushed by CR3 write)

  op ΞLookup
    la?   : LIN_ADDR
    pa!   : PHYS_ADDR ∪ {MISS}
  where
    pa! = if la? ∈ dom entries then entries(la?) else MISS

  op ΔFlushAll
  where
    entries' = ∅

  op ΔFlushNonGlobal
  where
    entries' = global ▷ dom global
```

---

## 9. Interrupt & Exception Model

```z++
-- IDT Gate types
GateType ::= INTERRUPT_GATE | TRAP_GATE | TASK_GATE

class IDTGate
  state
    gtype    : GateType
    selector : WORD       -- target code segment
    offset   : QWORD      -- handler entry point (RIP)
    dpl      : {0..3}     -- descriptor privilege level
    present  : 𝔹

-- IDT: 256 gates
IDT == {0..255} → IDTGate

-- Interrupt delivery precondition
DeliverInterrupt : CPU × VECTOR → 𝔹
DeliverInterrupt(cpu, v) ==
  let gate == cpu.idt(v)
  in  gate.present
      ∧ (gate.gtype ∈ {INTERRUPT_GATE, TRAP_GATE})
      ∧ cpu.mode ≠ HALTED

-- Exception delivery state transition
op ΔDeliverException
  v?   : VECTOR
  err? : DWORD ∪ {⊥}     -- ⊥ = no error code pushed
where
  DeliverInterrupt(self, v?)
  let gate == idt(v?)
  in  -- Save current context onto guest stack
      rsp' = rsp - (if err? = ⊥ then 24 else 32)  -- 3 or 4 QWORDs
      ∧ Write64(rsp', rflags)
      ∧ Write64(rsp' + 8, cs.selector ++ rip)
      ∧ (err? ≠ ⊥ ⟹ Write64(rsp' + 24, err?))
      -- Load handler
      ∧ rip'  = gate.offset
      ∧ cs'   = LoadCSDescriptor(gate.selector)
      -- For INTERRUPT_GATE: clear IF; for TRAP_GATE: preserve IF
      ∧ if_flag' = (if gate.gtype = INTERRUPT_GATE then false else if_flag)
```

---

## 10. Timer Scheduling Model

```z++
-- A timer slot (one of BX_MAX_TIMERS = 64)
class TimerSlot
  state
    in_use      : 𝔹
    active      : 𝔹
    continuous  : 𝔹
    period      : ℕ           -- fire interval in CPU ticks
    time_to_fire: ℕ           -- absolute tick to fire
    handler_id  : ℕ           -- index into device callback table
    param       : DWORD

-- PC System: the timer multiplexer
class PCSystem
  state
    timers      : {0..63} → TimerSlot
    tick_count  : ℕ
    num_timers  : ℕ

  invariant
    num_timers ≤ 64
    ∀ i : {0..63} •
      ¬timers(i).in_use ⟹ ¬timers(i).active

  -- Advance by n ticks: fire all expired timers
  op ΔTickN
    n? : ℕ
  where
    tick_count' = tick_count + n?
    ∧ ∀ i : {0..63} •
        timers(i).active ∧ timers(i).time_to_fire ≤ tick_count'
        ⟹  FireCallback(timers(i).handler_id, timers(i).param)
            ∧ (timers(i).continuous
               ⟹ timers'(i).time_to_fire = timers(i).time_to_fire + timers(i).period)
            ∧ (¬timers(i).continuous
               ⟹ timers'(i).active = false)

  -- Register a new timer
  op ΔRegisterTimer
    period?   : ℕ
    handler?  : ℕ
    cont?     : 𝔹
    slot!     : {0..63}
  where
    ∃ i : {0..63} • ¬timers(i).in_use ∧ slot! = i
    timers'(slot!) = ⟨ in_use      ↦ true,
                       active      ↦ true,
                       continuous  ↦ cont?,
                       period      ↦ period?,
                       time_to_fire↦ tick_count + period?,
                       handler_id  ↦ handler?,
                       param       ↦ 0 ⟩
```

---

## 11. Neural Tensor Formal Model

The following formally specifies the transformer-based LLM architecture in Z++, using the same structural conventions as the CPU model above.

### 11.1 Basic Neural Types

```z++
-- A real-valued tensor: rank r, shape s
Tensor{r : ℕ} == seq^r ℝ         -- r-dimensional real array

-- Vocabulary and token types
[VOCAB]   -- finite set of vocabulary tokens
[TOKEN]   == {t : ℕ | 0 ≤ t < |VOCAB|}
[DIM]     -- embedding / hidden dimension ℕ
[LAYERS]  -- number of transformer layers ℕ

-- Activation function type: ℝ → ℝ
ActivationFn == ℝ → ℝ
```

### 11.2 Embedding Layer

```z++
class EmbeddingLayer
  state
    W_emb  : TOKEN → Tensor{1}     -- lookup table: token id → d-dim vector
    d_model: DIM                    -- embedding dimension

  invariant
    ∀ t : TOKEN • #W_emb(t) = d_model

  op ΞEmbed
    tokens? : seq TOKEN
    embeds! : seq Tensor{1}
  where
    embeds! = [W_emb(tokens?(i)) | i ← 1..|tokens?|]
```

### 11.3 Attention Mechanism

```z++
class MultiHeadAttention
  state
    W_Q, W_K, W_V : Tensor{2}   -- (d_model × d_model) weight matrices
    W_O            : Tensor{2}
    n_heads        : ℕ
    d_head         : DIM         -- d_model / n_heads

  -- Scaled dot-product attention (single head)
  ScaledDotProduct : Tensor{2} × Tensor{2} × Tensor{2} → Tensor{2}
  ScaledDotProduct(Q, K, V) ==
    let scores == Q ⊗ Kᵀ / √d_head   -- matrix multiply + scale
        weights == Softmax(scores)     -- row-wise softmax
    in  weights ⊗ V

  op ΞForward
    X?     : seq Tensor{1}         -- input sequence (seq_len × d_model)
    output!: seq Tensor{1}
  where
    let Q == X? ⊗ W_Q
        K == X? ⊗ W_K
        V == X? ⊗ W_V
        heads == [ScaledDotProduct(HeadSlice(Q,i), HeadSlice(K,i), HeadSlice(V,i))
                  | i ← 1..n_heads]
        concat == Concatenate(heads)
    in  output! = concat ⊗ W_O
```

### 11.4 Feed-Forward Network (MLP)

```z++
class FeedForward
  state
    W1      : Tensor{2}        -- (d_model × d_ff)
    W2      : Tensor{2}        -- (d_ff × d_model)
    b1, b2  : Tensor{1}        -- bias vectors
    phi     : ActivationFn     -- e.g., GeLU, ReLU, SwiGLU

  invariant
    -- phi is the policy gate: determines which features "fire"
    ∀ x : ℝ • phi(x) ∈ ℝ  -- totality

  op ΞForward
    x?  : Tensor{1}
    y!  : Tensor{1}
  where
    y! = W2 ⊗ phi(W1 ⊗ x? + b1) + b2
```

### 11.5 Transformer Block

```z++
class TransformerBlock
  state
    attn  : MultiHeadAttention
    ffn   : FeedForward
    ln1   : LayerNorm
    ln2   : LayerNorm

  op ΞForward
    X?    : seq Tensor{1}
    X'!   : seq Tensor{1}
  where
    let A    == X? + attn.ΞForward(ln1.ΞNormalize(X?))  -- residual + attn
        X'!  == A  + ffn.ΞForward(ln2.ΞNormalize(A))    -- residual + FFN
```

### 11.6 Full LLM (Autoregressive Transformer)

```z++
class LLM
  state
    embed  : EmbeddingLayer
    blocks : seq TransformerBlock           -- N layers
    ln_f   : LayerNorm                      -- final layer norm
    W_out  : Tensor{2}                      -- (d_model × |VOCAB|) unembedding
    kv_cache : seq (Tensor{2} × Tensor{2}) -- K/V cache per layer

  -- Forward pass: tokens → logits over vocabulary
  op ΞForward
    ctx?    : seq TOKEN
    logits! : Tensor{1}                     -- |VOCAB|-dim log-probability vector
  where
    let H_0  == embed.ΞEmbed(ctx?)
        H_n  == fold(blocks, H_0,
                  λ h b → b.ΞForward(h))   -- apply each block in sequence
        H_f  == ln_f.ΞNormalize(H_n)
        logits! == H_f[last] ⊗ W_out       -- project last token to vocab

  -- Sampling: logits → next token (stochastic)
  op ΔSample
    logits? : Tensor{1}
    temp?   : ℝ                             -- temperature ∈ (0, ∞)
    token!  : TOKEN
  where
    let probs == Softmax(logits? / temp?)
    token! ∈ TOKEN
    Pr(token! = t) = probs(t)              -- probabilistic selection

  -- Training step: update weights to minimise cross-entropy loss
  op ΔTrainStep
    batch?  : seq (seq TOKEN × TOKEN)      -- (context, target) pairs
    lr?     : ℝ                            -- learning rate
  where
    let loss == mean[CrossEntropy(ΞForward(ctx).logits, tgt)
                     | (ctx, tgt) ← batch?]
        grads == ∂loss / ∂(W_out, blocks, embed)   -- backpropagation
    in  W_out'   = W_out   - lr? × grads.W_out
        ∧ blocks' = blocks - lr? × grads.blocks
        ∧ embed'  = embed  - lr? × grads.embed
```

---

## 12. Structural Correspondence Theorem

Having specified both systems formally, we can state the structural isomorphism precisely:

```z++
-- The correspondence is a simulation relation between CPU and LLM
SimRel : CPU × LLM → 𝔹
SimRel(cpu, llm) ==
  ∃ f_state : CPUState → Tensor{1}  •   -- state embedding function
  ∃ f_input : BYTE → TOKEN          •   -- input codec
  ∃ f_output: BYTE → TOKEN          •
    -- Every CPU step corresponds to an LLM forward pass
    (∀ insn : DecodedInsn •
       let s  == f_state(cpu.state_before)
           t  == f_input(insn.opcode)
           s' == llm.ΞForward(t ++ s).logits    -- "next state" prediction
       in  f_state(cpu.state_after) ≈_ε s')     -- ε-approximation

-- Convergence: with sufficient training data and capacity,
-- the LLM can simulate the CPU to arbitrary precision
ConvergenceTheorem ==
  ∀ cpu : CPU • ∀ ε : ℝ | ε > 0 •
    ∃ llm : LLM | llm.blocks > N_min(ε) •
      ∀ insn_trace : seq DecodedInsn •
        SimRel(cpu, llm)
        ⟹ ‖f_state(CPUTrace(cpu, insn_trace).final_state)
           - LLMTrace(llm, f_input(insn_trace)).final_hidden‖ < ε
```

**Informal reading:**  
Given enough transformer layers and training data, an LLM can approximate the CPU execution function to any desired precision. The activation function `phi` in each `FeedForward` block corresponds to the opcode dispatch gate in the CPU's execute pipeline. The KV-cache corresponds to the iCache. The weight tensors `W_Q, W_K, W_V` correspond to the CPU's decode tables.

The key asymmetry: the CPU's tables are *fixed by Intel*; the LLM's weights are *learned from data*. Both are universal computation substrates. The LLM's training process is the gradient-descent analog of Intel's chip design process.

---

## Notation Reference

| Symbol | Meaning |
|---|---|
| `⇸` | Partial function |
| `→` | Total function |
| `↦` | Maplet (single-pair function) |
| `⊕` | Function override |
| `⊗` | Matrix / tensor multiplication |
| `▷` | Domain restriction |
| `dom` | Domain of a function |
| `seq` | Finite sequence type |
| `fold` | Left fold over a sequence |
| `∂/∂` | Partial derivative (backpropagation) |
| `≈_ε` | Approximate equality within ε |
| `‖·‖` | Vector / tensor norm |
| `Pr(·)` | Probability measure |

---

## References

- **Z Notation**: Spivey, J.M. *The Z Notation: A Reference Manual*, 2nd ed., 1992
- **Z++**: Lano, K. & Haughton, H. *Object-Oriented Specification Case Studies*, 1994
- **ISO/IEC 13568**: *Information Technology — Z Formal Specification Notation*, 2002
- **Attention Is All You Need**: Vaswani et al., 2017 (arXiv:1706.03762)
- **Intel SDM**: *Intel 64 and IA-32 Architectures Software Developer's Manual*, Vol. 1–3
- **Bochs source**: `bochs/cpu/cpu.h`, `bochs/cpu/decoder/`, `bochs/memory/memory.cc`
