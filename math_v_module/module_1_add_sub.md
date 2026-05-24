    `timescale 1ns / 1ps

    (* keep_hierarchy = "yes" *)
    module add_sub (
        input [31:0] a,
        input [31:0] b,
        input op, 
        output [31:0] result
    );
        wire sign_a = a[31];
        wire sign_b = b[31] ^ op; 
        wire [7:0] exp_a = a[30:23];
        wire [7:0] exp_b = b[30:23];
        wire [23:0] mant_a = {1'b1, a[22:0]}; 
        wire [23:0] mant_b = {1'b1, b[22:0]};

        // --- Align Exponents ---
        wire [7:0] exp_diff = (exp_a >= exp_b) ? (exp_a - exp_b) : (exp_b - exp_a);
        wire [23:0] mant_a_shift = (exp_a >= exp_b) ? mant_a : (mant_a >> exp_diff);
        wire [23:0] mant_b_shift = (exp_a >= exp_b) ? (mant_b >> exp_diff) : mant_b;
        wire [7:0] exp_large = (exp_a >= exp_b) ? exp_a : exp_b;

        // --- Add/Sub Mantissas ---
        wire [24:0] mant_add = mant_a_shift + mant_b_shift;
        wire [24:0] mant_sub = (mant_a_shift >= mant_b_shift) ? 
                                                     (mant_a_shift - mant_b_shift) : 
                                                     (mant_b_shift - mant_a_shift);

        wire [24:0] mant_sum = (sign_a == sign_b) ? mant_add : mant_sub;
        wire sign_res_pre = (sign_a == sign_b) ? sign_a : (mant_a_shift >= mant_b_shift) ? sign_a : sign_b;

        // --- Normalize (Synthesizable Priority Encoder) ---
        reg [7:0] exp_res;
        reg [23:0] mant_res;
        (* mux_style = "muxf" *) reg [4:0] shift_amt;

        always @(*) begin
            if (mant_sum == 0) begin
                exp_res = 0;
                mant_res = 0;
            end else if (mant_sum[24]) begin // Overflow
                exp_res = exp_large + 1;
                mant_res = mant_sum[24:1];
            end else begin
                // Manual Priority Encoder (Faster than a for-loop for Synthesis)
                if      (mant_sum[23]) shift_amt = 0;
                else if (mant_sum[22]) shift_amt = 1;
                else if (mant_sum[21]) shift_amt = 2;
                else if (mant_sum[20]) shift_amt = 3;
                else if (mant_sum[19]) shift_amt = 4;
                else if (mant_sum[18]) shift_amt = 5;
                else if (mant_sum[17]) shift_amt = 6;
                else if (mant_sum[16]) shift_amt = 7;
                else if (mant_sum[15]) shift_amt = 8;
                else if (mant_sum[14]) shift_amt = 9;
                else if (mant_sum[13]) shift_amt = 10;
                else if (mant_sum[12]) shift_amt = 11;
                else if (mant_sum[11]) shift_amt = 12;
                else if (mant_sum[10]) shift_amt = 13;
                else if (mant_sum[9])  shift_amt = 14;
                else if (mant_sum[8])  shift_amt = 15;
                else if (mant_sum[7])  shift_amt = 16;
                else if (mant_sum[6])  shift_amt = 17;
                else if (mant_sum[5])  shift_amt = 18;
                else if (mant_sum[4])  shift_amt = 19;
                else if (mant_sum[3])  shift_amt = 20;
                else if (mant_sum[2])  shift_amt = 21;
                else if (mant_sum[1])  shift_amt = 22;
                else                   shift_amt = 23;

                if (exp_large >= shift_amt) begin
                    exp_res = exp_large - shift_amt;
                    mant_res = mant_sum[23:0] << shift_amt;
                end else begin
                    exp_res = 0;
                    mant_res = 0;
                end
            end
        end

        assign result = {sign_res_pre, exp_res, mant_res[22:0]};
    endmodule


The module extracts IEEE-754 fields from the inputs, aligns exponents, performs mantissa addition/subtraction, normalizes the result, and repacks it into 32-bit floating-point format.

![RTL/ block diagram ](rtl_mod1.png)


## Working/usage of module

Input 
| Signal | Width  | Description                 |
| ------ | ------ | --------------------------- |
| `a`    | 32-bit | First floating-point input  |
| `b`    | 32-bit | Second floating-point input |
| `op`   | 1-bit  | Operation select            |

Operation selection
| `op` Value | Operation Performed |
| ---------- | ------------------- |
| `0`        | `a + b`             |
| `1`        | `a - b`             |

Result will also be a 32 bit floating point output.

Example:
a  = 32'h40B00000; // 5.5
b  = 32'h40100000; // 2.25
op = 0;                                here op = 0 meaning addition.

Internal Working

### The module works in the following stages:
    -1. Extract sign, exponent, and mantissa from both inputs.
    -2. Compare exponents and align mantissas(such that the exponent will become the same number).
    -3. Perform mantissa addition or subtraction.
    -4. Normalize the result using a priority encoder(by shifting the mantissa and so as the exponent).
    -5.  Reconstruct the IEEE-754 formatted output.

This module compares the sign and implements it directly in the result. and operates by comparing thee magnitute of mantissa. If we have to subtract for A and B, if A is bigger then A-B and if B is bigger B-A will be the result with sign of the larger magnitude mantissa.

if both numbers are negative operation will be addition but the end sign will be -ve.
similarly if one of the sign is negative it will compare the mantissa value in standardise form and decide the output  sign.

## Advantages 
    - Simple floating-point arithmetic implementation
    - Fully synthesizable
    - Fast combinational architecture
    - Uses priority encoder for normalization
## Limitations
    -    Not fully IEEE-754 compliant
    - No rounding support
    - No NaN or Infinity handling
    - No denormal number support
    - Large combinational delay possible
    - No pipelining

## Testbench 
    `timescale 1ns / 1ps

    module tb_add_sub;

        reg  [31:0] a;
        reg  [31:0] b;
        reg  op;
        wire [31:0] result;

        add_sub uut (.a(a),.b(b),.op(op),.result(result));

        initial begin
            // Test 1 : 5.5 + 2.25 = 7.75
            a  = 32'h40B00000; // 5.5
            b  = 32'h40100000; // 2.25
            op = 0;
        
            #10;
        
            // Test 2 : 5.5 - 2.25 = 3.25
            // ---------------------------------------------------------
            a  = 32'h40B00000; // 5.5
            b  = 32'h40100000; // 2.25
            op = 1;

            #10;
 
            // Test 3 : -5.5 + 2.25 = -3.25
            // ---------------------------------------------------------
            a  = 32'hC0B00000; // -5.5
            b  = 32'h40100000; // 2.25
            op = 0;

            #10;

            // Test 4 : 10.0 + 10.0 = 20.0
            // ---------------------------------------------------------
            a  = 32'h41200000; // 10.0
            b  = 32'h41200000; // 10.0
            op = 0;

            #10;

            // Test 5 : 1.0 - 1.0 = 0
            // ---------------------------------------------------------
            a  = 32'h3F800000; // 1.0
            b  = 32'h3F800000; // 1.0
            op = 1;

            #10;

            // Test 6 : Small exponent difference
            // 100.0 + 1.5
            // ---------------------------------------------------------
            a  = 32'h42C80000; // 100.0
            b  = 32'h3FC00000; // 1.5
            op = 0;

            #10;
            $finish;

        end

    endmodule

The testbench tests:

    - Floating-point addition
    - Floating-point subtraction
    - Negative number handling
    - Zero result generation
    - Exponent alignment cases

Test cases used:

| Test Case   | Operation                            | Expected Result | IEEE-754 Hex |
| ----------- | ------------------------------------ | --------------- | ------------ |
| 5.5 + 2.25  | Addition                             | 7.75            | `40F80000`   |
| 5.5 - 2.25  | Subtraction                          | 3.25            | `40500000`   |
| -5.5 + 2.25 | Signed Addition                      | -3.25           | `C0500000`   |
| 10.0 + 10.0 | Addition with Overflow Normalization | 20.0            | `41A00000`   |
| 1.0 - 1.0   | Zero Result                          | 0               |    `0`       |
| 100.0 + 1.5 | Exponent Alignment                   | 101.5           | `42CB0000`   |

![simulation results](mod1_sim.png)


