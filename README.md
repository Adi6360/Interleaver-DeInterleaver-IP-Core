WORKING CODE ONLY (0,1) ARE SHOWING


============================================================

`timescale 1ns / 1ps

module interleaver (
    input  wire aclk,
    input  wire aresetn
);

    /* ============================================================
       PARAMETERS
       ============================================================ */
    localparam DATA_WIDTH = 8;
    localparam BLOCK_SIZE = 16;

    /* ============================================================
       BRAM SIGNALS
       ============================================================ */
    wire [3:0] bram_addr;
    reg  [3:0] sample_cnt;

    assign bram_addr = sample_cnt;

    /* ============================================================
       AXI STREAM INPUT
       ============================================================ */
    wire [DATA_WIDTH-1:0] s_axis_data_tdata;
    reg                   s_axis_data_tvalid;
    reg                   s_axis_data_tready;
    reg                   s_axis_data_tlast;

    wire data_tready;
    wire data_tlast;
    wire data_tvalid;

    /* ============================================================
       AXI STREAM OUTPUT
       ============================================================ */
    wire [DATA_WIDTH-1:0] m_axis_data_tdata;
    wire                  m_axis_data_tvalid;
    wire                  m_axis_data_tlast;
    wire [1:0]            m_axis_data_tuser;
    wire                  m_axis_data_tready;

    assign m_axis_data_tready = 1'b1;

    assign data_tready = s_axis_data_tready;
    assign data_tvalid = s_axis_data_tvalid;
    assign data_tlast  = s_axis_data_tlast;

    reg [1:0] state;

    /* ============================================================
       FSM STATES
       ============================================================ */
    localparam IDLE = 2'b00;
    localparam SEND = 2'b01;
    localparam WAIT = 2'b10;
    localparam DONE = 2'b11;

    /* ============================================================
       FSM SEQUENTIAL LOGIC
       ============================================================ */
    always @(posedge aclk) begin
        if (!aresetn) begin
            sample_cnt         <= 0;
            s_axis_data_tvalid <= 0;
            s_axis_data_tlast  <= 0;
            s_axis_data_tready <= 0;   // ✅ FIX #3
            state              <= IDLE;
        end
        else begin
            s_axis_data_tvalid <= 1;

            case (state)

            IDLE: begin
                sample_cnt        <= 0;
                s_axis_data_tlast <= 0;
                state             <= SEND;
            end

            SEND: begin
                s_axis_data_tready <= 1'b1;

                if (s_axis_data_tready && s_axis_data_tvalid) begin
                    if (sample_cnt < BLOCK_SIZE-1)   // ✅ FIX #2
                        sample_cnt <= sample_cnt + 1;
                end

                if (sample_cnt == BLOCK_SIZE-1) begin
                    s_axis_data_tlast <= 1;
                    state <= WAIT;
                end
            end

            WAIT: begin
                s_axis_data_tvalid <= 1'b0;
                s_axis_data_tlast  <= 1'b0;
                state <= DONE;
            end

            DONE: begin
                state <= DONE;
            end

            endcase
        end
    end

    /* ============================================================
       BLOCK MEMORY GENERATOR
       ============================================================ */
    blk_mem_gen_0 bram_inst (
        .clka  (aclk),
        .wea   (1'b0),
        .addra (bram_addr),
        .dina  (8'd0),
        .douta (s_axis_data_tdata)
    );
      /* ---------------- INTERLEAVER ---------------- */
    interleaver_0 interleaver_inst (
        .aclk                  (aclk),
        .aresetn               (aresetn),

        .s_axis_data_tdata     (s_axis_data_tdata),
        .s_axis_data_tvalid    (data_tvalid),
        .s_axis_data_tready    (data_tready),
        .s_axis_data_tlast     (data_tlast),

        .m_axis_data_tvalid    (m_axis_data_tvalid),
        .m_axis_data_tready    (m_axis_data_tready),
        .m_axis_data_tdata     (m_axis_data_tdata),
        .m_axis_data_tlast     (m_axis_data_tlast),
        .m_axis_data_tuser     (m_axis_data_tuser),

        .event_tlast_unexpected(event_tlast_unexpected),
        .event_tlast_missing   (event_tlast_missing),
        .event_halted          (event_halted)
    );
    


endmodule
//////////////////////////////////////////////////////////////////////////////////

<img width="1919" height="967" alt="image" src="https://github.com/user-attachments/assets/05b1d93f-95fd-472b-b2a9-87dff35789a8" />

<img width="1918" height="990" alt="image" src="https://github.com/user-attachments/assets/ac3ade18-b200-43f9-bde4-baaa968a2f77" />






