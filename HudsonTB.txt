module CPUTB();

    // Clock and reset
    reg clock, reset;

    // PC Connectors
    wire [31:0] pcinSig;
    wire [31:0] pcADD;

    // IF Connectors
    wire [31:0] IFpc;
    wire [31:0] IFinst;

    // ID Connectors
    wire [31:0] IDpc;
    wire [31:0] IDinstr;
    wire [31:0] IDimm;
    wire [31:0] IDr1, IDr2;

    // ID Connectors
    wire        IDalu, IDspc, IDmemR, IDmemW;
    wire        IDbrz, IDbrn, jump_ctrl, topW_ctrl, bottomW_ctrl, bz_sig, bn_sig;
    wire        IDregW, IDjumpMem, IDmemReg;
    wire [2:0]  IDaluOP;
    reg         ID_branch_flag;

    // EXMEM Connectors
    wire [31:0] EXMpcO;
    wire [31:0] EXMr1O, EXMr2O;
    wire [5:0]  EXMrdO;
    wire [31:0] EXMimmO;
    wire        EXMaluSRC, EXMspc, EXMmemread, EXMmemwrite;
    wire        EXMregwrite, EXMjumpMem, EXMmemtoreg;
    wire [2:0]  EXMALUOP;
    wire        EXMzeroFlag, EXMnegFlag;

    // ALU Connectors
    wire [31:0] ALUin1, ALUin2, EXMaluR;

    // Data memory Connectors
    wire [31:0] EXMreadD;

    // WB Connectors
    wire        WBregW, WBjumpMem, WBmemToReg;
    wire [31:0] WBalu;
    wire [31:0] WBmemread;
    wire [5:0]  WBrdAddress;
    wire [31:0] WBwriteData;
    wire [31:0] pcADDR;
    wire jump_output;

    //clock

    initial begin
        reset = 0;
        clock = 0;
        forever #5 clock = ~clock;
    end

    // Muxesto select PC value
    twoToOneMux selectPcA(
        .A(EXMreadD),  
        .B(EXMr1O),  
        .sel(EXMjumpMem),
        .out(pcADDR)
    );
    twoToOneMux selectPcB(
        .A(pcADD),  
        .B(pcADDR), 
        .sel(ID_branch_flag),
        .out(pcinSig)
    );





    // INSTRUCTION FETCH
    PC pc (
        .clock(clock),
        .pc_in(pcinSig),
        .reset(reset),
        .pc_out(IFpc)
    );

    instructionMemory IM (
        .clock(clock),
        .address(IFpc),
        .instruction_out(IFinst)
    );

    // INSTRUCTION DECODE
    ifIdBuffer ifIDbuffer (
        .clock(clock),
        .pc(IFpc),
        .inst(IFinst),
        .pcPo(pcADD),
        .pc_out(IDpc),
        .inst_out(IDinstr)
    );

    control controlU (
        .instr(IDinstr),
        .reg_write(IDregW),
        .alu_src(IDalu),
        .alu_op(IDaluOP),
        .spc(IDspc),
        .mem_read(IDmemR),
        .mem_write(IDmemW),
        .jump_mem(IDjumpMem),
        .mem_to_reg(IDmemReg),
        .jump(jump_ctrl),
        .brz(IDbrz),
        .brn(IDbrn)
    );

    registerFile registerFileModule (
        .clock(clock),
        .write(WBregW),
        .rs(IDinstr[21:16]),
        .rt(IDinstr[15:10]),
        .rd(WBrdAddress),
        .datain(WBwriteData),
        .rs_out(IDr1),
        .rt_out(IDr2)
    );

    immGen immGenModule (
        .immIn(IDinstr),
        .immOut(IDimm)
    );

    //branch selction muxes

    twoToOneMux top( 
        .A(32'b1),
        .B(EXMnegFlag),
        .sel(bn_sig),
        .out(topW_ctrl)
    );

    twoToOneMux bottom(
        .A(32'b1),
        .B(EXMzeroFlag),
        .sel(bz_sig),
        .out(bottomW_ctrl)
    );

    always @(*) begin //jump branch AND gate
        ID_branch_flag = 1'b0;
        if (jump_output & topW_ctrl & bottomW_ctrl)
            ID_branch_flag = 1'b1;
    end

    
idExMemBuffer idTOexmemBuffer (
    .clk            (clock),
    // WB‐stage 
    .in_reg_write   (IDregW),
    .in_mem_to_reg  (IDmemReg),
    .in_jump_mem    (IDjumpMem),
    // Mem/ALU‐stage 
    .in_mem_read    (IDmemR),
    .in_mem_write   (IDmemW),
    .in_alu_op      (IDaluOP),
    .in_save_pc     (IDspc),
    .in_alu_src     (IDalu),
    // Data
    .in_pc          (IDpc),
    .in_r1          (IDr1),
    .in_r2          (IDr2),
    .in_imm         (IDimm),
    .in_rd_addr     (IDinstr[27:22]),
    // Branch/jump flags
    .in_brz         (IDbrz),
    .in_brn         (IDbrn),
    .in_jump        (jump_ctrl),
    // WB‐stage out
    .out_reg_write  (EXMregwrite),
    .out_mem_to_reg (EXMmemtoreg),
    .out_jump_mem   (EXMjumpMem),
    // Mem/ALU‐stage out
    .out_mem_read   (EXMmemread),
    .out_mem_write  (EXMmemwrite),
    .out_alu_op     (EXMALUOP),
    .out_save_pc    (EXMspc),
    .out_alu_src    (EXMaluSRC),
    // Data out
    .out_pc         (EXMpcO),
    .out_r1         (EXMr1O),
    .out_r2         (EXMr2O),
    .out_imm        (EXMimmO),
    .out_rd_addr    (EXMrdO),
    // Branch/jump out
    .out_brz        (bz_sig),
    .out_brn        (bn_sig),
    .out_jump       (jump_output)
);

    twoToOneMux aluin1 (
        .A(EXMr1O),
        .B(EXMpcO),
        .sel(EXMspc),
        .out(ALUin1)
    );

    twoToOneMux aluin2 (
        .A(EXMr2O),
        .B(EXMimmO),
        .sel(EXMaluSRC),
        .out(ALUin2)
    );

    ALU alu (
        .A(ALUin1),
        .B(ALUin2),
        .sel(EXMALUOP),
        .out(EXMaluR),
        .Z(EXMzeroFlag),
        .N(EXMnegFlag)
    );

    datam DM (
        .clock(clock),
        .read(EXMmemread),
        .wrt(EXMmemwrite),
        .addr(EXMr1O),
        .datain(EXMr2O),
        .dataout(EXMreadD)
    );
    //WB

    exWbBuffer exWbBuffer (
    .clk             (clock),
    .in_reg_write    (EXMregwrite),
    .in_mem_to_reg   (EXMmemtoreg),
    .in_jump_mem     (EXMjumpMem),
    .in_alu_result   (EXMaluR),
    .in_mem_data     (EXMreadD),
    .in_rd_addr      (EXMrdO),
    .out_reg_write   (WBregW),
    .out_mem_to_reg  (WBmemToReg),
    .out_jump_mem    (WBjumpMem),
    .out_alu_result  (WBalu),
    .out_mem_data    (WBmemread),
    .out_rd_addr     (WBrdAddress)
    );

    twoToOneMux wb_out_select (
        .A(WBalu),
        .B(WBmemread),
        .sel(WBmemToReg),
        .out(WBwriteData)
    );

    initial begin
        #2000;
    end

endmodule


