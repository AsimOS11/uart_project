`default_nettype none

module tt_um_uart_transceiver #(
    parameter CLOCK_FREQ = 50_000_000, // 50 MHz default system clock
    parameter BAUD_RATE  = 9600        // 9600 bits per second
) (
    input  wire [7:0] ui_in,    // Dedicated inputs
    output wire [7:0] uo_out,   // Dedicated outputs
    input  wire [7:0] uio_in,   // IOs: Input path
    output wire [7:0] uio_out,  // IOs: Output path
    output wire [7:0] uio_oe,   // IOs: Enable path (active high: 1=output, 0=input)
    input  wire       ena,      // always 1 when the design is powered
    input  wire       clk,      // clock
    input  wire       rst_n     // reset_n - low to reset
);

    // Calculate how many clock cycles per UART bit
    localparam CLOCKS_PER_BIT = CLOCK_FREQ / BAUD_RATE;

    // --- I/O MAPPING ---
    wire [7:0] tx_data_in  = ui_in;
    wire       rx_in       = uio_in[0];
    wire       tx_start    = uio_in[1];
    wire       parity_sel  = uio_in[2];

    wire       tx_out;
    wire       rx_valid;
    wire       parity_error;
    wire [7:0] rx_data_out_internal;

    // Set bidirectional pins: 0,1,2 as input (0), 3,4,5 as output (1), 6,7 as output (1 - unused)
    assign uio_oe       = 8'b11111000;
    assign uio_out[2:0] = 3'b000;      // unused inputs
    assign uio_out[3]   = tx_out;
    assign uio_out[4]   = rx_valid;
    assign uio_out[5]   = parity_error;
    assign uio_out[7:6] = 2'b00;       // unused outputs

    assign uo_out = rx_data_out_internal;

    // --- UART TRANSMITTER ---
    localparam TX_IDLE   = 3'd0;
    localparam TX_START  = 3'd1;
    localparam TX_DATA   = 3'd2;
    localparam TX_PARITY = 3'd3;
    localparam TX_STOP   = 3'd4;

    reg [2:0]  tx_state;
    reg [15:0] tx_clk_cnt;
    reg [2:0]  tx_bit_cnt;
    reg [7:0]  tx_shift_reg;
    reg [7:0]  tx_data_latch; // Latch data so it doesn't change during transmission
    reg        tx_reg;

    assign tx_out = tx_reg;

    always @(posedge clk) begin
        if (!rst_n) begin
            tx_state      <= TX_IDLE;
            tx_clk_cnt    <= 0;
            tx_bit_cnt    <= 0;
            tx_shift_reg  <= 0;
            tx_data_latch <= 0;
            tx_reg        <= 1'b1; // Idle state for UART is HIGH
        end else begin
            case (tx_state)
                TX_IDLE: begin
                    tx_reg     <= 1'b1;
                    tx_clk_cnt <= 0;
                    tx_bit_cnt <= 0;
                    if (tx_start) begin
                        tx_state      <= TX_START;
                        tx_shift_reg  <= tx_data_in;
                        tx_data_latch <= tx_data_in;
                    end
                end
                TX_START: begin
                    tx_reg <= 1'b0; // Start bit is LOW
                    if (tx_clk_cnt < CLOCKS_PER_BIT - 1) begin
                        tx_clk_cnt <= tx_clk_cnt + 1;
                    end else begin
                        tx_clk_cnt <= 0;
                        tx_state   <= TX_DATA;
                    end
                end
                TX_DATA: begin
                    tx_reg <= tx_shift_reg[0]; // Send LSB first
                    if (tx_clk_cnt < CLOCKS_PER_BIT - 1) begin
                        tx_clk_cnt <= tx_clk_cnt + 1;
                    end else begin
                        tx_clk_cnt   <= 0;
                        tx_shift_reg <= {1'b0, tx_shift_reg[7:1]};
                        if (tx_bit_cnt < 7) begin
                            tx_bit_cnt <= tx_bit_cnt + 1;
                        end else begin
                            tx_bit_cnt <= 0;
                            tx_state   <= TX_PARITY;
                        end
                    end
                end
                TX_PARITY: begin
                    // parity_sel: 0 = Even, 1 = Odd
                    tx_reg <= parity_sel ? ~(^tx_data_latch) : (^tx_data_latch);
                    if (tx_clk_cnt < CLOCKS_PER_BIT - 1) begin
                        tx_clk_cnt <= tx_clk_cnt + 1;
                    end else begin
                        tx_clk_cnt <= 0;
                        tx_state   <= TX_STOP;
                    end
                end
                TX_STOP: begin
                    tx_reg <= 1'b1; // Stop bit is HIGH
                    if (tx_clk_cnt < CLOCKS_PER_BIT - 1) begin
                        tx_clk_cnt <= tx_clk_cnt + 1;
                    end else begin
                        tx_clk_cnt <= 0;
                        tx_state   <= TX_IDLE;
                    end
                end
                default: tx_state <= TX_IDLE;
            endcase
        end
    end

    // --- UART RECEIVER ---
    localparam RX_IDLE   = 3'd0;
    localparam RX_START  = 3'd1;
    localparam RX_DATA   = 3'd2;
    localparam RX_PARITY = 3'd3;
    localparam RX_STOP   = 3'd4;

    reg [2:0]  rx_state;
    reg [15:0] rx_clk_cnt;
    reg [2:0]  rx_bit_cnt;
    reg [7:0]  rx_shift_reg;
    reg [7:0]  rx_data_out_reg;
    reg        rx_valid_reg;
    reg        parity_err_reg;

    // Double register RX input to prevent metastability
    reg rx_sync1, rx_sync2;
    always @(posedge clk) begin
        if (!rst_n) begin
            rx_sync1 <= 1'b1;
            rx_sync2 <= 1'b1;
        end else begin
            rx_sync1 <= rx_in;
            rx_sync2 <= rx_sync1;
        end
    end

    assign rx_data_out_internal = rx_data_out_reg;
    assign rx_valid     = rx_valid_reg;
    assign parity_error = parity_err_reg;

    always @(posedge clk) begin
        if (!rst_n) begin
            rx_state        <= RX_IDLE;
            rx_clk_cnt      <= 0;
            rx_bit_cnt      <= 0;
            rx_shift_reg    <= 0;
            rx_data_out_reg <= 0;
            rx_valid_reg    <= 0;
            parity_err_reg  <= 0;
        end else begin
            rx_valid_reg <= 1'b0; // Default to 0, pulse high for 1 clock cycle only

            case (rx_state)
                RX_IDLE: begin
                    rx_clk_cnt <= 0;
                    rx_bit_cnt <= 0;
                    if (rx_sync2 == 1'b0) begin // Start bit detected
                        rx_state <= RX_START;
                    end
                end
                RX_START: begin
                    // Wait half a bit period to sample the center of the bit
                    if (rx_clk_cnt < (CLOCKS_PER_BIT / 2) - 1) begin
                        rx_clk_cnt <= rx_clk_cnt + 1;
                    end else begin
                        rx_clk_cnt <= 0;
                        if (rx_sync2 == 1'b0) begin // Verify it's still low (not a noise glitch)
                            rx_state <= RX_DATA;
                        end else begin
                            rx_state <= RX_IDLE; // False start, go back to idle
                        end
                    end
                end
                RX_DATA: begin
                    if (rx_clk_cnt < CLOCKS_PER_BIT - 1) begin
                        rx_clk_cnt <= rx_clk_cnt + 1;
                    end else begin
                        rx_clk_cnt   <= 0;
                        rx_shift_reg <= {rx_sync2, rx_shift_reg[7:1]}; // Shift data in
                        if (rx_bit_cnt < 7) begin
                            rx_bit_cnt <= rx_bit_cnt + 1;
                        end else begin
                            rx_bit_cnt <= 0;
                            rx_state   <= RX_PARITY;
                        end
                    end
                end
                RX_PARITY: begin
                    if (rx_clk_cnt < CLOCKS_PER_BIT - 1) begin
                        rx_clk_cnt <= rx_clk_cnt + 1;
                    end else begin
                        rx_clk_cnt <= 0;
                        // Check if received bit matches our expected parity
                        if (rx_sync2 != (parity_sel ? ~(^rx_shift_reg) : (^rx_shift_reg))) begin
                            parity_err_reg <= 1'b1;
                        end else begin
                            parity_err_reg <= 1'b0;
                        end
                        rx_state <= RX_STOP;
                    end
                end
                RX_STOP: begin
                    if (rx_clk_cnt < CLOCKS_PER_BIT - 1) begin
                        rx_clk_cnt <= rx_clk_cnt + 1;
                    end else begin
                        rx_clk_cnt <= 0;
                        if (rx_sync2 == 1'b1) begin // Verify Stop bit is high
                            rx_data_out_reg <= rx_shift_reg; // Push to output
                            rx_valid_reg    <= 1'b1;         // Signal success
                        end
                        rx_state <= RX_IDLE;
                    end
                end
                default: rx_state <= RX_IDLE;
            endcase
        end
    end

endmodule
