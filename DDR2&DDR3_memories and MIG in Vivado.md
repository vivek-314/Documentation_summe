# What DDR memory is?
DDR stands for Double Data Rate memory.
It is a type of RAM (Random Access Memory) that transfers data on both rising edge and falling edge of clock.
Therefore in one cycle data is transferred twice, giving higer speed and better bandwidth comapred to Synchroniced DRAN.

For example if the clock frequency = 100 MHz

In SDRAM effective data rate will be 100 MT/s
but in DDR it will be 200 MT/s

## DDR memory contains:
  - Memory cells
  - Row decoder
  - Column decoder
  - Sense Amplifier
  - Banks
  - Control Logic
  - I/O buffers

![image of ddr](ddr4-basics-banks.png)

## Organisatioon of DDR Memory
DDR memory is divided into banks and these banks contains rows and columns. data is accessed through row address and column address.

## Basic DDR Operations
  1. Activate Command
    Opens a row in a bank.
  2. Read Command
    Reads data from column.
  3. Write Command
    Writes data into memory.
  4. Precharge Command
    Closes currently opened row.
  5. Refresh Command
    Restores charge in memory capacitors.

## DDR timing Parameters 
  CAS Latency (CL): Delay between read command and output data.

  tRCD: Row to column delay.

  tRP: Precharge delay.

  tRAS: Minimum row active time.
  
Advantages of DDR Memory
  - Faster data transfer
  - High bandwidth
  - Efficient memory access
  - Suitable for high-speed systems

Applications of DDR Memory
  - Computers
  - FPGA systems
  - GPUs
  - Embedded systems
  - Servers
  - Networking devices
Basic Verilog code for showing data tranfer on both edges

           module basic_ddr(
                input clk,
                input data_rise,
                input data_fall,
                output reg q
          );

          always @(posedge clk)
              q <= data_rise;

          always @(negedge clk)
              q <= data_fall;
          endmodule

# DDR2 memory (Doble data rate Synchronous DRAM 2nd Gen)
## What is DDR2 memory?
It is the second generation of DDR memory technology and is faster and more power efficient than basic DDR (discussed before). 
It also transferred data at falling and rising edge both but does it faster than DDR memory due to improved architecture.

