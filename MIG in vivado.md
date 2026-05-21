# MIG in vivado

## What is MIG?
It stands for Memory Interface Generator.
It is used to interface external DDR memories with FPGA.

MIG automatically generates:
  - DDR controller
  - PHY layer
  - calibration logic
  - timing constraints
  - simulation files


## Why MIG is Needed
External DDR memory is complex because it requires:

  - precise timing
  - high-speed signaling
  - refresh operations
  - burst transfers
  - clock alignment
  - calibration

Direct communication between FPGA logic and DDR memory is very difficult.
MIG solves this problem by generating the complete memory controller automatically.

### Role of MIG

      User Logic
         ↓
    MIG Controller
         ↓
    DDR3 / DDR4 Memory

## Components of MIG

  1) DDR Controller
     
     Handles
       - read/write commands
       - burst operations
       - refresh cycles
       - memory scheduling
    
  2) PHY layer
     
     Handles physical DDR communication:

        - DQS strobes
        - clock synchronization
        - signal timing
        - data alignment
    
  3) Calibration Logic
     
     At startup MIG performs calibration to:

        - adjust delays
        - align clocks and data
        - ensure reliable high-speed communication

## Simple working of MIG.

**Write Operation**

  1) User logic sends:
      - address
      - write command
      - write data
  2) MIG converts it into DDR protocol.
  3) Data is stored in DDR memory.  

**Read Operation**
  1) User logic sends read address.
  2) MIG fetches data from DDR memory.
  3) MIG returns read data to FPGA logic

## Difference between BRAM and DDR + MIG

| Feature           | BRAM          | DDR + MIG       |
| ----------------- | ------------- | --------------- |
| Location          | Internal FPGA | External Memory |
| Capacity          | Small         | Large           |
| Speed             | Very Fast     | High Bandwidth  |
| Complexity        | Simple        | Complex         |
| Controller Needed | No            | Yes (MIG)       |

**Advantages of MIG**
  - Simplifies DDR interfacing
  - Automatically handles timing
  - Supports high-speed memory
  - Provides calibration logic
  - Saves FPGA development time
  - Easy integration in Vivado
  - Applications of MIG


# MIG in VIVADO 

While we use external DDR memory we need MIG to use as interfave between them.
          FPGA Logic  ↔  MIG  ↔  DDR3/DDR4 Chip

MIG setup in Vivado

### 1) Create new Vivado project 
Select the name, FPGA board 






