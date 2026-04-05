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
                       
