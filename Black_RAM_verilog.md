# Block RAM (BRAM)

## What is a BRAM ?
It is internal memory block inside FPGA. 
  - It is used to store large amount of data inside FPGA board
  - Helps in faster read/write operation.
  - Used commonly for FIFOs, Image processing, lookup tables and storing data in DSP system 

## Why we need to use BRAM?
Using registers for large memory is waste of FPGA logic resources.
[Logic resources in FPGAs are the programmable hardware components used to implement digital circuits, primarily consisting of Look-Up Tables (LUTs), Flip-Flops (FFs), and Multiplexers (MUXes)]

  - It Protects Flip-Flops (FFs) from Bulk Data Exhaustion
    - **The Math**: A modest 32-bit wide by 1024-deep data buffer requires 32,768 flip-flops.
    - **The Waste**: This single buffer could easily consume 20% to 50% of the entire flip-flop supply on a small FPGA.
    - **The BRAM Solution**: Storing this same buffer in BRAM consumes zero flip-flops, leaving 100% of them available for your system's control logic and state machines.
  - It Frees Up Look-Up Tables (LUTs) from Addressing Duty.
    - **The Waste**: A 1024-to-1 multiplexer requires thousands of LUTs just to handle the addressing logic.
    - **The BRAM Solution**: BRAM blocks contain their own hardwired address decoders etched into the silicon. It handles all memory addressing internally, saving thousands of LUTs from being wasted as simple routing switches.
  - It Prevents Routing Channel Congestion
    - **The Waste**: Connecting tens of thousands of individual flip-flops together to form a memory array forces the compiler to use up miles of global routing wires. This creates a massive traffic jam, leaving fewer wires for the rest of your circuit design.
    - **The BRAM Solution**: Because the memory cells in a BRAM block are permanently wired together on the silicon, data stays localized. It only uses a tiny handful of entry and exit wires to connect to the rest of the FPGA fabric.
  - It Eliminates Place-and-Route Compiling Failures
    - **The Waste**: This often leads to compile errors (Fitting/Routing failures) where the design simply will not fit on the chip, forcing you to buy a larger, more expensive FPGA.
    - **The BRAM Solution**: The compiler drops the data into a single, pre-defined BRAM slot instantly, guaranteeing a successful build with minimal software effort.

