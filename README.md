# Class 03: Memory & Program Counter

> **Week 03 | Hanyang University ERICA Campus | Department of Robotics**  
> **Computer Architecture Course**

---

## 📚 Learning Objectives

After completing this class, you will be able to:

1. **Understand the von Neumann architecture**: Programs and data are stored in memory
2. **Implement Instruction Memory**: Store machine code for CPU execution
3. **Implement Data Memory**: Support `lw` and `sw` instructions
4. **Design the Program Counter (PC)**: Control sequential instruction execution
5. **Complete the IF Stage**: Combine PC and instruction memory into a complete fetch unit

---

## 🧠 Key Concepts

### Von Neumann Architecture Basics

```
  ┌──────────────────────────────────────────────────┐
  │                    CPU                           │
  │  ┌────────┐    ┌────────┐    ┌────────────────┐  │
  │  │   PC   │───→│ Instr  │───→│   Datapath     │  │
  │  │        │    │ Memory │    │ (ALU, RegFile) │  │
  │  └────────┘    └────────┘    └───────┬────────┘  │
  │                                      │           │
  │                              ┌───────↓───────┐   │
  │                              │  Data Memory  │   │
  │                              └───────────────┘   │
  └──────────────────────────────────────────────────┘
```

- **Program Counter (PC)**: Points to the address of the current instruction
- **Instruction Memory**: Read-only memory storing program machine code
- **Data Memory**: Read/write memory storing variables and data

### Why Two Separate Memories?

| Memory Type | Access Type | Content | Stage |
|-------------|-------------|---------|-------|
| Instruction Memory | Read-only | Machine code (.text) | IF Stage |
| Data Memory | Read/Write | Variables, arrays (.data) | MEM Stage |

> Note: This is a variant of the **Harvard architecture**. True von Neumann uses unified memory, but modern CPUs typically separate them for efficiency.

---

## 📁 File Structure

```
class_03/
├── pc.v                    # Program counter
├── instruction_memory.v    # Instruction memory
├── if_stage.v              # IF stage wrapper module
├── if_stage_tb.v           # IF stage testbench
├── memfile.dat             # Machine code data file
└── Makefile
```

---

## 💻 Code Walkthrough

### 1. `pc.v` - Program Counter

```verilog
module pc (
    input  wire        clk,
    input  wire        rst_n,
    input  wire        en,        // Enable signal (for stalling)
    input  wire [31:0] pc_next,   // Next instruction address
    output reg  [31:0] pc         // Current instruction address
);
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            pc <= 32'h00000000;   // Start from address 0 on reset
        else if (en)
            pc <= pc_next;        // Update PC
    end
endmodule
```

**Design points**:
- **Asynchronous reset (`negedge rst_n`)**: Active-low reset ensures reliable system startup
- **Enable control (`en`)**: Keeps PC unchanged when pipeline needs to stall
- **PC+4**: In the fetch stage, `pc_next` is typically connected to `pc + 4` (next instruction)

### 2. `instruction_memory.v` - Instruction Memory

```verilog
module instruction_memory (
    input  wire [31:0] addr,
    output wire [31:0] rd
);
    // 256 × 32-bit = 1KB instruction space
    reg [31:0] ROM [255:0];

    // Load machine code from file
    initial begin
        $readmemh("memfile.dat", ROM);
    end

    // Asynchronous read (combinational logic)
    // addr is byte address, divide by 4 to get word index
    assign rd = ROM[addr[31:2]];
endmodule
```

**Design points**:
- **`$readmemh`**: Verilog system function to load hex data from file
- **`addr[31:2]`**: MIPS instructions are 4-byte aligned, so use upper 30 bits as index
- **ROM**: Read-only memory, no write functionality needed

### 3. `memfile.dat` - Machine Code File Format

```
20020005   // addi $v0, $zero, 5
20030003   // addi $v1, $zero, 3
00622020   // add $a0, $v1, $v0
```

One 32-bit hex number per line, representing one MIPS instruction.

### 4. `if_stage.v` - IF Stage Wrapper

```verilog
module if_stage (
    input  wire        clk,
    input  wire        rst_n,
    output wire [31:0] instr,      // Fetched instruction
    output wire [31:0] current_pc  // Current PC value
);
    wire [31:0] pc_plus4;

    pc u_pc (
        .clk(clk), .rst_n(rst_n), .en(1'b1),
        .pc_next(pc_plus4), .pc(current_pc)
    );

    instruction_memory u_imem (
        .addr(current_pc), .rd(instr)
    );

    assign pc_plus4 = current_pc + 32'd4;
endmodule
```

**Dataflow**:
```
   PC ──→ Instruction Memory ──→ instr
    ↑           │
    └───── +4 ──┘
```

---

## 🎯 Design Highlights

### The Meaning of PC+4

MIPS uses **fixed-length instruction format** with each instruction being 32 bits (4 bytes).
- Sequential execution: `PC_next = PC + 4`
- Branch/Jump: `PC_next = BranchTarget` or `JumpTarget`

### Byte Address vs Word Address

```
Memory view (byte address):
  [0x00] [0x01] [0x02] [0x03] | [0x04] [0x05] [0x06] [0x07] | ...
  <-------- Instr 0 --------> | <-------- Instr 1 --------> |

Our design (word address):
  ROM[0] = Instruction 0
  ROM[1] = Instruction 1
```

So `addr[31:2]` converts byte address to word address (divide by 4).

---

## 🧪 Lab Exercise

### Step 1: Write a test program
Modify `memfile.dat` with these test instructions:
```
20080005   // addi $t0, $zero, 5
20090003   // addi $t1, $zero, 3
01094020   // add $t0, $t0, $t1  (result should be 8)
```

### Step 2: Run simulation
```bash
cd class_03
make
```

### Step 3: Observe waveform
- Verify `pc` increments by 4 each cycle
- Verify `instr` output matches `memfile.dat`

---

## 🔍 Think Deeper

### Question 1: Why is instruction memory read asynchronous?

> **Hint**: If reading required one cycle, the fetch stage would need 2 cycles. How would this affect the pipeline?

### Question 2: What happens if the program jumps to a non-aligned address?

MIPS requires all instructions to be 4-byte aligned. If PC = 0x05 (not aligned), how would hardware handle this?

### Question 3: Is `$readmemh` simulation-only?

> **Hint**: On a real FPGA, what methods are used to initialize ROM?

---

## 📖 Further Reading

- **Harvard vs von Neumann**: Pros and cons of two computer architectures
- **Block RAM (BRAM)**: Memory implementation on FPGAs
- **Memory-Mapped I/O**: Later classes will cover using address space to control peripherals

---

## ✅ Checkpoint

Before moving to the next class, make sure you can answer:

- [ ] Is PC sequential or combinational logic?
- [ ] Does instruction memory require a clock?
- [ ] Why does the address need to be right-shifted by 2 bits?

---

**Previous**: [Class 02 - Register File](../class_02/README.md)  
**Next**: [Class 04 - Single-Cycle Datapath](../class_04/README.md)
