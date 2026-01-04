# Interleaver-DeInterleaver-IP-Core
Below is a complete, end-to-end worked example for Rectangular Block Interleaver (4×4) using the Xilinx Interleaver/De-Interleaver v8.0 IP (PG049), exactly aligned with the Vivado AXI4-Stream interface
1️⃣ Why an FSM is REQUIRED (Conceptually)

Without an FSM:

Streaming starts immediately after reset

No clear block start

TLAST may assert, but block restart is uncontrolled

With an FSM:

Explicit IDLE → STREAM → DONE

One R×C block per trigger

Clean TLAST behavior

Supports repeated blocks (future extension)

2️⃣ FSM States Definition
typedef enum logic [1:0] {
    IDLE   = 2'b00,   // Waiting to start block
    STREAM = 2'b01,   // Streaming BRAM data
    DONE   = 2'b10    // Block completed
} state_t;

3️⃣ FSM-Controlled BRAM → AXI-Stream Source (Recommended)
// ============================================================
// FSM-Controlled BRAM → AXI4-Stream Source
// One complete block (DEPTH samples) per start
// ============================================================
module bram_axi_stream_source_fsm #(
    parameter DATA_WIDTH = 8,
    parameter DEPTH = 16
)(
    input  wire                     clk,
    input  wire                     rst,

    // Control
    input  wire                     start,   // Start block streaming
    output reg                      done,    // Block complete

    // AXI4-Stream Master
    output reg                      m_axis_tvalid,
    input  wire                     m_axis_tready,
    output reg  [DATA_WIDTH-1:0]    m_axis_tdata,
    output reg                      m_axis_tlast,

    // BRAM Interface
    output reg                      bram_en,
    output reg  [$clog2(DEPTH)-1:0] bram_addr,
    input  wire [DATA_WIDTH-1:0]    bram_dout
);

    // FSM state register
    typedef enum logic [1:0] {IDLE, STREAM, DONE} state_t;
    state_t state, next_state;

    // --------------------------------------------------------
    // FSM: State Register
    // --------------------------------------------------------
    always @(posedge clk) begin
        if (rst)
            state <= IDLE;
        else
            state <= next_state;
    end

    // --------------------------------------------------------
    // FSM: Next-State Logic
    // --------------------------------------------------------
    always @(*) begin
        next_state = state;
        case (state)
            IDLE:   if (start) next_state = STREAM;
            STREAM: if (m_axis_tready && bram_addr == DEPTH-1)
                        next_state = DONE;
            DONE:   next_state = IDLE;
        endcase
    end

    // --------------------------------------------------------
    // FSM: Output & Datapath Logic
    // --------------------------------------------------------
    always @(posedge clk) begin
        if (rst) begin
            bram_addr     <= 0;
            bram_en       <= 0;
            m_axis_tvalid <= 0;
            m_axis_tlast  <= 0;
            done          <= 0;
        end
        else begin
            case (state)

                // ---------------- IDLE ----------------
                IDLE: begin
                    bram_addr     <= 0;
                    bram_en       <= 0;
                    m_axis_tvalid <= 0;
                    m_axis_tlast  <= 0;
                    done          <= 0;
                end

                // ---------------- STREAM ----------------
                STREAM: begin
                    if (m_axis_tready) begin
                        bram_en       <= 1;
                        m_axis_tvalid <= 1;
                        m_axis_tdata  <= bram_dout;

                        m_axis_tlast  <= (bram_addr == DEPTH-1);
                        bram_addr     <= bram_addr + 1;
                    end
                end

                // ---------------- DONE ----------------
                DONE: begin
                    m_axis_tvalid <= 0;
                    m_axis_tlast  <= 0;
                    bram_en       <= 0;
                    done          <= 1;   // One-cycle pulse
                end

            endcase
        end
    end

endmodule

4️⃣ How This Connects to Interleaver / De-Interleaver
bram_axi_stream_source_fsm u_src (
    .clk(clk),
    .rst(rst),
    .start(start_block),
    .done(block_done),

    .m_axis_tvalid(s_axis_tvalid),
    .m_axis_tready(s_axis_tready),
    .m_axis_tdata (s_axis_tdata),
    .m_axis_tlast (s_axis_tlast),

    .bram_en  (bram_en),
    .bram_addr(bram_addr),
    .bram_dout(bram_dout)
);


start_block → asserted once to send one R×C block

block_done → indicates TLAST already sent

Perfect for Interleaver → De-Interleaver chaining

5️⃣ Timing Behavior (What the Examiner Looks For)
Signal	Behavior
start	Triggers block transmission
TVALID	Active only in STREAM
TLAST	High on final symbol
done	1-cycle pulse after TLAST
Backpressure	Handled via TREADY
6️⃣ IEEE / Thesis-Ready Explanation

An FSM-controlled AXI4-Stream source is implemented to regulate the start and termination of each rectangular block. The FSM transitions from an idle state to a streaming state upon assertion of a start signal and transmits exactly R×C symbols from BRAM while monitoring AXI backpressure. TLAST is asserted on the final symbol, and a completion signal is generated before returning to the idle state, ensuring deterministic block-level data transfer.

7️⃣ Optional Enhancements (Tell me if you want)

Multi-block continuous streaming

BLOCK_START flag generation

AXI-stream skid buffer

ILA trigger logic

Unified Interleaver + De-Interleaver FSM

You now have a PG049-clean, examiner-safe block start/stop design ✔
