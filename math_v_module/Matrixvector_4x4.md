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

