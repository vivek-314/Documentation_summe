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
  - Select the name, FPGA board.
### 2) Open IP catalog 
  -Tools &rarr IP Catalog (search Memory Interface Generator) &rarr Double click MIG PI
### 3) Start MIG Configuration Wizard
  - MIG wizard opens.
  - Select Memory Type (eg. DDR3)
### 4) Select Controller Options
  - Choose:
      - Data width (eg. 16-bit DDR3)
      - Frequency (eg. 800 MT/s)
      - Burst length (eg. BL8 burst)
### 5) Select FPGA pin configuration
  - It needs 
      - memory pins
      - clocks
      - reset pins
  - Vivado may autofill constraints else manually assign pins.
    eg. ddr3_dq
        ddr3_addr
        ddr3_ba
        ddr3_clk
        ddr3_dqs
### 6) Choose input clock 
  - eg. 100 MHz system clock
  - MIG internally generates:
      - memory clock
      - reference clocks
      - PHY clocks
### 7) Select interface type 
#### A) Native interface
It is low level interface directly connected to the MIG controller.
It manually send:
  - addresses
  - commands
  - write data 
  - read requests
to MIG
### Command signals:
  - app_addr
  - app_cmd
  - app_en
### Write Data Signals:
  - app_wdf_data
  - app_wdf_wren
### Write workflow
    User Logic
        ↓
    Provide Address
        ↓
    Provide Command
        ↓
    Provide Write Data
        ↓
    MIG writes to DDR
  
### Read data signals:
  - app_rd_data
  - app_rd_data_valid
### Read workflow
      User Logic
          ↓
    Send Read Command
          ↓
    MIG accesses DDR
          ↓
    Read Data Returned

### Advantages of Native Interface
  - More control over DDR operations
  - Lower overhead
  - Can achieve high performance
  - Useful for custom memory controllers
### Disadvantages of Native Interface
  - More complex
  - User handles handshaking manually
  - Harder timing management
  - Difficult for beginners

#### B) AXI Interface (Advanced eXtensible Interface)
It makes DDR access much easier.
##### AXI Achitecture
    Processor / DMA / IP
          ↓
      AXI Bus
          ↓
         MIG
          ↓
      DDR Memory
### Write address channel
  - awaddr
  - awvalid
  - awready
### Write Data channel 
  - wdata
  - wvalid
  - wready
### Read data channel 
  - rdata
  - rvalid
  - rready

### Advantages of AXI Interface
  - Easy integration
  - Standard protocol
  - Compatible with many IPs
  - Easier debugging
  - Better scalability
  - Supported by Vivado IP Integrator
### Disadvantages of AXI Interface
  - Slightly higher overhead
  - Less low-level control
  - More logic resources used

### Which Interface Should Be Used?
  - Use Native Interface When:
    - designing custom high-speed systems
    - maximum performance is needed
    - fine memory control required
  - Use AXI Interface When:
    - using processors
    - connecting standard IP blocks
    - beginner-friendly design needed
    - rapid development required
      
### 8) Generate Output codes
  - Vivado will generate
      - HDL
      - simulation models
      - constraints
      - calibration modules
### 9) Add MIG to Block Design
  - Open IP integrator and add:
      - MIG
      - clock wizard
      - processor/IP  
it will connect the interface

## Important MIG signals
Clock : 
  - sys_clk_i
  - clk_ref_i
Reset :
  - sys_rst
DDR physical signals
  - ddr3_dq
  - ddr3_dqs_p
  - ddr3_dqs_n
  - ddr3_addr
  - ddr3_ba
User interface signals
  - Write signals
    - app_addr
    - app_cmd
    - app_en

    - app_wdf_data
    - app_wdf_wren

  - Read Signals
    - app_rd_data
    - app_rd_data_valid
   
An important signal = **init_calib_complete**, if it is high it indicates DDR is ready and it is safe to write/read.

## Basic write operation

    app_addr <= 32'h000000010;            provides address
    app_cmd <= 3'b000;                    gives write command here 000 indicates write
    app_wdf_data <= 120'hAABBCCDDA        data given to write\
    app_en <= 1'b1                        it enables the write 
    app_wdf_wren <= 1'b1                  and stores data into ddr 

## Basic read operation

    app_addr <= 32'h000000010;            provides address
    app_cmd <= 3'b001;                    gives read command
    app_rd_data_valid                     waitign for valid signal
    app_rd_data                           Reads data

### How MIG works
        Camera
          ↓
    FPGA Processing
          ↓
         MIG
          ↓
      DDR3 Memory
          ↓
    Frame Buffer Storage

To avoid errors, always wait for **init_calib_complete == 1** 












