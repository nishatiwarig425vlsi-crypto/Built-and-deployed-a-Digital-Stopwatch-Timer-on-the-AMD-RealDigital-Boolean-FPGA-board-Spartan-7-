# Built-and-deployed-a-Digital-Stopwatch-Timer-on-the-AMD-RealDigital-Boolean-FPGA-board-Spartan-7-
→ Verilog RTL design with clock divider, edge-detection button logic, and BCD counter → Start/Stop control via on-board button with edge-detection to prevent multiple triggers → Live count displayed on 4-digit 7-segment display - verified running in hardware → Post-implementation timing simulation completed in Xilinx Vivado 2025.1  
CODE:-
module top(
    input clk,
    input reset,
    input start_stop,
    output [6:0] seg,
    output [3:0] an
);

wire clk_1hz;
wire [3:0] d0, d1, d2, d3;

clock_divider cd(clk, reset, clk_1hz);
stopwatch sw(clk_1hz, reset, start_stop, d0, d1, d2, d3);
display_mux dm(clk, d0, d1, d2, d3, an, seg);

endmodule
//////////////////////////////////
module clock_divider(
    input clk,
    input reset,
    output reg clk_1hz
);

reg [26:0] count;

always @(posedge clk or posedge reset) begin
    if (reset) begin
        count <= 0;
        clk_1hz <= 0;
    end else begin
        if (count == 50_000_000) begin
            clk_1hz <= ~clk_1hz;
            count <= 0;
        end else begin
            count <= count + 1;
        end
    end
end

endmodule
///////////////////////
module stopwatch(
    input clk,
    input reset,
    input start_stop,
    output reg [3:0] d0, d1, d2, d3
);

reg running = 0;
reg prev_btn = 0;
reg [5:0] seconds = 0;

// Button edge detection (IMPORTANT FIX)
always @(posedge clk) begin
    if (start_stop && !prev_btn)
        running <= ~running;

    prev_btn <= start_stop;
end

// Counter
always @(posedge clk or posedge reset) begin
    if (reset) begin
        seconds <= 0;
    end else if (running) begin
        if (seconds == 59)
            seconds <= 0;
        else
            seconds <= seconds + 1;
    end
end

// Digit split
always @(*) begin
    d0 = seconds % 10;
    d1 = seconds / 10;
    d2 = 0;
    d3 = 0;
end

endmodule

///////////////////////////////

module display_mux(
    input clk,
    input [3:0] d0, d1, d2, d3,
    output reg [3:0] an,
    output [6:0] seg
);

reg [19:0] counter = 0;
reg [1:0] sel;
reg [3:0] digit;

always @(posedge clk) begin
    counter <= counter + 1;
end

// Proper refresh speed (IMPORTANT FIX)
always @(*) begin
    sel = counter[19:18];
end

// Digit selection
always @(*) begin
    case(sel)
        2'b00: begin an = 4'b1110; digit = d0; end
        2'b01: begin an = 4'b1101; digit = d1; end
        2'b10: begin an = 4'b1011; digit = d2; end
        2'b11: begin an = 4'b0111; digit = d3; end
    endcase
end

seven_seg s1(.digit(digit), .seg(seg));

endmodule
////////////////////////

module seven_seg(
    input [3:0] digit,
    output reg [6:0] seg
);

always @(*) begin
    case(digit)
        4'd0: seg = 7'b1000000;
        4'd1: seg = 7'b1111001;
        4'd2: seg = 7'b0100100;
        4'd3: seg = 7'b0110000;
        4'd4: seg = 7'b0011001;
        4'd5: seg = 7'b0010010;
        4'd6: seg = 7'b0000010;
        4'd7: seg = 7'b1111000;
        4'd8: seg = 7'b0000000;
        4'd9: seg = 7'b0010000;
        default: seg = 7'b1111111;
    endcase
end

endmodule
