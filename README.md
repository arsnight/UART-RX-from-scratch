# UART-RX-from-scratch
UART Receiver (RX) implemented in Verilog using an unconventional 28× oversampled clock. Designed from scratch with mid-bit sampling, edge-triggered bit capture, and a fully synchronous FSM. without reliance on external IP cores

## Features

- 28× oversampled clock design
- Mid-bit sampling for reliable reception
- Edge-triggered bit capture
- Falling-edge start bit detection
- Fully synchronous FSM
- No vendor IP cores used
- Simulation-tested with multiple byte sequences

## Design Flow

Started on the Receiver module right after the tx, the approach here is a bit more complicated than the latter. There are many ways to write the RX, but it all narrows to either using the baud rate tick method as used in TX or a new technique called Oversampling.

The basic gist of oversampling is rather simple. Instead of sampling once every 868 clk cycles(for the standard baud rate 115200 and 100MHz master clock), we instead sample multiple times over the course of 868 cycles. This allows to use one of the samples to attain any part of our serial input, most commonly the exact mid part of the serial input is sampled. Mid bit sampling is very good as it prevents sampling during transitions and preventing data corruption.

Usually most common oversampling I noted was 16x oversampling which has it's own problems. For my project here with baud rate 115200, we obtain a decimal divider.
A divider is basically, how many master clock cycles you wait before generating one oversampled tick.
Generally,

                       Divider = (Master Clock frequency)/ (Baud Rate x Oversampling rate)
                       
Since I'm Working with a 100MHz master clock,

                       Divider(16x) = 100,000,000/115200 x 16 = 54.253
                       
A decimal divider. Which opens up error if we approximate either way. 55 clock cycles would be too fast and 54 would be too slow, bound to cause timing issues and small errors. The main reason of using 16x oversampling in the older times was because it was cheaper.
My approach was to just find an oversampling rate that evaluated the divider to an integer. The closest next oversampling rate was 28x,

                       Divider(28x) = 100,000,000/115200 x 28 = 31.000
                       
More precisely,

                       115200 x 28  = 3225600
                       3225600 x 31 = 99,993,600
Error,

          100,000,000 - 99,993,600  = 6400
                  6400/100,000,000  = 0.0064% error only
                       
Now that the oversampling rate is decided, the time has come to finally design the module itself. Forget 868, Instead use the theory of divider to sample once every 31 clock cycles and use a counter to sample 28 times and reset once the counter hits 27(counting starts at 0):
``` verilog
module oversampled_clk(
    input clk,
    input rst_trigger, // Added a Reset trigger for future use
    output reg tick
    );
    
    reg [5:0] count = 0;
    
    
    always @(posedge clk) begin
        tick <= 0;
        if (rst_trigger) begin
            count <= 0;
            tick <= 0;
        end
        
        else if (count == 5'd30) begin
            tick <= 1;
            count <= 0;
        end else begin
            count <= count + 1'b1;
        end    
    end
endmodule
```
And designing the counter:
``` verilog
module Oversampled_counter(
    input clk,
    input rst_trigger,
    output reg [4:0] counter = 0
);

    wire tick;

    oversampled_clk clk_tick(
        .clk(clk),
        .rst_trigger(rst_trigger),
        .tick(tick)
    );
    
    always @(posedge clk) begin
    
    
        if (rst_trigger) begin  
            counter <= 0;           
        end
        else if (tick) begin
           if (counter == 5'd27) begin
                counter <= 0;
           end
           else begin
                counter <= counter + 1'b1;
           end
        end
    end    
endmodule
```
using if (counter == 5'd14) allows mid sampling.

Now time to proceed to the Finite state machine. After making both TX and RX modules the one thing I can't ignore is the level triggered FSM designs breaking the entire module. Level triggers are annoyingly bad, causing frustrations in debugging phases and simulation checks. Always make edge triggered design unless there is an explicit need for a level triggered module. The FSM was straightforward and just like the FSM of the TX. Aside from the level triggered issue I committed during the first designs the rest was simple:
``` verilog
`timescale 1ns / 1ps

module UART_RX(
    input clk,
    input serial_out,
    output reg [7:0] parallel_out = 0
); 
    
    reg [2:0] state = 0;
    reg prev_serial_out = 0;
    reg [2:0] count = 0;
    reg rst_trigger = 0;
    reg [4:0] prev_counter = 0;
    wire [4:0] counter; //wire shouldn't be initialized, Module output already drives it
    wire falling_edge;
    
    assign falling_edge = (prev_serial_out == 1 && serial_out == 0); //falling_edge is always recomputed whenever prev_serial_out OR serial_out changes
    
    
    localparam IDLE = 0;
    localparam START_RX = 1;
    localparam DATA_RX = 2;
    localparam STOP_RX = 3;
    
    Oversampled_counter clk_counter(
        .clk(clk),
        .rst_trigger(rst_trigger),
        .counter(counter)
    );
    
    always @(posedge clk) begin
       rst_trigger <= 0;
       case(state)
            IDLE: begin
                if (prev_serial_out == 1 && serial_out == 0) begin
                    rst_trigger <= 1;
                    state <= START_RX;
                end
            end
            START_RX: begin
                if (counter == 5'd14 && prev_counter != counter) begin
                    if(serial_out == 0) begin
                        state <= DATA_RX;
                    end else begin
                        state <= IDLE;
                    end
                end
            end
            DATA_RX: begin
                if (counter == 5'd14 && prev_counter != counter) begin
                    parallel_out[count] <= serial_out;
        
                if (count == 3'd7) begin
                    state <= STOP_RX;
                    end else begin
                        count <= count + 1'b1;
                    end
                end
            end   
            STOP_RX: begin
                if (counter == 5'd14 && prev_counter != counter) begin
                    state <= IDLE;
                    count <= 0;
                    rst_trigger <= 1;
                end               
            end
            default: state <= IDLE;
        endcase
        prev_counter <= counter;
        prev_serial_out <= serial_out;
   end
   
endmodule
```
