          module Cordic(
        master_clk,angle,Xin,Yin,Xout,Yout
    );
      parameter BW=32;
        localparam iter=BW;
        input master_clk;
        input signed [31:0] angle;
        input signed [BW-1:0] Xin;
        input signed [BW-1:0] Yin;
        output signed [BW:0] Xout;
        output signed [BW:0] Yout;
        wire signed [31:0] taninv[0:30];
        //reg taninv [31:0] [0:30];
        assign taninv[0] = 32'b00100000000000000000000000000000;
        assign taninv[1] = 32'b00010010111001000000010100011110;
        assign taninv[2] = 32'b00001001111110110011100001011011;
        assign taninv[3] = 32'b00000101000100010001000111010100;
        assign taninv[4] = 32'b00000010100010110000110101000011;
        assign taninv[5] = 32'b00000001010001011101011111100001;
        assign taninv[6] = 32'b00000000101000101111011000011110;
        assign taninv[7] = 32'b00000000010100010111110001010101;
        assign taninv[8] = 32'b00000000001010001011111001010011;
        assign taninv[9] = 32'b00000000000101000101111100101111;
        assign taninv[10] = 32'b00000000000010100010111110011000;
        assign taninv[11] = 32'b00000000000001010001011111001100;
        assign taninv[12] = 32'b00000000000000101000101111100110;
        assign taninv[13] = 32'b00000000000000010100010111110011;
        assign taninv[14] = 32'b00000000000000001010001011111010;
        assign taninv[15] = 32'b00000000000000000101000101111101;
        assign taninv[16] = 32'b00000000000000000010100010111110;
        assign taninv[17] = 32'b00000000000000000001010001011111;
        assign taninv[18] = 32'b00000000000000000000101000110000;
        assign taninv[19] = 32'b00000000000000000000010100011000;
        assign taninv[20] = 32'b00000000000000000000001010001100;
        assign taninv[21] = 32'b00000000000000000000000101000110;
        assign taninv[22] = 32'b00000000000000000000000010100011;
        assign taninv[23] = 32'b00000000000000000000000001010001;
        assign taninv[24] = 32'b00000000000000000000000000101001;
        assign taninv[25] = 32'b00000000000000000000000000010100;
        assign taninv[26] = 32'b00000000000000000000000000001010;
        assign taninv[27] = 32'b00000000000000000000000000000101;
        assign taninv[28] = 32'b00000000000000000000000000000011;
        assign taninv[29] = 32'b00000000000000000000000000000001;
        assign taninv[30] = 32'b00000000000000000000000000000001;
        reg signed [BW:0] X [0:iter-1];
        reg signed [BW:0] Y [0:iter-1];
        reg signed [31:0] Z [0:iter-1];
        wire [1:0] quadrant;
        assign quadrant=angle[31:30];
        always @(posedge master_clk)
        begin
            case(quadrant)
                2'b00,2'b11: begin
                    X[0]<=Xin;
                    Y[0]<=Yin;
                    Z[0]<=angle;
                end
                2'b01: begin
                    X[0] <= -Yin;
                    Y[0] <= Xin;
                    Z[0] <= {2'b00,angle[29:0]};
                end
                2'b10: begin
                    X[0] <= Yin;
                    Y[0] <= -Xin;
                    Z[0] <= {2'b11,angle[29:0]};
                end
            endcase
        end

        genvar i;
        generate
            for(i=0;i<(iter-1);i=i+1) 
                begin: XYZ
                wire Z_sign;
                wire [BW:0] X_shr,Y_shr;
                assign X_shr=X[i]>>>(i);
                assign Y_shr=Y[i]>>>(i);
                assign Z_sign=Z[i][31];
                    always @(posedge master_clk) begin
                        X[i+1] <= Z_sign ? (X[i] + Y_shr) : (X[i] - Y_shr);
                        Y[i+1] <= Z_sign ? (Y[i] - X_shr) : (Y[i] + X_shr);
                        Z[i+1] <= Z_sign ? (Z[i] + taninv[i]) : (Z[i] - taninv[i]);
                    end  
                end
        endgenerate
        assign Xout=X[iter-1];
        assign Yout=Y[iter-1];
    endmodule

This module implements the CORDIC algorithm in rotational mode to rotate the vector.
Takes input xin yin and a angle to which it will be rotatted and gives output xout yout.

It used add_sub modules and shift operation for this process instead of multipliers because it uses large FPGA resources.

Mathematically it is

    Xout = Xin​cos(θ) − Yin​sin(θ)
    ​Yout​​ = Xin​sin(θ) + Yin​cos(θ)
    

# RTL/Block diagram

                    +------------------+
    Xin ----------->|                  |
    Yin ----------->| Quadrant Adjust  |
    angle --------->|                  |
                    +---------+--------+
                              |
                              v
                    +------------------+
                    | Stage 0          |
                    | Shift/Add/Sub    |
                    +---------+--------+
                              |
                              v
                    +------------------+
                    | Stage 1          |
                    | Shift/Add/Sub    |
                    +---------+--------+
                              |
                              v
                             ...
                              |
                              v
                    +------------------+
                    | Stage 30         |
                    | Shift/Add/Sub    |
                    +---------+--------+
                              |
                     +--------+--------+
                     |                 |
                     v                 v
                   Xout              Yout

                   

    Input Vector
           ↓
    Quadrant Correction
           ↓
    Repeated Small Rotations
    (using shifts + add/sub)
           ↓
    Residual angle becomes 0
           ↓
    Output Rotated Vector

This module uses 3 arrats 
- x[i] --> x value at stage `i`
- y[i] --> y value at stage `i`
- z[i] --> Angle remaining after stage `i`

Quadrant is checked using 

          quadrant= angle[31:30]

Depending on the quadrant:

- the input vector may be pre-rotated
- the angle may be adjusted

At every stage:

- The sign of the remaining angle Z[i] is checked.
- The vector is rotated slightly clockwise or anticlockwise.
- The remaining angle is reduced.

Shift operatiin us used to division by powers of 2 instead of using multiplier with 2^-i.

For every iteration:

- If angle is positive:

  - Xnew = X - Yshift
  - Ynew = Y + Xshift
  - Znew = Z - angle_constant

- If angle is negative:

  - Xnew = X + Yshift
  - Ynew = Y - Xshift
  - Znew = Z + angle_constant

This gradually rotates the vector toward the target angle.

### Pipeline Operation

The design is pipelined.

Each stage performs:

one micro rotation & 
one angle update

After every clock cycle:

data moves to the next stage.

Latency is equal to the number of stages.

In this design:

32 stages
32 clock cycle latency

### Example 

If input is (1,0) and rotation angle is 45

final output should be approx (0.707,0.707), absolute value by matematical formula.


`Stage 0 : `
Step 1: check angle sign.
x0 = +45
positive angle, positive direction rotation.

step 2:
for stage 0 shift amt = 2^-0 = 1

x_shift = x0 >> 0
y_shift = y0 >> 0

cordic equation:

x1= x0 - y_shift
y1= y0 + x_shift
z1= z0 - atan(1)   [atan(1) = 45]

updated values:

x1 = 1
y1 = 1
z1 = 0

`Stage 1 :`
Step 1: Shift operation 
for stage 1 shift amt = 2^-1 = 0.5

x_shift = x0 >> 1
y_shift = y0 >> 1

x_shift = 0.5 
y_shift = 0.5

cordic equaltions:

x2= x1 - y_shift
y2= y1 + x_shift
z2= z1 - atan(0.5)  

substituting values 

x2= 1 - 0.5 = 0.5
y2= 1 + 0.5 = 1.5
z2= 0 - 26.565

updated values:

x2 = 0.5 
y2 = 1.5
z2 = -26.565

here z2 becomes nagtive so now the angle will change direction of rotation from positive to negative.
negative z2 shows that the vector was rotated little too much.

`Stage 2 :`
Step 1: Shift operation 
for stage 2 shift amt = 2^-2 = 0.25

x_shift = x0 >> 2
y_shift = y0 >> 2

x_shift = 0.125
y_shift = 0.375

cordic equaltions:

angle is negative opposite rotation,

x2= x1 + y_shift
y2= y1 - x_shift
z2= z1 + atan(0.5)  

substituting values 

x2= 0.5 + 0.375 = 0.875
y2= 1.5 - 0.125 = 1.375
z2= -26.565 + 14.036 = -12.529

updated values:

x2 = 0.875
y2 = 1.375
z2 = -12.529

again the angle is -vee but magnitude is reduced representing it is converging towards the target.
similarly this iterative proces will repeat, shift angle will become smaller, correction angle becomes smaller and the output becomes more accurate.

| Iteration | Shift Value (2^-i) | Rotation Angle (deg) | X Value | Y Value | Remaining Angle Z |
|---|---|---|---|---|---|
| Initial | - | - | 1.0 | 0.0 | 45.0 |
| 0 | 1 | 45.0 | 1.0 | 1.0 | 0.0 |
| 1 | 0.5 | 26.5650511771 | 0.5 | 1.5 | -26.5650511771 |
| 2 | 0.25 | 14.0362434679 | 0.875 | 1.375 | -12.5288077092 |
| 3 | 0.125 | 7.1250163489 | 1.046875 | 1.265625 | -5.4037913602 |
| 4 | 0.0625 | 3.576334375 | 1.1259765625 | 1.2001953125 | -1.8274569853 |
| 5 | 0.03125 | 1.7899106082 | 1.163482666 | 1.1650085449 | -0.037546377 |
| 6 | 0.015625 | 0.8951737102 | 1.1816859245 | 1.1468291283 | 0.8576273332 |
| 7 | 0.0078125 | 0.4476141709 | 1.172726322 | 1.1560610496 | 0.4100131623 |
| 8 | 0.00390625 | 0.2238105004 | 1.1682104585 | 1.1606420117 | 0.186202662 |
| 9 | 0.001953125 | 0.1119056771 | 1.1659435796 | 1.1629236728 | 0.0742969849 |
| 10 | 0.0009765625 | 0.0559528919 | 1.1648079119 | 1.1640622896 | 0.018344093 |
| 11 | 0.00048828125 | 0.0279764526 | 1.1642395221 | 1.1646310434 | -0.0096323596 |
| 12 | 0.000244140625 | 0.0139882271 | 1.1645238559 | 1.1643468053 | 0.0043558675 |
| 13 | 0.0001220703125 | 0.0069941137 | 1.1643817237 | 1.1644889591 | -0.0026382461 |
| 14 | 6.103515625e-05 | 0.0034970569 | 1.1644527985 | 1.1644178908 | 0.0008588107 |
| 15 | 3.0517578125e-05 | 0.0017485284 | 1.1644172632 | 1.1644534271 | -0.0008897177 |
| 16 | 1.52587890625e-05 | 0.0008742642 | 1.1644350314 | 1.1644356595 | -1.54535e-05 |
| 17 | 7.62939453125e-06 | 0.0004371321 | 1.1644439153 | 1.1644267756 | 0.0004216786 |
| 18 | 3.814697265625e-06 | 0.0002185661 | 1.1644394734 | 1.1644312176 | 0.0002031126 |
| 19 | 1.9073486328125e-06 | 0.000109283 | 1.1644372524 | 1.1644334386 | 9.38295e-05 |
| 20 | 9.5367431640625e-07 | 5.46415e-05 | 1.1644361419 | 1.1644345491 | 3.9188e-05 |
| 21 | 4.76837158203125e-07 | 2.73208e-05 | 1.1644355867 | 1.1644351043 | 1.18673e-05 |
| 22 | 2.384185791015625e-07 | 1.36604e-05 | 1.1644353091 | 1.1644353819 | -1.7931e-06 |
| 23 | 1.1920928955078125e-07 | 6.8302e-06 | 1.1644354479 | 1.1644352431 | 5.0371e-06 |
| 24 | 5.960464477539063e-08 | 3.4151e-06 | 1.1644353785 | 1.1644353125 | 1.622e-06 |
| 25 | 2.9802322387695312e-08 | 1.7075e-06 | 1.1644353438 | 1.1644353472 | -8.56e-08 |
| 26 | 1.4901161193847656e-08 | 8.538e-07 | 1.1644353611 | 1.1644353299 | 7.682e-07 |
| 27 | 7.450580596923828e-09 | 4.269e-07 | 1.1644353524 | 1.1644353386 | 3.413e-07 |
| 28 | 3.725290298461914e-09 | 2.134e-07 | 1.1644353481 | 1.1644353429 | 1.279e-07 |
| 29 | 1.862645149230957e-09 | 1.067e-07 | 1.1644353459 | 1.1644353451 | 2.12e-08 |
| 30 | 9.313225746154785e-10 | 5.34e-08 | 1.1644353449 | 1.1644353462 | -3.22e-08 |
| 31 | 4.656612873077393e-10 | 2.67e-08 | 1.1644353454 | 1.1644353456 | -5.5e-09 |







