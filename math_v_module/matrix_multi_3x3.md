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


