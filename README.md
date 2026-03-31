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
