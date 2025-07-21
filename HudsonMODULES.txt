`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
// Company: 
// Engineer: 
// 
// Create Date: 06/05/2025 02:36:53 PM
// Design Name: 
// Module Name: Main
// Project Name: 
// Target Devices: 
// Tool Versions: 
// Description: 
// 
// Dependencies: 
// 
// Revision:
// Revision 0.01 - File Created
// Additional Comments:
// 
//////////////////////////////////////////////////////////////////////////////////
// I divided each stage into a module and wrote skeleton code with it...



//MAJOR BLOCKS



module datam(clk, read, wr_enable, address_in, data_in, data_out);
    input clk, read, wr_enable;
    input [31:0] address_in, data_in;
    output reg [31:0] data_out;
    reg [31:0] data[65535:0];
    initial begin
         data[1] = 32'd2; //setting data memory
         data[2] = 32'd31;
         data[3] = 32'd1024;
         data[4] = 32'd9;
         data[5] = -32'd2048;
         data[6] = 32'd10;
         
         data[13] = 32'd23;
         data[14] = 32'd2;
         data[15] = 32'd16;
         data[16] = 32'd14;
         data[11] = 32'd0;
         data[9] = 32'd0;
         data[10]= 32'd31;
    end
    always @(posedge clk) begin
        if (wr_enable)
            data[address_in[15:0]] <= data_in;
    end
    always @(*) begin
        if (read)
            data_out <= data[address_in];
    end
endmodule

module threeToOneMux(input [31:0] inputA,input [31:0] inputB,input [31:0] inputC,input [1:0] select,output reg [31:0] result);
   always @(*) begin
        case (select)
            2'b00: result = inputA;
            2'b01: result = inputB;
            2'b10: result = inputC;
            default: result = 32'b0;
        endcase
    end
endmodule

module twoToOneMux(input [31:0] inputA, input  [31:0] inputB, input select, output reg [31:0] result );
    always @(*) begin
        result = select ? inputB : inputA;
    end
endmodule

module oneBitAdder(
    input A, B, carry_in,output reg sum, carry_out);
    always @(A, B, carry_in) begin
        sum  = A ^ B ^ carry_in;
        carry_out = (A & B) | ((A ^ B) & carry_in);
    end
endmodule

module rippleAdder32(A, B, carry_in, sum, carry_out);
    input [31:0] A, B;
    input carry_in;
    output [31:0] sum;
    output carry_out;

    wire [32:0] carry;

    assign carry[0] = 0;
    genvar i;
    generate
        for (i = 0; i < 32; i = i + 1) begin: adder_chain
            oneBitAdder fa(A[i], B[i], carry[i], sum[i], carry[i + 1]);
        end
    endgenerate
    assign carry_out = carry[32];
endmodule

module negate( input [31:0] in, output reg [31:0] result);
    always @(in) begin
        result = ~in + 1'b1; //twoCOMPLEMENT
    end
endmodule

`define ADD_OP 3'b000
`define NEG_OP 3'b110
`define SUB_OP 3'b101
`define PASS_A_OP 3'b111
`define NO_OP 3'b010

module ALU( input [31:0] A, B, input [2:0] sel, output reg [31:0] out, output reg Z, output reg N);
 always @(*) begin
   case(sel)
      `ADD_OP:    out = B +  A;
      `SUB_OP:    out = B - A;
      `NEG_OP:    out = ~B + 1;
      `PASS_A_OP: out =  A;
      default:    out = 32'd0;
    endcase
    Z = (sel != `PASS_A_OP) && (out == 0);
    N = (sel != `PASS_A_OP) && out[31];
  end
endmodule
 
//All OPP CODES FOR INSTRUCTION
`define SVPC_OPCODE 4'b1111
`define LD_OPCODE 4'b1110
`define ST_OPCODE 4'b0011
`define ADD_OPCODE 4'b0100
`define INC_OPCODE 4'b0101
`define NEG_OPCODE 4'b0110
`define SUB_OPCODE 4'b0111
`define JUMP 4'b1000 
`define BRZ_OPCODE 4'b1001
`define BRN_OPCODE 4'b1011
`define NOP_OPCODE 4'b0000
`define JUMPMEM_OPCODE 4'b1010 

module control(
    input [31:0] instr, 
    output reg reg_write,
    output reg alu_src,
    output reg [2:0] alu_op,
    output reg spc, 
    output reg mem_read,
    output reg mem_write,
    output reg jump_mem,
    output reg mem_to_reg,
    output reg jump,
    output reg brz,
    output reg brn );
    wire [3:0] opcode = instr[31:28];

  always @(*) begin
    reg_write  = 1'b0;
    alu_src    = 1'b0;
    alu_op     = `PASS_A_OP;
    spc        = 1'b0;
    mem_read   = 1'b0;
    mem_write  = 1'b0;
    jump_mem   = 1'b0;
    mem_to_reg = 1'b0;
    jump       = 1'b0;
    brz        = 1'b0;
    brn        = 1'b0;

    //turning ON when NEEDED
      `SVPC_OPCODE: begin
        reg_write  = 1'b1;
        alu_src    = 1'b1;
        alu_op     = `ADD_OP;
        spc        = 1'b1;
      end

      `LD_OPCODE: begin
        reg_write  = 1'b1;
        alu_src    = 1'b0;
        alu_op     = `PASS_A_OP;
        mem_read   = 1'b1;
        mem_to_reg = 1'b1;
      end

      `ST_OPCODE: begin
        mem_write  = 1'b1;
      end

      `ADD_OPCODE: begin
        reg_write  = 1'b1;
        alu_src    = 1'b0;
        alu_op     = `ADD_OP;
      end

      `INC_OPCODE: begin
        reg_write  = 1'b1;
        alu_src    = 1'b1;
        alu_op     = `ADD_OP;
      end

      `NEG_OPCODE: begin
        reg_write  = 1'b1;
        alu_src    = 1'b0;
        alu_op     = `NEG_OP;
      end

      `SUB_OPCODE: begin
        reg_write  = 1'b1;
        alu_src    = 1'b0;
        alu_op     = `SUB_OP;
      end

      `JUMP_OPCODE: begin
        // immediate jump: PC = PC + imm
        jump       = 1'b1;
      end

      `JUMPMEM_OPCODE: begin
        // register‐based jump: PC = M[rs]
        mem_read   = 1'b1;
        jump_mem   = 1'b1;
        mem_to_reg = 1'b1;
        jump       = 1'b1;
      end

      `BRZ_OPCODE: begin
        jump       = 1'b1;
        brz        = 1'b1;
      end

      `BRN_OPCODE: begin
        jump       = 1'b1;
        brn        = 1'b1;
      end

      default: begin
        // NOP 
      end
    endcase
  end
endmodule

module exWbBuffer (
    input  wire        clk,
    input  wire        in_reg_write,
    input  wire        in_mem_to_reg,
    input  wire        in_jump_mem,
    input  wire [31:0] in_alu_result,
    input  wire [31:0] in_mem_data,
    input  wire [5:0]  in_rd_addr,
    output reg         out_reg_write,
    output reg         out_mem_to_reg,
    output reg         out_jump_mem,
    output reg [31:0]  out_alu_result,
    output reg [31:0]  out_mem_data,
    output reg [5:0]   out_rd_addr);

    always @(posedge clk) begin
        out_reg_write   <= in_reg_write;
        out_mem_to_reg  <= in_mem_to_reg;
        out_jump_mem    <= in_jump_mem;
        out_alu_result  <= in_alu_result;
        out_mem_data    <= in_mem_data;
        out_rd_addr     <= in_rd_addr;
    end
endmodule



module idExMemBuffer (
    input  wire clk,
    input  wire in_reg_write,
    input  wire in_jump_mem,
    input  wire in_mem_to_reg,
    input  wire in_mem_read,
    input  wire in_mem_write,
    input  wire [2:0]  in_alu_op,
    input  wire in_save_pc,
    input  wire in_alu_src,
    input  wire [31:0] in_pc,
    input  wire [31:0] in_r1,
    input  wire [31:0] in_r2,
    input  wire [31:0] in_imm,
    input  wire [5:0]  in_rd_addr,
    input  wire in_brz,
    input  wire in_brn,
    input  wire in_jump,
    output reg  out_reg_write,
    output reg  out_jump_mem,
    output reg  out_mem_to_reg,
    output reg  out_mem_read,
    output reg  out_mem_write,
    output reg [2:0]   out_alu_op,
    output reg  out_save_pc,
    output reg  out_alu_src,
    output reg [31:0]  out_pc,
    output reg [31:0]  out_r1,
    output reg [31:0]  out_r2,
    output reg [31:0]  out_imm,
    output reg [5:0]   out_rd_addr,
    output reg  out_brz,
    output reg  out_brn,
    output reg  out_jump
);

    always @(posedge clk) begin
        // WB controls
        out_reg_write  <= in_reg_write;
        out_jump_mem   <= in_jump_mem;
        out_mem_to_reg <= in_mem_to_reg;

        out_mem_read   <= in_mem_read;
        out_mem_write  <= in_mem_write;
        out_alu_op     <= in_alu_op;
        out_save_pc    <= in_save_pc;
        out_alu_src    <= in_alu_src;

        out_pc         <= in_pc;
        out_r1         <= in_r1;
        out_r2         <= in_r2;
        out_imm        <= in_imm;
        out_rd_addr    <= in_rd_addr;

        out_brz        <= in_brz;
        out_brn        <= in_brn;
        out_jump       <= in_jump;
    end

endmodule


module ifIdBuffer(input clock, 
                 input [31:0] pc,
                 input [31:0] inst,
                 output reg [31:0] pc_out, pcPo,
                 output reg [31:0] inst_out);
    always@(posedge clock)
    begin
        pcPo = pc+1; //adding one for the PC COUNTER
        pc_out = pc;
        inst_out = inst;   
    end   
endmodule


module immGen( input  wire [31:0] immIn, output reg  [31:0] immOut);
  wire [3:0] op = immIn[31:28];

  always @(*) begin
    case (op)
      `SVPC_OPCODE:   immOut = {{10{immIn[21]}}, immIn[21:0]};
      `LD_OPCODE,
      `ST_OPCODE,
      `INC_OPCODE,
      `BRZ_OPCODE,
      `BRN_OPCODE,
      `JUMPMEM_OPCODE:
                      immOut = {{16{immIn[15]}}, immIn[15:0]};
      default:        immOut = 32'd0;
    endcase
  end
endmodule




module instructionMemory(input clock, input [7:0] address, output reg [31:0] instruction_out);
        wire [31:0] data [255:0];        
assign data[0]  = 32'b0000_000000_000000_000000_0000000000; // NOP
assign data[1]  = 32'b1111_001000_000000_000000_0000000100; // SVPC x8, 4        ; x1 = PC + 4
assign data[2]  = 32'b1111_000000_000000_000000_0000000000; // SVPC x0, 0        ; x0 = PC (likely redundant since x0 usually = 0)
assign data[3]  = 32'b0000_000000_000000_000000_0000000000; // NOP
assign data[4]  = 32'b0000_000000_000000_000000_0000000000; // NOP
assign data[5]  = 32'b0000_000000_000000_000000_0000000000; // NOP
assign data[6]  = 32'b0000_000000_000000_000000_0000000000; // NOP

assign data[7]  = 32'b1110_000010_001000_000000_0000000000; // LD x2, x8        ; x2 = Mem[x1]

assign data[8]  = 32'b0000_000000_000000_000000_0000000000; // NOP
assign data[9]  = 32'b0000_000000_000000_000000_0000000000; // NOP
assign data[10] = 32'b0000_000000_000000_000000_0000000000; // NOP

assign data[11] = 32'b0101_000011_001000_000010_0000000000; // ADD x3, x8, x2    ; x3 = x8 + x2 = 16
assign data[12] = 32'b0011_001000_001000_000000_0000000000; // STR x8 x8
assign data[13] = 32'b0110_000101_000000_000010_0000000000; // NEG x5, x2        ; x5 = -x2



assign data[14] = 32'b0111_001100_000010_000010_0000000000; //turn this into sub x12 x2, x2
//  9: BRZ  x0         
assign data[15]  = 32'b1001_000000_000000_000000_0000000000;

assign data[16] = 32'b0111_001011_000101_000010_0000000000;// turn this into sub x11 x5-x2 `define JUMPMEM_OPCODE 4'b1000 define ST_OPCODE 4'b0011 define INC_OPCODE 4'b0101
// 10: BRN  x8        
assign data[17] = 32'b1011_000000_001000_000000_0000000000;
assign data[18] = 32'b1000_001110_000000_000000_0000000000; // Jumpmem

assign data[19]  = 32'b1000_000000_000010_000000_0000000000; // Jump to x2 ie inst 10

    always@(posedge clock)
    begin
        instruction_out = data[address]; // reads the instructions at the given input address
    end
endmodule


module PC(input clock, 
          input [31:0] pc_in, 
          input reset,
          output reg [31:0] pc_out);
    
    initial begin
        pc_out = 0;
    end
          
    always @(negedge clock) 
        begin 
            if (pc_in) //Works because we dont route back to PC=0
                begin 
                    pc_out = pc_in;
                end 
         end

endmodule 


module registerFile(input clk, 
    input write, 
    input [5:0] rs, 
    input [5:0] rt, 
    input [5:0] rd, 
    input [31:0] data_in, 
    output reg [31:0] rs_out, 
    output reg [31:0] rt_out
);
    reg [31:0] registers[63:0];
    always @(posedge clk) begin
        if (write)
            registers[rd] = data_in;
    end
    always @(*) begin
        rs_out = registers[rs];
        rt_out = registers[rt];
    end
endmodule

