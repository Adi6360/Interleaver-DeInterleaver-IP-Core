Below is a complete, exam / project-ready explanation that directly matches what is shown in your handwritten IP-core instantiation notes (Interleaver IP + blk_mem_gen_0) and answers exactly what you asked:

Rectangular Block Interleaver (4×4) using BRAM → AXI-Stream → Interleaver IP,
with MATLAB COE generation and behavioral simulation procedure.

I am writing this in UGC / IEEE project style, suitable for Vivado simulation + report submission.

1. System Overview (What you are building)

You are implementing a 4×4 Rectangular Block Interleaver using:

IP blocks

blk_mem_gen_0

Stores input data (from MATLAB .coe)

Acts as input buffer

AXI-Stream Interleaver IP (sdi_0)

Reads sequential data

Outputs interleaved data

Custom RTL control logic

BRAM read address FSM

AXI-Stream handshaking

Data flow
MATLAB → interleaver_input.coe
        ↓
blk_mem_gen_0 (BRAM)
        ↓
AXI-Stream (s_axis)
        ↓
Interleaver IP (Rectangular 4×4)
        ↓
AXI-Stream (m_axis)
        ↓
Simulation output

2. 4×4 Rectangular Block Interleaving Rule
Input Matrix (Row-wise fill)
0   1   2   3
4   5   6   7
8   9  10  11
12 13  14  15

Interleaved Output (Column-wise read)
0,4,8,12,
1,5,9,13,
2,6,10,14,
3,7,11,15


This is exactly what the Xilinx Interleaver IP performs when configured for rectangular block mode.

3. MATLAB COE File Generation (Input to BRAM)
MATLAB Code (interleaver_input.coe)
% ============================================================
% COE file for 4x4 Rectangular Block Interleaver
% Vivado blk_mem_gen input
% ============================================================

clc; clear;

R = 4;
C = 4;
N = R * C;

data = 0:N-1;   % Sequential symbols

fid = fopen('interleaver_input.coe','w');

fprintf(fid,'memory_initialization_radix=10;\n');
fprintf(fid,'memory_initialization_vector=\n');

for i = 1:N
    if i == N
        fprintf(fid,'%d;\n', data(i));
    else
        fprintf(fid,'%d,\n', data(i));
    end
end

fclose(fid);


✔ Load this .coe into blk_mem_gen_0
✔ Data width: 8 bits
✔ Depth: 16

4. blk_mem_gen_0 Configuration (Important)
Parameter	Value
Interface	Native
Memory Type	Single Port ROM
Data Width	8
Depth	16
Load Init File	interleaver_input.coe
Enable Port	Always enabled
5. Interleaver IP Core Configuration

From your handwritten sdi_0 instantiation:

Key settings
Parameter	Value
Mode	Rectangular Block
Rows	4
Columns	4
Data Width	8
Interface	AXI-Stream
Latency	Auto
6. Top-Level RTL (BRAM → AXI-Stream → Interleaver)
interleaver_top.v
module interleaver_top (
    input  wire        aclk,
    input  wire        aresetn,
    output wire [7:0]  interleaved_out,
    output wire        out_valid
);

    // --------------------------------------------------------
    // BRAM signals
    // --------------------------------------------------------
    reg  [3:0] bram_addr;
    wire [7:0] bram_dout;

    // --------------------------------------------------------
    // AXI-Stream signals
    // --------------------------------------------------------
    reg        s_axis_tvalid;
    wire       s_axis_tready;
    reg  [7:0] s_axis_tdata;
    reg        s_axis_tlast;

    wire [7:0] m_axis_tdata;
    wire       m_axis_tvalid;

    // --------------------------------------------------------
    // BRAM Instance
    // --------------------------------------------------------
    blk_mem_gen_0 u_bram (
        .clka(aclk),
        .ena(1'b1),
        .addra(bram_addr),
        .douta(bram_dout)
    );

    // --------------------------------------------------------
    // Interleaver IP Instance (as per your notes)
    // --------------------------------------------------------
    sdi_0 u_interleaver (
        .aclk(aclk),
        .aresetn(aresetn),

        .s_axis_data_tvalid(s_axis_tvalid),
        .s_axis_data_tready(s_axis_tready),
        .s_axis_data_tdata(s_axis_tdata),
        .s_axis_data_tlast(s_axis_tlast),

        .m_axis_data_tvalid(m_axis_tvalid),
        .m_axis_data_tdata(m_axis_tdata)
    );

    // --------------------------------------------------------
    // BRAM Read + AXI Stream Control FSM
    // --------------------------------------------------------
    always @(posedge aclk) begin
        if (!aresetn) begin
            bram_addr      <= 0;
            s_axis_tvalid  <= 0;
            s_axis_tlast   <= 0;
        end else begin
            s_axis_tvalid <= 1'b1;

            if (s_axis_tready) begin
                s_axis_tdata <= bram_dout;
                bram_addr   <= bram_addr + 1;

                if (bram_addr == 15)
                    s_axis_tlast <= 1'b1;
                else
                    s_axis_tlast <= 1'b0;
            end
        end
    end

    assign interleaved_out = m_axis_tdata;
    assign out_valid       = m_axis_tvalid;

endmodule

7. Testbench (Behavioral Simulation)
tb_interleaver.v
module tb_interleaver;

    reg clk = 0;
    reg rstn = 0;

    wire [7:0] out_data;
    wire out_valid;

    always #5 clk = ~clk;

    interleaver_top DUT (
        .aclk(clk),
        .aresetn(rstn),
        .interleaved_out(out_data),
        .out_valid(out_valid)
    );

    initial begin
        #20 rstn = 1;
        #500 $finish;
    end

    always @(posedge clk) begin
        if (out_valid)
            $display("Time=%0t Interleaved Data=%0d",
                      $time, out_data);
    end

endmodule

8. Expected Simulation Output
0
4
8
12
1
5
9
13
2
6
10
14
3
7
11
15


✔ Confirms correct rectangular interleaving

9. Vivado Simulation Procedure (Step-by-Step)

Create Vivado project

Add RTL files (interleaver_top.v, tb_interleaver.v)

Generate IP:

blk_mem_gen_0

sdi_0

Load MATLAB .coe into BRAM

Run Behavioral Simulation

Observe:

s_axis_tvalid / tready

m_axis_tvalid

Interleaved output sequence

10. Key Viva / Report Points

BRAM decouples memory and processing

AXI-Stream enables scalable streaming

Interleaver IP abstracts permutation logic

COE allows MATLAB ↔ RTL verification

Same design is synthesizable on FPGA

If you want next:
✔ De-interleaver RTL
✔ AXI-Stream waveform explanation
✔ Vivado screenshots + report write-up
✔ ILA hardware validation
✔ IEEE block diagram

Just tell me 👍
