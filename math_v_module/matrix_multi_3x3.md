    module MatrixMultiply3x3 (
        input clk,
        input [287:0] A,
        input [287:0] B,
        output [287:0] Result
    );

        wire [31:0] A_unpacked [0:2][0:2];
        wire [31:0] B_unpacked [0:2][0:2];
        reg [31:0] sum_final [0:2][0:2];

        genvar i, j;

        // ---------------------------------------------------------
        // 1. UNPACKING (Done separately to avoid multiple drivers)
        // ---------------------------------------------------------
        generate
            for (i = 0; i < 3; i = i + 1) begin : unpack_row
                for (j = 0; j < 3; j = j + 1) begin : unpack_col
                    // Map flattened input to 2D array ONCE
                    assign A_unpacked[i][j] = A[(8 - (i*3 + j)) * 32 +: 32];
                assign B_unpacked[i][j] = B[(8 - (i*3 + j)) * 32 +: 32];
                end
            end
        endgenerate

        // ---------------------------------------------------------
        // 2. CALCULATION (Pipelined Math Loop)
        // ---------------------------------------------------------
        generate
            for (i = 0; i < 3; i = i + 1) begin : row_calc
                for (j = 0; j < 3; j = j + 1) begin : col_calc
                
                    // Combinational wires
                    wire [31:0] mul0, mul1, mul2 ;
                    wire [31:0] sum01_wire, sum_final_wire ;
                  
                    // Pipeline Registers
                    reg [31:0] mul0_reg, mul1_reg, mul2_reg ;
                    reg [31:0] sum01_reg, mul2_delay_reg ;
                
                    // --- STAGE 1: Multiplication ---
                    fp32 u_mult0 (.a(A_unpacked[i][0]), .b(B_unpacked[0][j]), .product(mul0));
                    fp32 u_mult1 (.a(A_unpacked[i][1]), .b(B_unpacked[1][j]), .product(mul1));
                    fp32 u_mult2 (.a(A_unpacked[i][2]), .b(B_unpacked[2][j]), .product(mul2));
                
                    always @(posedge clk) begin
                        mul0_reg <= mul0;
                        mul1_reg <= mul1;
                        mul2_reg <= mul2;
                        sum01_reg      <= sum01_wire;
                        mul2_delay_reg <= mul2_reg; 
                        sum_final[i][j] <= sum_final_wire;
                    end
                
                    // --- STAGE 2: First Addition & Delay Matching ---
                    add_sub u_add0 (.a(mul0_reg), .b(mul1_reg), .op(1'b0), .result(sum01_wire));
                
                    // --- STAGE 3: Final Addition ---
                    (* keep_hierarchy = "yes" *) add_sub u_add1 (.a(sum01_reg), .b(mul2_delay_reg), .op(1'b0), .result(sum_final_wire));
                
                end
            end
        endgenerate

        // ---------------------------------------------------------
        // 3. PACKING RESULT
        // ---------------------------------------------------------
        assign Result = { 
                sum_final[0][0], sum_final[0][1], sum_final[0][2],
                sum_final[1][0], sum_final[1][1], sum_final[1][2],
                sum_final[2][0], sum_final[2][1], sum_final[2][2] 
        };
    endmodule

This module implements the matrix 3x3 multiplication  for Floating point 32 number.
Verilog cannot easily pass the 2D arrays across the module port easily the modules acepts the flatten 1D bit vector as input (3 x 3 x 32(int = 4 byte = 32 bits) = 288 bits.) converts into 2D matrix multiply and then then return a flatten result as output.

## Block Diagram


        UNPACKING LAYER              STAGE 1                     STAGE 2                     STAGE 3
     (Combinational Nets)       (Multiplication)           (First Addition)             (Final Addition)
 
       A_unpacked[i][0] ──┐
                          ├─► [ fp32_mult0 ] ── mul0 ──► [ mul0_reg ] ──┐
       B_unpacked[0][j] ──┘                                             ├──► [ add_sub_0 ] ── sum01_wire
                                                                        │
       A_unpacked[i][1] ──┐                                             │
                          ├─► [ fp32_mult1 ] ── mul1 ──► [ mul1_reg ] ──┘
       B_unpacked[1][j] ──┘
                                                                                          ┌──► [ sum_final[i][j] ]
                                                                                          │     (To Result Packing)
       A_unpacked[i][2] ──┐                                                               │
                          ├─► [ fp32_mult2 ] ── mul2 ──► [ mul2_reg ] ──► [ mul2_delay ] ─┼──► [ add_sub_1 ]
       B_unpacked[2][j] ──┘                                                                     sum_final_wire
                                                                                      


Here the flatten input is taken adn unpacked as 3X3 2D matrix using loops.
The martix we need is 3x3 so we assign a loop which start from 0,0 till 2,2 diving 32 bit that is one integer at each spot to form 2D matrix.

For calculation the basic logic of multiplying two 3x3 matrix is column to row multiplicaation.

    C[i][j] = A[i][0] × B[0][j] + A[i][1] × B[1][j] + A[i][2] × B[2][j]

Here we need multiplication of float so we can use f32 multiple module created earlier and 32 bit add_sub module for addition.
Hence we get final output which we need to pack again in 288 bit 1-D vector to give output.

for unpacking the input into matrix:
    
    generate
        for (i = 0; i < 3; i = i + 1) begin : unpack_row
            for (j = 0; j < 3; j = j + 1) begin : unpack_col
                // Map flattened input to 2D array ONCE
                assign A_unpacked[i][j] = A[(8 - (i*3 + j)) * 32 +: 32];
                assign B_unpacked[i][j] = B[(8 - (i*3 + j)) * 32 +: 32];
            end
        end
    endgenerate

Here loop is used to assign each position and maps it to the assigned bits.

| Matrix Coordinate | Vector Bit-Range |
| --- | --- |
| **Row 0, Col 0** | `[287:256]` |
| **Row 0, Col 1** | `[255:224]` |
| **Row 0, Col 2** | `[223:192]` |
| **Row 1, Col 0** | `[191:160]` |
| **Row 1, Col 1** | `[159:128]` |
| **Row 1, Col 2** | `[127:96]` |
| **Row 2, Col 0** | `[95:64]` |
| **Row 2, Col 1** | `[63:32]` |
| **Row 2, Col 2** | `[31:0]` |

Here after unpacking the matrix a loop is used to perform matrix multiplication of two matrices by using the multiplication f32 module and add_sub module used before. 

A pipeline structure is used bu having the multiplication of 3 elements of mat[A] and mat[B] and subsequently are added to form a whole element of resulting matrix.

Here after multiplying we have mult0, mult1 and mult2
which is stored in mult0_reg, mult1_reg and mult2_reg.
Initial addition is done between mult0_reg and mult1_reg and result is sum01_wire which is stored in sum01_reg meanwhile the sum2_reg is stored in mul2_delay_reg to avoid time lag and the second addition to be performed with some new value. This delay is necessery to allign all values.

the sum01_reg and mul2_delay_reg is added again using add_sub module to obtail final resulting element of resulting matrix.
sum_final_wire will store the result in sum_final_[i][j] 

This is repeated for each resulting element from [0][0] to [2][2].
Thus, we have our matrix multiplication of 3x3 matrices.

**How it works with clock:- **

At each Clock Edge, What Happens Inside the Hardware?

- **Start (Cycle 0)** 
    - You apply Matrix A and Matrix B to the input pins. All 27 multipliers instantly start working on the data combinationally.
- **Clock Edge 1** 
    - "Stage 1 Ends: The 27 products are captured into the first layer of registers (mul0_reg, mul1_reg, mul2_reg). The first adders instantly start working."
- **Clock Edge 2** 
    - Stage 2 Ends: The first addition results are captured into sum01_reg. The final adders instantly start working.
- **Clock Edge 3** 
    - Stage 3 Ends: The final addition completes. The 9 final answers are captured into sum_final[i][j] and instantly appear at the Result output pin.

## Advantages

**1. Parallel Computation**

- The design computes all 9 matrix elements simultaneously.
- Each element has its own:
    - multipliers
    - adders
    - pipeline registers
So operations happen in parallel. Very high throughput and fast computation.

**2. Pipelined Architecture**

The design uses pipeline stages:

- Stage	Operation
    1.	Multiplication
    2.	First addition
    3.	Final addition

- using registers:
    - mul0_reg
    - sum01_reg
    - mul2_delay_reg

Breaks long combinational paths. Better FPGA performance

**3. Modular Design**
Uses separate reusable modules:
- fp32
- add_sub
- Advantage
Easy to debug and use it independently.

**4. Generate Loop Based**

it Uses:
generate
for(...)
Advantage

Code becomes scalable and compact. Instead of manually writing 9 matrix computations.

**5. Proper Timing Alignment**

This line:

        mul2_delay_reg <= mul2_reg;

synchronizes pipeline stages and prevents incorrect data mixing between stages.

## Limitations 

**1. Very High Hardware Usage**

Total: 
- FP Multipliers	27
- FP Adders	18
  
are used which consumes large FPGA resources:

- DSP slices
- LUTs
- flip-flops

**2. High Power Consumption**

- Since all units operate simultaneously power usage becomes high.
- Not ideal for low-power systems.

**3. Increased Latency**

- Pipeline introduces delay. Output does NOT appear immediately.

Example:
    - Result may appear after **3 clock cycles**


**4. No Overflow/Exception Handling**

Current code uses fp32 and add_sub modules which does not explicitly handle:

    - NaN
    -Infinity
    - overflow
    - underflow
    - denormal numbers
    - Limitation

May produce incorrect IEEE-754 edge-case behavior.

**5. Scaling Problem**

- For NxN matrix hardware grows rapidly.

Example

For 8×8 Operation Count

    - Multipliers	512
    - Adders	Huge
    
Design becomes impractical for large matrices.


## Testbench 
        `timescale 1ns / 1ps

        module tb_MatrixMultiply3x3;

            reg clk;
            reg  [287:0] A;
            reg  [287:0] B;
            wire [287:0] Result;

            MatrixMultiply3x3 uut (.clk(clk),.A(A),.B(B),.Result(Result)
            );

            initial begin
                clk = 0;
                forever #5 clk = ~clk;
            end


            // MONITOR
            // =========================================================
            initial begin
                $monitor("TIME=%0t RESULT=%h", $time, Result);
            end

            // =========================================================
            // TEST CASES

            initial begin
    
            // TEST 1 : Identity Matrix × Matrix
            // Expected Result = B


                A = {
                    32'h3F800000, // 1
                    32'h00000000, // 0
                    32'h00000000, // 0

                    32'h00000000, // 0
                    32'h3F800000, // 1
                    32'h00000000, // 0

                    32'h00000000, // 0
                    32'h00000000, // 0
                    32'h3F800000  // 1
                };

                B = {
                    32'h3F800000, // 1
                    32'h40000000, // 2
                    32'h40400000, // 3

                    32'h40800000, // 4
                    32'h40A00000, // 5
                    32'h40C00000, // 6

                    32'h40E00000, // 7
                    32'h41000000, // 8
                    32'h41100000  // 9
                };

                #80;

            // TEST 2 : Zero Matrix × Matrix
            // Expected Result = Zero Matrix
                A = {
                    32'h00000000,
                    32'h00000000,
                    32'h00000000,

                    32'h00000000,
                    32'h00000000,
                    32'h00000000,

                    32'h00000000,
                    32'h00000000,
                    32'h00000000
                };

                B = {
                    32'h3F800000, // 1
                    32'h40000000, // 2
                    32'h40400000, // 3

                    32'h40800000, // 4
                    32'h40A00000, // 5
                    32'h40C00000, // 6

                    32'h40E00000, // 7
                    32'h41000000, // 8
                    32'h41100000  // 9
                };

                #80;

            // TEST 3 : Normal Matrix × Matrix
    
                // Matrix A
                // [1 2 3]
                // [4 5 6]
                // [7 8 9]

                A = {
                    32'h3F800000, // 1
                    32'h40000000, // 2
                    32'h40400000, // 3

                    32'h40800000, // 4
                    32'h40A00000, // 5
                    32'h40C00000, // 6

                    32'h40E00000, // 7
                    32'h41000000, // 8
                    32'h41100000  // 9
                };

                // Matrix B
                // [5 3 4]
                // [2 6 1]
                // [9 7 8]

                B = {
                    32'h40A00000, // 5
                    32'h40400000, // 3
                    32'h40800000, // 4

                    32'h40000000, // 2
                    32'h40C00000, // 6
                    32'h3F800000, // 1

                    32'h41100000, // 9
                    32'h40E00000, // 7
                    32'h41000000  // 8
                };

                #80;
        
            // TEST 4 : Positive × Negative Numbers

                // Matrix A
                // [-1 -2 -3]
                // [ 4  5  6]
                // [ 7  8  9]

                A = {
                    32'hBF800000, // -1
                    32'hC0000000, // -2
                    32'hC0400000, // -3

                    32'h40800000, // 4
                    32'h40A00000, // 5
                    32'h40C00000, // 6

                    32'h40E00000, // 7
                    32'h41000000, // 8
                    32'h41100000  // 9
                };

                // Matrix B
                // [ 1  2  3]
                // [-4 -5 -6]
                // [ 7  8  9]

                B = {
                    32'h3F800000, // 1
                    32'h40000000, // 2
                    32'h40400000, // 3

                    32'hC0800000, // -4
                    32'hC0A00000, // -5
                    32'hC0C00000, // -6

                    32'h40E00000, // 7
                    32'h41000000, // 8
                    32'h41100000  // 9
                };

                #80;

                $finish;

            end

        endmodule



Output
<img width="1626" height="228" alt="image" src="https://github.com/user-attachments/assets/88789c5e-dabf-40e7-8e93-aa88c0d072e4" />


