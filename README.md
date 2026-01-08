`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
module interleaver_top (
    input  wire aclk,
    input  wire aresetn
);

    localparam DATA_WIDTH = 8;
    localparam BLOCK_SIZE = 16;

    /* ================= BRAM ================= */
    reg  [3:0] sample_cnt;
    wire [3:0] bram_addr = sample_cnt;
    wire [7:0] bram_data;

    /* ================= AXI INPUT ================= */
    reg                   s_axis_data_tvalid;
    wire                  s_axis_data_tready;   // ← FROM IP
    reg                   s_axis_data_tlast;
    wire [7:0]             s_axis_data_tdata = bram_data;

    /* ================= AXI CTRL ================= */
    reg  [31:0]            s_axis_ctrl_tdata;
    reg                   s_axis_ctrl_tvalid;
    wire                  s_axis_ctrl_tready;

    /* ================= AXI OUTPUT ================= */
    wire [7:0]             m_axis_data_tdata;
    wire                   m_axis_data_tvalid;
    wire                   m_axis_data_tlast;
    wire [1:0]             m_axis_data_tuser;

    /* ================= FSM ================= */
    reg [1:0] state;
    localparam IDLE = 2'd0,
               SEND = 2'd1,
               WAIT = 2'd2,
               DONE = 2'd3;

    always @(posedge aclk) begin
        if (!aresetn) begin
            sample_cnt           <= 0;
            s_axis_data_tvalid   <= 0;
            s_axis_data_tlast    <= 0;
            s_axis_ctrl_tvalid   <= 0;
            state                <= IDLE;
        end else begin
            case (state)

            IDLE: begin
                sample_cnt         <= 0;
                s_axis_ctrl_tdata  <= BLOCK_SIZE;   // 16
                s_axis_ctrl_tvalid <= 1'b1;
                state              <= SEND;
            end

            SEND: begin
                s_axis_ctrl_tvalid <= 0;
                s_axis_data_tvalid <= 1'b1;

                if (s_axis_data_tvalid && s_axis_data_tready) begin
                    sample_cnt <= sample_cnt + 1;

                    if (sample_cnt == BLOCK_SIZE-2)
                        s_axis_data_tlast <= 1'b1;
                end

                if (sample_cnt == BLOCK_SIZE-1) begin
                    s_axis_data_tvalid <= 0;
                    s_axis_data_tlast  <= 0;
                    state <= WAIT;
                end
            end

            WAIT: begin
                state <= DONE;
            end

            DONE: begin
                state <= DONE;
            end

            endcase
        end
    end

    /* ================= BRAM ================= */
    blk_mem_gen_0 bram_inst (
        .clka  (aclk),
        .wea   (1'b0),
        .addra (bram_addr),
        .dina  (8'd0),
        .douta (bram_data)
    );

    /* ================= INTERLEAVER IP ================= */
    interleaver_0 interleaver_inst (
        .aclk                  (aclk),
        .aresetn               (aresetn),

        .s_axis_data_tdata     (s_axis_data_tdata),
        .s_axis_data_tvalid    (s_axis_data_tvalid),
        .s_axis_data_tready    (s_axis_data_tready),
        .s_axis_data_tlast     (s_axis_data_tlast),

        .s_axis_ctrl_tdata     (s_axis_ctrl_tdata),
        .s_axis_ctrl_tvalid    (s_axis_ctrl_tvalid),
        .s_axis_ctrl_tready    (s_axis_ctrl_tready),

        .m_axis_data_tdata     (m_axis_data_tdata),
        .m_axis_data_tvalid    (m_axis_data_tvalid),
        .m_axis_data_tlast     (m_axis_data_tlast),
        .m_axis_data_tuser     (m_axis_data_tuser),
        .m_axis_data_tready    (1'b1)
    );

endmodule
//////////////////////////////////////////////////////////////////////////////////



DEINTERLEAVER

`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
// De-Interleaver Top Module
// ---------------------------------------------------------------
// • Reads interleaved data from BRAM
// • Sends AXI-Stream data to Xilinx De-Interleaver IP
// • Rectangular Block Mode (4×4)
// • Symbol width = 8 bits, Block size = 16
// • Complies with PG049 user guide
//////////////////////////////////////////////////////////////////////////////////
module deinterleaver_top (
    input  wire aclk,        // AXI clock
    input  wire aresetn       // Active-low reset
);

    /* ============================================================
       DESIGN PARAMETERS
       ============================================================ */

    localparam DATA_WIDTH = 8;      // Width of each symbol
    localparam BLOCK_SIZE = 16;     // 4×4 rectangular block

    /* ============================================================
       BRAM SIGNALS
       ============================================================ */

    reg  [3:0] sample_cnt;          // Counts symbols (0 → 15)
    wire [3:0] bram_addr;           // BRAM address
    wire [7:0] bram_data;           // Data read from BRAM

    // Address is driven by sample counter
    assign bram_addr = sample_cnt;

    /* ============================================================
       AXI-STREAM INPUT (S_AXIS_DATA)
       ============================================================ */

    reg                   s_axis_data_tvalid;  // Input data valid
    wire                  s_axis_data_tready;  // Ready from IP
    reg                   s_axis_data_tlast;   // End of block indicator
    wire [DATA_WIDTH-1:0] s_axis_data_tdata;   // Input symbol

    // Feed BRAM output directly to AXI stream
    assign s_axis_data_tdata = bram_data;

    /* ============================================================
       AXI-STREAM CONTROL (S_AXIS_CTRL)
       Required for Rectangular Block mode
       ============================================================ */

    reg  [31:0] s_axis_ctrl_tdata;   // Control word
    reg         s_axis_ctrl_tvalid;  // Control valid
    wire        s_axis_ctrl_tready;  // Control ready from IP

    /*
       Control Word (PG049):
       Bits [15:0]  → Block size (16)
       Bits [31:16] → Reserved
    */

    /* ============================================================
       AXI-STREAM OUTPUT (M_AXIS_DATA)
       ============================================================ */

    wire [DATA_WIDTH-1:0] m_axis_data_tdata;   // De-interleaved output
    wire                  m_axis_data_tvalid;  // Output valid
    wire                  m_axis_data_tlast;   // Output TLAST
    wire [1:0]            m_axis_data_tuser;   // User sideband

    /* ============================================================
       FSM DECLARATION
       ============================================================ */

    reg [1:0] state;

    localparam IDLE = 2'd0,   // Send control info
               SEND = 2'd1,   // Stream data
               WAIT = 2'd2,   // Wait for IP completion
               DONE = 2'd3;   // End state

    /* ============================================================
       FSM SEQUENTIAL LOGIC
       ============================================================ */

    always @(posedge aclk) begin
        if (!aresetn) begin
            // Reset all state and control signals
            sample_cnt         <= 0;
            s_axis_data_tvalid <= 0;
            s_axis_data_tlast  <= 0;
            s_axis_ctrl_tvalid <= 0;
            state              <= IDLE;
        end
        else begin
            case (state)

            /* ----------------------------------------------------
               IDLE STATE
               Send block size control to De-Interleaver
               ---------------------------------------------------- */
            IDLE: begin
                sample_cnt         <= 0;
                s_axis_ctrl_tdata  <= BLOCK_SIZE; // 16 symbols
                s_axis_ctrl_tvalid <= 1'b1;
                state              <= SEND;
            end

            /* ----------------------------------------------------
               SEND STATE
               Stream interleaved symbols into IP
               ---------------------------------------------------- */
            SEND: begin
                s_axis_ctrl_tvalid <= 1'b0;        // Control sent once
                s_axis_data_tvalid <= 1'b1;        // Data valid

                // AXI handshake
                if (s_axis_data_tvalid && s_axis_data_tready) begin
                    sample_cnt <= sample_cnt + 1;

                    // Assert TLAST on last-1 transfer
                    if (sample_cnt == BLOCK_SIZE-2)
                        s_axis_data_tlast <= 1'b1;
                end

                // After last symbol
                if (sample_cnt == BLOCK_SIZE-1) begin
                    s_axis_data_tvalid <= 1'b0;
                    s_axis_data_tlast  <= 1'b0;
                    state <= WAIT;
                end
            end

            /* ----------------------------------------------------
               WAIT STATE
               Allow De-Interleaver to output data
               ---------------------------------------------------- */
            WAIT: begin
                state <= DONE;
            end

            /* ----------------------------------------------------
               DONE STATE
               Terminal state
               ---------------------------------------------------- */
            DONE: begin
                state <= DONE;
            end

            endcase
        end
    end

    /* ============================================================
       BLOCK MEMORY GENERATOR
       Holds INTERLEAVED input data
       ============================================================ */

    blk_mem_gen_0 bram_inst (
        .clka  (aclk),        // Clock
        .wea   (1'b0),        // Read-only
        .addra (bram_addr),   // Address
        .dina  (8'd0),        // Not used
        .douta (bram_data)    // Output symbol
    );

    /* ============================================================
       DE-INTERLEAVER IP CORE
       (Configured as De-Interleaver in Vivado)
       ============================================================ */

    deinterleaver_0 deinterleaver_inst (
        .aclk                  (aclk),
        .aresetn               (aresetn),

        // AXI-Stream data input
        .s_axis_data_tdata     (s_axis_data_tdata),
        .s_axis_data_tvalid    (s_axis_data_tvalid),
        .s_axis_data_tready    (s_axis_data_tready),
        .s_axis_data_tlast     (s_axis_data_tlast),

        // AXI-Stream control input
        .s_axis_ctrl_tdata     (s_axis_ctrl_tdata),
        .s_axis_ctrl_tvalid    (s_axis_ctrl_tvalid),
        .s_axis_ctrl_tready    (s_axis_ctrl_tready),

        // AXI-Stream output
        .m_axis_data_tdata     (m_axis_data_tdata),
        .m_axis_data_tvalid    (m_axis_data_tvalid),
        .m_axis_data_tlast     (m_axis_data_tlast),
        .m_axis_data_tuser     (m_axis_data_tuser),
        .m_axis_data_tready    (1'b1)
    );

endmodule
//////////////////////////////////////////////////////////////////////////////////
