    module MatrixVectorMultiply4x4 (
        input clk,
        input [511:0] T,        // 16x32-bit packed 4x4 matrix
        input [127:0] pt,      // 4x32-bit packed 4x1 vector (x,y,z,1)
        output reg [127:0] result // 4x32-bit packed output (x',y',z',w')
    );
        // Unpacked matrix and vector
        wire [31:0] T1 [0:3][0:3];
        wire [31:0] pt1 [0:3];
    
        // Wire to hold the final combinational sum before the output register
        wire [31:0] sum_final_wire [0:3];

        genvar ii;
        generate
            for (ii = 0; ii < 4; ii = ii + 1) begin : row_gen
                // Combinational wires
                wire [31:0] mul0_w, mul1_w, mul2_w, mul3_w;
                wire [31:0] sum01_w, sum23_w;

                // Pipeline Registers
                reg [31:0] mul0_reg, mul1_reg, mul2_reg, mul3_reg;
                reg [31:0] sum01_reg, sum23_reg;

                // Unpack logic (Remains the same)
                assign T1[ii][0] = T[(15 - (ii*4 + 0))*32 +: 32];
                assign T1[ii][1] = T[(15 - (ii*4 + 1))*32 +: 32];
                assign T1[ii][2] = T[(15 - (ii*4 + 2))*32 +: 32];
                assign T1[ii][3] = T[(15 - (ii*4 + 3))*32 +: 32];

                assign pt1[0] = pt[(3 - 0)*32 +: 32];
                assign pt1[1] = pt[(3 - 1)*32 +: 32];
                assign pt1[2] = pt[(3 - 2)*32 +: 32];
                assign pt1[3] = pt[(3 - 3)*32 +: 32];
  
                // --- STAGE 1: Multiplication ---
                fp32 u_mult0 (.a(T1[ii][0]), .b(pt1[0]), .product(mul0_w));
                fp32 u_mult1 (.a(T1[ii][1]), .b(pt1[1]), .product(mul1_w));
                fp32 u_mult2 (.a(T1[ii][2]), .b(pt1[2]), .product(mul2_w));
                fp32 u_mult3 (.a(T1[ii][3]), .b(pt1[3]), .product(mul3_w));

                always @(posedge clk) begin
                    mul0_reg <= mul0_w;
                    mul1_reg <= mul1_w;
                    mul2_reg <= mul2_w;
                    mul3_reg <= mul3_w;
                end

                // --- STAGE 2: First Addition Layer ---
                add_sub u_add0 (.a(mul0_reg), .b(mul1_reg), .op(1'b0), .result(sum01_w));
                add_sub u_add1 (.a(mul2_reg), .b(mul3_reg), .op(1'b0), .result(sum23_w));

                always @(posedge clk) begin
                    sum01_reg <= sum01_w;
                    sum23_reg <= sum23_w;
                end

                // --- STAGE 3: Final Addition Layer ---
                add_sub u_add2 (.a(sum01_reg), .b(sum23_reg), .op(1'b0), .result(sum_final_wire[ii]));
            end
        endgenerate

        // --- STAGE 4: Output Register ---
        // Register the final result on the clock edge
        always @(posedge clk) begin
            result <= { sum_final_wire[0], sum_final_wire[1], sum_final_wire[2], sum_final_wire[3] };
        end
    endmodule

In this  module we are multiplying a 4x4 matrix with a 4x1 matrix.
It is similar to matrix multiplication of 3x3 module.

The matrix-vector multiplication is:

        [a1 a2 a3 a4]       [x]      [xr]
        [b1 b2 b3 b4]   X   [y]   =  [yr]
        [c1 c2 c3 c4]       [z]      [zr]
        [d1 d2 d3 d4]       [w]      [wr]

example of one output elemment: xr = a1*x + a2*y + a3*z + a4w
similarly, yr zr wr are obtained

                 ┌───────────────────────────┐      ┌───────────────────────────┐             
                 │   Packed Matrix T[511:0]  │      │   Packed Vector pt[127:0] │               
                 └─────────────┬─────────────┘      └─────────────┬─────────────┘
                               │                                  |
                        UNPACK MATRIX                        UNPACK VECTOR
                               │                                  |
                 ┌─────────────▼─────────────┐      ┌─────────────▼─────────────┐
                 │      T1[4][4] Matrix      │      │        pt1[4] Vector      │
                 └───────────────────────────┘      └───────────────────────────┘

         
        ┌────────────────────────────────────────────┐
        │               ROW COMPUTE UNIT             │
        │                                            │
        │  T[i][0] × pt[0] → mul0                    │
        │  T[i][1] × pt[1] → mul1                    │
        │  T[i][2] × pt[2] → mul2                    │
        │  T[i][3] × pt[3] → mul3                    │
        │                                            │
        │            PIPELINE REGISTER               │
        │                                            │
        │      mul0+mul1 → sum01                     │
        │      mul2+mul3 → sum23                     │
        │                                            │
        │            PIPELINE REGISTER               │
        │                                            │
        │      sum01 + sum23 → final_sum             │
        └────────────────────────────────────────────┘


                 ┌───────────────────────────┐
                 │     OUTPUT REGISTER       │
                 │ result[127:0]             │
                 └───────────────────────────┘

        T[i][0] ──×──┐
                     │
        pt[0] ───────┘

        T[i][1] ──×──┐
                     ├── + ── sum01
        pt[1] ───────┘


        T[i][2] ──×──┐
                     │
        pt[2] ───────┘

        T[i][3] ──×──┐
                     ├── + ── sum23
        pt[3] ───────┘


              sum01 ───┐
                        ├── + ── final result
              sum23 ───┘


The input is taken as a flatten matrix which is then unpacked then made aa 2-d matrix of 4x4 and 4x1.
for 4x4 the input is 512 bits, 1 integer will take 4 byte = 32 bits and 32 x 4 x4 = 512. similarly then 4x1 will have 4x1x32 = 128 bits.

Once the input is taken then it is unpacked in 32 bit pack as a element using loop.

        assign T1[ii][0] = T[(15 - (ii*4 + 0))*32 +: 32];
        assign T1[ii][1] = T[(15 - (ii*4 + 1))*32 +: 32]; 
        assign T1[ii][2] = T[(15 - (ii*4 + 2))*32 +: 32]; 
        assign T1[ii][3] = T[(15 - (ii*4 + 3))*32 +: 32];
        
here ii is the variable therefore the it will cover from [0][0] to [4][4]. 
for [0][0] the bit selected are 511 to 480 similarly for [3][3] it is 31 to 0.

|511................480|479................448| ... |31.............0|
|        T00           |         T01          | ... |      T33       |


| Matrix Element | Linear Index | Bit Range  |
| -------------- | ------------ | ---------- |
| T1[0][0]       | 0            | T[511:480] |
| T1[0][1]       | 1            | T[479:448] |
| T1[0][2]       | 2            | T[447:416] |
| T1[0][3]       | 3            | T[415:384] |
| T1[1][0]       | 4            | T[383:352] |
| T1[1][1]       | 5            | T[351:320] |
| T1[1][2]       | 6            | T[319:288] |
| T1[1][3]       | 7            | T[287:256] |
| T1[2][0]       | 8            | T[255:224] |
| T1[2][1]       | 9            | T[223:192] |
| T1[2][2]       | 10           | T[191:160] |
| T1[2][3]       | 11           | T[159:128] |
| T1[3][0]       | 12           | T[127:96]  |
| T1[3][1]       | 13           | T[95:64]   |
| T1[3][2]       | 14           | T[63:32]   |
| T1[3][3]       | 15           | T[31:0]    |

Same process is for the 4x1 martix with only 4 element so it divides easily.
| pt[0]       | pt[127:96] |
| pt[1]       | pt[95:64]  |
| pt[2]       | pt[63:32]  |
| pt[3]       | pt[31:0]   |

After we have both the matrices we use the multiplication f32 module to have row to column multiplication. of each element hence we will have 4 products mult0, mult1, mult2, mult3.

which will be saved in sum0_0, sum1_reg, sum2_reg, and sum3_reg to store value and have pipelined structure. 

It is followed by addition of all elements. Since addition of 4 element is difficult we can perform addition 3 times. [(mult0_reg + mult1_reg) + (mult2_reg + mult3_reg)]

mult0_reg + mult1_reg = sum01 which will be saved in sum01_reg & 
mult2_reg + mult3_reg = sum23 which will be saved in sum01_reg 

Final addition will be done. sum01_reg + sum23_reg to have out sum_final_wire 
It will be transferred to result, thus we will obtain out 4x4 * 4x1 matirix multiplication.

Here we have not used (((mult0+mult1)+mult2)+mult3) other wise we need to shift or delay mult2 one time and mult3 two times to allign all values for operation and avoid any value is replaced by any new value.

## Advantages 

**1. High speed parallel processing**
- all 4 rows are computed simultaneously.

**2. Piplined Architecture**
- Faster processing and no delay.

**3. Floating point supported**
- using module f32 and add_sub and using them in loop speeds up process.

## Limitation
**1. FPGA resources ussed in lareg quntity.**
- 16 multipliers and 12 adders are used.

**2. high power consumption**
- many parallel blocks active simultaneouly.

## Testbench 

`timescale 1ns / 1ps

    module tb_MatrixVectorMultiply4x4;

        reg clk;
        reg  [511:0] T;
        reg  [127:0] pt;
    
        wire [127:0] result;

        MatrixVectorMultiply4x4 uut (.clk(clk),.T(T),.pt(pt),.result(result));

        // CLOCK GENERATION

        initial begin
            clk = 0;
            forever #5 clk = ~clk;
        end

        // TEST CASES
        // =========================================================
        initial begin

        // TEST 1 : Identity Matrix × Vector
        // Expected Result = Same Vector

            // Identity Matrix

            T = {

                32'h3F800000, 32'h00000000, 32'h00000000, 32'h00000000,
                32'h00000000, 32'h3F800000, 32'h00000000, 32'h00000000,
                32'h00000000, 32'h00000000, 32'h3F800000, 32'h00000000,
                32'h00000000, 32'h00000000, 32'h00000000, 32'h3F800000

            };

            // Vector [1 2 3 1]

            pt = {
                32'h3F800000, // 1
                32'h40000000, // 2
                32'h40400000, // 3
                32'h3F800000  // 1
            };

            #100;

        // TEST 2 : Zero Matrix × Vector
        // Expected Result = Zero Vector

            T = 0;

            pt = {
                32'h3F800000,
                32'h40000000,
                32'h40400000,
                32'h3F800000
            };
    
            #100;

        // TEST 3 : Normal Matrix × Vector

            // Matrix
            //
            // [1  2  3  4 ]
            // [5  6  7  8 ]
            // [9 10 11 12 ]
            // [13 14 15 16]

            T = {

                32'h3F800000, // 1
                32'h40000000, // 2
                32'h40400000, // 3
                32'h40800000, // 4

                32'h40A00000, // 5
                32'h40C00000, // 6
                32'h40E00000, // 7
                32'h41000000, // 8

                32'h41100000, // 9
                32'h41200000, // 10
                32'h41300000, // 11
                32'h41400000, // 12

                32'h41500000, // 13
                32'h41600000, // 14
                32'h41700000, // 15
                32'h41800000  // 16
            };

            // Vector [1 2 3 4]

            pt = {

                32'h3F800000, // 1
                32'h40000000, // 2
                32'h40400000, // 3
                32'h40800000  // 4
            };

            #100;

        // TEST 4 : Positive and Negative Numbers

            // Matrix
            //
            // [-1 -2 -3 -4]
            // [ 5  6  7  8]
            // [-9 -10 -11 -12]
            // [13 14 15 16]

            T = {
    
                32'hBF800000, // -1
                32'hC0000000, // -2
                32'hC0400000, // -3
                32'hC0800000, // -4

                32'h40A00000, // 5
                32'h40C00000, // 6
                32'h40E00000, // 7
                32'h41000000, // 8

                32'hC1100000, // -9
                32'hC1200000, // -10
                32'hC1300000, // -11
                32'hC1400000, // -12

                32'h41500000, // 13
                32'h41600000, // 14
                32'h41700000, // 15
                32'h41800000  // 16
            };

            // Vector [1 -2 3 -4]

            pt = {

                32'h3F800000, // 1
                32'hC0000000, // -2
                32'h40400000, // 3
                32'hC0800000  // -4
            };

            #100;

            $finish;

        end

    endmodule

    
Output

<img width="1134" height="206" alt="image" src="https://github.com/user-attachments/assets/cf408bc4-50a9-47f4-8336-fd8002196168" />
