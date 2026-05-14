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


Generally BRAMs wok with a clock signal.
   - Read and write occur on clock edges.
   - Output is usually available after 1 cycle.

## Single Port BRAM
A Single-Port BRAM features only one shared interface to access the memory array. This interface consists of one clock line, one address bus, one data input bus, one data output bus, and a write enable signal.

![Single port data image](singleportbram.png)

**How It Works:** 
The block can perform only one operation per clock cycle. You can either read from an address or write to an address, but you cannot do both at the same time.
**Limitations:** If your circuit needs to save new incoming data while simultaneously reading out old data, a single-port BRAM will cause a processing bottleneck.
**Common Use Cases:** 
  - Storing permanent microprocessor firmware or initialization boot ROMs.
  - Storing static calibration data or mathematical lookup tables (e.g., Sine/Cosine tables).
  - State machine buffers where read and write operations never happen concurrently.

## Dual Port BRAM

A Dual-Port BRAM splits access into two entirely independent interfaces, typically labeled Port A and Port B. Each port has its own distinct set of clock, address, data in, data out, and write enable lines connected to the exact same underlying memory bank.

![dual port bram image](dualportbram.png)

**How It Works:** 
The memory can handle two separate operations simultaneously in a single clock cycle. Port A can write data to Address 5 while Port B simultaneously reads data from Address 100.

**True vs. Simple Dual-Port:**
  - Simple Dual-Port (SDP): Port A is strictly reserved for writing data, and Port B is strictly reserved for reading data.
  - True Dual-Port (TDP): Both Port A and Port B can independently perform both read and write operations at the same time.
**Clock Domain Crossing (CDC):**
 Because Port A and Port B have completely separate clock pins, they can run at different frequencies. Data can be written into Port A at 100 MHz from one circuit component, and read out of Port B at 250 MHz by a completely different component.

**Common Use Cases:**
  - **FIFO (First-In, First-Out) Buffers:** Passing data safely between two distinct processing systems running on different clock speeds.
![fifo bram](fifobram.png)
  - **Video Frame Buffers:** A camera interface writes incoming pixels into Port A, while a display controller pulls pixels out of Port B to draw the image on a screen.
  - **Shared Processor Memory:** Allowing a hardware accelerator and an embedded CPU core to read and update the same data table concurrently.


