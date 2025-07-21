# PipelinedCPU
an embedded system cpu pipeline for a created instruction set made in Verilog
- `HudsonMODULES.txt`: Contains all core Verilog modules for the pipelined CPU, including:
  - ALU, control unit, register file, data memory, instruction memory
  - Pipeline registers (`IF/ID`, `ID/EX`, `EX/WB`)
  - Multiplexers, immediate generator, and PC logic

- `HudsonTB.txt`: The top-level testbench that integrates all modules to simulate the CPU. Handles:
  - Clock/reset logic
  - Branch/jump control muxes
  - Instruction flow through IF, ID, EX, MEM, WB stages
  - Final waveform verification for instruction execution

<img width="830" height="583" alt="Screenshot 2025-07-21 at 10 26 37 AM" src="https://github.com/user-attachments/assets/d9702dc3-0c22-4008-8bf1-d1fc6beaa60b" />

This project implements a 32-bit pipelined CPU in Verilog based on the SCU ISA. The CPU supports 12 instructions across a standard 5-stage pipeline: IF, ID, EX, MEM, WB. It was designed to execute assembly programs including a MIN function and vector addition.

Supported instructions: NOP, SVPC, LD, ST, ADD, INC, NEG, SUB, J, JM, BRZ, BRN

- 5-stage pipeline (IF, ID, EX, MEM, WB)
- Separate pipeline registers (IF/ID, ID/EX, EX/MEM, MEM/WB)
- Centralized control unit with truth table for signal management
- ALU with ADD, NEG, SUB operations and flag outputs (Z/N)
- - **Clock cycle time**: 4 ns (critical path through EX/MEM)
- **Theoretical CPI**: ~1.5
- **Measured CPI**: ~2.36 (due to unresolved bugs/stalls)
- **Clock frequency**: 250 MHz

- ALU
 <img width="510" height="516" alt="Screenshot 2025-07-21 at 10 29 49 AM" src="https://github.com/user-attachments/assets/1c42511c-b598-4ff2-9910-97f801fa0a4f" />

-verification
The min program finds the minimum element value from a given array, without using the specialized MIN instruction. The vector addition code uses a problem size n and the base addresses of arrays a, b, and c to compute the calculation:
(NOP,SVPC,LD,ST,ADD,INC,NEG,SUB,J,JM,BRZ,BRN)

Min Program:
//  0: SVPC x20, 60 ? x20 = PC+60 = 60  (exit/halt at data[60])
assign data[0]  = 32'b1111_010100_0000000000000000010000;
//  1: NOP
assign data[1]  = 32'b0000_000000_000000_000000_0000000000;

//  2: SVPC x21, 26 ? x21 = PC+26 = 28  (loop-test at data[28])
assign data[2]  = 32'b1111_010101_0000000000000000011001;
//  3: NOP
assign data[3]  = 32'b0000_000000_000000_000000_0000000000;

//  4: SVPC x22, 50 ? x22 = PC+50 = 54  (skip-update at data[54])
assign data[4]  = 32'b1111_010110_0000000000000000110001;
//  5: NOP
assign data[5]  = 32'b0000_000000_000000_000000_0000000000;

//  6: INC x8,  x0, 2  ? x8 = 2
assign data[6]  = 32'b0101_001000_000000_0000000000000010;
//  7: NOP
assign data[7]  = 32'b0000_000000_000000_000000_0000000000;

//  8: INC x7,  x0, 5  ? x7 = 5
assign data[8]  = 32'b0101_000111_000000_0000000000000101;
//  9: NOP
assign data[9]  = 32'b0000_000000_000000_000000_0000000000;

// 10: SUB x13, x7, x0 ? x13 = x7 - 0
assign data[10] = 32'b0111_001101_000111_000000_0000000000;
// 11: NOP
assign data[11] = 32'b0000_000000_000000_000000_0000000000;

// 12: NOP (stall for BRZ)
assign data[12] = 32'b0000_000000_000000_000000_0000000000;
// 13: NOP
assign data[13] = 32'b0000_000000_000000_000000_0000000000;

// 14: BRZ x20 ? if Z=1, jump to data[60]
assign data[14] = 32'b1001_000000_010100_000000_0000000000;
// 15: NOP
assign data[15] = 32'b0000_000000_000000_000000_0000000000;

// 16: NOP (branch delay)
assign data[16] = 32'b0000_000000_000000_000000_0000000000;
// 17: NOP
assign data[17] = 32'b0000_000000_000000_000000_0000000000;

// 18: LD  x10, x8     ? x10 = M[x8]
assign data[18] = 32'b1110_001010_001000_000000_0000000000;
// 19: NOP
assign data[19] = 32'b0000_000000_000000_000000_0000000000;

// 20: NOP (stall for load-use)
assign data[20] = 32'b0000_000000_000000_000000_0000000000;
// 21: NOP
assign data[21] = 32'b0000_000000_000000_000000_0000000000;

// 22: NOP (another stall)
assign data[22] = 32'b0000_000000_000000_000000_0000000000;
// 23: NOP
assign data[23] = 32'b0000_000000_000000_000000_0000000000;

// 24: INC x11, x0, 1  ? x11 = 1
assign data[24] = 32'b0101_001011_000000_0000000000000001;
// 25: NOP
assign data[25] = 32'b0000_000000_000000_000000_0000000000;

// 26: J   x21         ? jump to data[28] (loop-test)
//    (uses x21 loaded above)
assign data[26] = 32'b1000_000000_010101_000000_0000000000;
// 27: NOP
assign data[27] = 32'b0000_000000_000000_000000_0000000000;

// 28: SUB x13, x7, x11 ? x13 = N - i  (sets Z when i==N)
assign data[28] = 32'b0111_001101_000111_001011_0000000000;
// 29: NOP
assign data[29] = 32'b0000_000000_000000_000000_0000000000;

// 30: NOP (stall for BRZ)
assign data[30] = 32'b0000_000000_000000_000000_0000000000;
// 31: NOP
assign data[31] = 32'b0000_000000_000000_000000_0000000000;

// 32: BRZ x20 ? if i==N, jump to halt at data[60]
assign data[32] = 32'b1001_000000_010100_000000_0000000000;
// 33: NOP
assign data[33] = 32'b0000_000000_000000_000000_0000000000;

// 34: NOP (branch delay)
assign data[34] = 32'b0000_000000_000000_000000_0000000000;
// 35: NOP
assign data[35] = 32'b0000_000000_000000_000000_0000000000;

// 36: ADD x12, x8, x11 ? addr = base + i
assign data[36] = 32'b0100_001100_001000_001011_0000000000;
// 37: NOP
assign data[37] = 32'b0000_000000_000000_000000_0000000000;

// 38: LD  x1,  x12   ? x1 = M[addr]
assign data[38] = 32'b1110_000001_001100_000000_0000000000;
// 39: NOP
assign data[39] = 32'b0000_000000_000000_000000_0000000000;

// 40: NOP (stall for load-use of x1)
assign data[40] = 32'b0000_000000_000000_000000_0000000000;
// 41: NOP
assign data[41] = 32'b0000_000000_000000_000000_0000000000;

// 42: NOP (another stall)
assign data[42] = 32'b0000_000000_000000_000000_0000000000;
// 43: NOP
assign data[43] = 32'b0000_000000_000000_000000_0000000000;

// 44: SUB x13, x10, x1 ? x13 = min - element (sets N if negative)
assign data[44] = 32'b0111_001101_000001_001010_0000000000;
// 45: NOP
assign data[45] = 32'b0000_000000_000000_000000_0000000000;

// 46: NOP (stall for BRN)
assign data[46] = 32'b0000_000000_000000_000000_0000000000;
// 47: NOP
assign data[47] = 32'b0000_000000_000000_000000_0000000000;

// 48: BRN x22       ? if N=1, skip update (to data[54])
assign data[48] = 32'b1011_000000_010110_000000_0000000000;
// 49: NOP
assign data[49] = 32'b0000_000000_000000_000000_0000000000;

// 50: NOP (branch delay)
assign data[50] = 32'b0000_000000_000000_000000_0000000000;
// 51: NOP
assign data[51] = 32'b0000_000000_000000_000000_0000000000;

// 52: ADD x10, x1, x0 ? update min = element
assign data[52] = 32'b0100_001010_000001_000000_0000000000;
// 53: NOP
assign data[53] = 32'b0000_000000_000000_000000_0000000000;

// 54: J   x22         ? jump to data[54] skip-update entry
assign data[54] = 32'b1000_000000_010110_000000_0000000000;
// 55: NOP
assign data[55] = 32'b0000_000000_000000_000000_0000000000;

// 56: INC x11, x11, 1 ? i++
assign data[56] = 32'b0101_001011_001011_0000000000000001;
// 57: NOP
assign data[57] = 32'b0000_000000_000000_000000_0000000000;

// 58: J   x21         ? back to loop-test at data[28]
assign data[58] = 32'b1000_000000_010101_000000_0000000000;
// 59: NOP
assign data[59] = 32'b0000_000000_000000_000000_0000000000;

// 60: NOP (halt/end)
assign data[60] = 32'b0000_000000_000000_000000_0000000000;
Vector Addition Program:
// -- Add corresponding elements of two arrays, one by one --
assign data[0]  = 32'b1111_001000_000000_000000_0000000001; // SVPC x8, 1   → x8 = 1  (start addr of A)
assign data[1]  = 32'b1111_001001_000000_000000_0000000001; // SVPC x9, 1   → x9 = 2  (start addr of B)
assign data[2]  = 32'b1111_010011_000000_000000_0000011100; // SVPC x19,18  → x19 = 30 (end addr of B)
assign data[3]  = 32'b1111_001010_000000_000000_0000000011; // SVPC x10,3  → x10 = 6  (end addr of A)

// x20 will accumulate (A[i]+B[i]), so initialize it to zero:
assign data[4]  = 32'b0111_010100_001000_001000_0000000000; // SUB  x20, x8, x8  → x20 = 0
assign data[5]  = 32'b0000_000000_000000_000000_0000000000; // NOP
assign data[6]  = 32'b0000_000000_000000_000000_0000000000; // NOP

// load A[i] and B[i]:
assign data[7]  = 32'b1110_000110_001001_000000_0000000000; // LD   x6, x9
assign data[8]  = 32'b0000_000000_000000_000000_0000000000; // NOP
assign data[9]  = 32'b0000_000000_000000_000000_0000000000; // NOP
assign data[10] = 32'b1110_000111_010011_000000_0000000000; // LD   x7, x19
assign data[11] = 32'b0000_000000_000000_000000_0000000000; // NOP
assign data[12] = 32'b0000_000000_000000_000000_0000000000; // NOP

// sum = A[i]+B[i]:
assign data[13] = 32'b0100_010100_000111_000110_0000000000; // ADD  x20, x7, x6
assign data[14] = 32'b0000_000000_000000_000000_0000000000; // NOP
assign data[15] = 32'b0000_000000_000000_000000_0000000000; // NOP

// store the result back to memory at A’s location:
assign data[16] = 32'b0011_000000_001001_010100_0000000000; // ST   x9, x20
assign data[17] = 32'b0000_000000_000000_000000_0000000000; // NOP
assign data[18] = 32'b0000_000000_000000_000000_0000000000; // NOP

// reload for a check (optional):
assign data[19] = 32'b1110_000110_001001_000000_0000000000; // LD   x6, x9
assign data[20] = 32'b0000_000000_000000_000000_0000000000; // NOP
assign data[21] = 32'b0000_000000_000000_000000_0000000000; // NOP

// advance both pointers:
assign data[22] = 32'b0101_001001_001001_000000_0000000001; // INC  x9, x9, 1
assign data[23] = 32'b0101_010011_010011_000000_0000000001; // INC  x19, x19, 1

// loop‐termination test (if B_ptr − B_end < 0, branch):
assign data[24] = 32'b0111_001011_001001_001010_0000000000; // SUB  x11, x9, x10
assign data[25] = 32'b0000_000000_000000_000000_0000000000; // NOP
assign data[26] = 32'b1011_000000_001010_000000_0000000000; // BRN  x10
assign data[27] = 32'b0000_000000_000000_000000_0000000000; // NOP
assign data[28] = 32'b0000_000000_000000_000000_0000000000; // NOP
assign data[29] = 32'b0000_000000_000000_000000_0000000000; // NOP

