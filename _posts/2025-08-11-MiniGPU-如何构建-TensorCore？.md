本文是 miniGPU 系列的第二篇文章，介绍我们设计的 TensorCore。TensorCore 的主要作用是 在一个周期内实现 FMA 运算。也就是一个乘加运算（$D=AB+C$），我们的设计中，TensorCore 处理的是 $4\times 4$ 的矩阵；另一个比较重要的特性是，TensorCore 可以进行混合精度运算。
### 混合精度
在 TensorCore 中，$A$ 和 $B$ 的精度是半精度（仅用 16 个比特位表示一个浮点数），$C$ 和 $D$ 的精度是全精度（用 32 个比特位表示一个浮点数）。这种结合不同精度的训练方式称为混合精度训练。**这样一方面半精度可以省略内存，另一方面，硬件层面会对半精度计算做优化，提升整体训练的速度。**
### 简要分析
在一个 $4 \times 4$ FMA 运算中，需要执行 64 个浮点数乘法以及 64 个浮点数加法。所以我们需要用到若干乘法器以及加法器，对于每一个运算，我们都使用一个乘法器或加法器进行处理。   
TensorCore 整体上接受指令、矩阵 $A B C$、时钟信号、复位信号作为输入；输出矩阵 $D$。每个时钟，TensorCore 分析指令，判断要进行什么形式的矩阵运算，随后分别进行处理。

### 代码
```System Verilog
   typedef logic [3:0][3:0][31:0] operand_t;
   typedef logic        bool_t;
   typedef logic [3:0][3:0][31:0] input_t;
   typedef logic [15:0] vinstr_t;
   // --------------
   // TensorCore
   module tensorcore #(
       parameter int unsigned DataWidth		= 32,
       parameter int unsigned MatrixLength   = 4, // The TensorCore can compute a MatrixLength x MatrixLength matrix's FMA
       parameter int unsigned INTeger = 0,
       parameter int unsigned mix_precision= 1,
       parameter int unsigned fp_32 = 2
   ) 
   (
       input logic      reset,
       input logic      clk,
       input vinstr_t   vecop,
       input input_t    matrix_a,
       input input_t    matrix_b,
       input input_t    matrix_c,
       output operand_t out
   );
   logic [DataWidth-1:0] temp;
   logic [MatrixLength-1:0][MatrixLength-1:0][MatrixLength-1:0][DataWidth-1:0] temp_multi;
   logic [MatrixLength-1:0][MatrixLength-2:0][MatrixLength-1:0][DataWidth-1:0] temp_add;
   logic [MatrixLength-1:0][MatrixLength-1:0][DataWidth-1:0] temp_out_fp_32;
   logic [MatrixLength-1:0][MatrixLength-1:0][MatrixLength-1:0][DataWidth-1:0] temp_multi_mix;
   logic [MatrixLength-1:0][MatrixLength-2:0][MatrixLength-1:0][DataWidth-1:0] temp_add_mix;
   logic [MatrixLength-1:0][MatrixLength-1:0][DataWidth-1:0] temp_out_fp_mix;
   genvar i;
   genvar j;
   genvar k;
   generate
   for(i = 0; i < MatrixLength; i++) begin:multi_1
        for(j = 0; j < MatrixLength; j++) begin:multi_2
            for(k = 0;k < MatrixLength; k++) begin:multi_3
            multiplier m(.a(matrix_a[i][k]),
            .b(matrix_b[k][j]),
            .o(temp_multi[i][k][j]));
            
            multiplier #(.DataWidth(16), .exp_len(5), .mid(20)) m_16(
                .a(matrix_a[i][k][15:0]),
            .b(matrix_b[k][j][15:0]),
            .o(temp_multi_mix[i][k][j]));
            end
        end
   end
   endgenerate
   generate
   for(i = 0; i < MatrixLength; i++) begin:add_1
        for(j = 0; j < MatrixLength; j++) begin:add_2
            for(k = 0;k < MatrixLength; k++) begin:add_3
            if (k == 0) begin
                adder a(.a(temp_multi[i][k][j]),
                .b(temp_multi[i][k+1][j]),
                .o(temp_add[i][k][j]));
                
                adder a_16(.a(temp_multi_mix[i][k][j]),
                .b(temp_multi_mix[i][k+1][j]),
                .o(temp_add_mix[i][k][j]));
            end
            else if (k == MatrixLength - 1)begin
                adder a(.a(temp_add[i][k-1][j]),
                .b(matrix_c[i][j]),
                .o(temp_out_fp_32[i][j]));
                
                adder a_16(.a(temp_add_mix[i][k - 1][j]),
                .b(matrix_c[i][j]),
                .o(temp_out_fp_mix[i][j]));
            end
            else begin
                adder a(.a(temp_add[i][k-1][j]),
                .b(temp_multi[i][k+1][j]),
                .o(temp_add[i][k][j]));
                
                adder a_16(.a(temp_add_mix[i][k-1][j]),
                .b(temp_multi_mix[i][k+1][j]),
                .o(temp_add_mix[i][k][j]));
            end
            end
        end
   end
   endgenerate
   always @(posedge clk or posedge reset) begin
      if (reset) begin
         out <= 0; // each element of output matrix is 0.
      end else begin
         if (vecop[1:0] == INTeger) begin
            for (int i = 0; i < MatrixLength; i++) begin
               for (int j = 0; j < MatrixLength; j++) begin
                  temp = matrix_c[i][j];
                  for (int k = 0; k < MatrixLength; k++) begin
                        temp = temp + matrix_a[i][k] * matrix_b[k][j];
                   end
                   out[i][j] <= temp;
                  end
               end
            end
         else if (vecop[1:0] == fp_32) begin
            out <= temp_out_fp_32;
         end
         else if (vecop[1:0] == mix_precision) begin
            out <= temp_out_fp_mix;
         end
        end
      end
   endmodule
```

上面的便是我们的代码，用到了我们之前文章中提到过的浮点数加法器以及乘法器。

### 仿真结果

```
[0] mode1: reset
reset success
[40000] mode3: random matrix integer
         0          0          0          0
         1          4          8         12
         1          9         16         24
         1         13         25         36
[50000] mode4: random matrix fp_32
0.000000 0.000000 0.000000 0.000000
0.100000 4.000000 8.000000 12.000000
0.100000 8.099999 16.000000 24.000000
0.100000 12.099999 24.099998 36.000000
[60000] mode4: random matrix fp_mix
0.000000 0.000000 0.000000 0.000000
0.099698 4.000000 8.000000 12.000000
0.099698 8.099697 16.000000 24.000000
0.099698 12.099697 24.099697 36.000000
```
仿真结果如上，reset 信号有效；整数，全精度，混合精度结果均完全正确。同时，可以发现半精度确实带来了一定的精度损失。
#### 附仿真文件
```System Verilog
`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
// Company: 
// Engineer: 
// 
// Create Date: 2025/07/29 13:52:54
// Design Name: 
// Module Name: sim_tensorcore
// Project Name: 
// Target Devices: 
// Tool Versions: 
// Description: 
// 
// Dependencies: 
// 
// Revision:
// Revision 0.01 - File Created
// Additional Comments:
// 
//////////////////////////////////////////////////////////////////////////////////
`timescale 1ns/1ps
module sim_tensorcore();
  typedef logic [3:0][3:0][31:0] operand_t;
  typedef logic [3:0][3:0][31:0] input_t;
  typedef logic [15:0] vinstr_t;
  localparam INTeger = 0;
  localparam fp_32 = 2;
  localparam mix_precision = 1; 
  logic clk;
  logic reset;
  vinstr_t vecop;
  input_t matrix_a;
  input_t matrix_b;
  input_t matrix_c;
  operand_t out;

  // 时钟生成（周期10ns）
  always #5 clk = ~clk;
  
  // 实例化被测模块
  tensorcore u_tensorcore (
    .clk(clk),
    .reset(reset),
    .vecop(vecop),
    .matrix_a(matrix_a),
    .matrix_b(matrix_b),
    .matrix_c(matrix_c),
    .out(out)
  );
  
  // 主测试逻辑
  initial begin
    // 初始化信号
    clk = 0;
    reset = 1;
    vecop = 0;
    matrix_a = 0;
    matrix_b = 0;
    matrix_c = 0;
    
    // 测试1: 复位功能
    $display("[%0t] mode1: reset", $time);
    # 20;
    if (out !== 0) $error("reset fail");
    else $display("reset success");
    
    // 释放复位
    reset = 0;
    # 20;
  
     // 随机矩阵
    $display("\n[%0t] mode3: random matrix integer", $time);
    vecop = INTeger;  // 设置整数模式
    
    // 创建矩阵A
    for (int i = 0; i < 4; i++) begin
      for (int j = 0; j < 4; j++) begin
        matrix_a[i][j] = i;
      end
    end
    // 创建矩阵B
    for (int i = 0; i < 4; i++) begin
      for (int j = 0; j < 4; j++) begin
        matrix_b[i][j] = j;
      end
    end
    // 创建矩阵C
    for (int i = 0; i < 4; i++) begin
      for (int j = 0; j < 4; j++) begin
        matrix_c[i][j] = (i > j)? 1 : 0;
      end
    end
    
     # 6;
    for (int i = 0; i < 4; i++) begin
        $display("%d %d %d %d", out[i][0], out[i][1], out[i][2], out[i][3]);
    end
    # 4;
    
     // 随机矩阵
    $display("\n[%0t] mode4: random matrix fp_32", $time);
    vecop = fp_32;  // 设置fp_32数模式
    
    // 创建矩阵A
    for (int i = 0; i < 4; i++) begin
      for (int j = 0; j < 4; j++) begin
        matrix_a[i][j] = $shortrealtobits(shortreal(i));
//         $display("%f", $bitstoshortreal(matrix_a[i][j]) * $bitstoshortreal(matrix_a[i][j]));
      end
    end
    // 创建矩阵B
    for (int i = 0; i < 4; i++) begin
      for (int j = 0; j < 4; j++) begin
        matrix_b[i][j] = $shortrealtobits(shortreal(j));
      end
    end
    // 创建矩阵C
    for (int i = 0; i < 4; i++) begin
      for (int j = 0; j < 4; j++) begin
        matrix_c[i][j] = (i > j)? $shortrealtobits(0.1) : $shortrealtobits(0.0);
      end
    end
    # 10;
    for (int i = 0; i < 4; i++) begin
        $display("%f %f %f %f", $bitstoshortreal(out[i][0]), $bitstoshortreal(out[i][1]), $bitstoshortreal(out[i][2]), $bitstoshortreal(out[i][3]));
    end
    
    // 随机矩阵
    $display("\n[%0t] mode4: random matrix fp_mix", $time);
    vecop = mix_precision;  // 设置fp_32数模式
    
    // 创建矩阵A
    for (int i = 0; i < 4; i++) begin
      for (int j = 0; j < 4; j++) begin
        reg [31:0] x = $shortrealtobits(shortreal(i));
        if (x!=0) begin
            matrix_a[i][j][15] = x[31];
            matrix_a[i][j][14:10] = x[30:23] - 112;
            matrix_a[i][j][9:0] = (x[22:0] >> 13); 
        end
        else begin
            matrix_a[i][j]  = 0;
        end
//         $display("%f", $bitstoshortreal(matrix_a[i][j]) * $bitstoshortreal(matrix_a[i][j]));
      end
    end
    // 创建矩阵B
    for (int i = 0; i < 4; i++) begin
      for (int j = 0; j < 4; j++) begin
        reg [31:0] x = $shortrealtobits(shortreal(j));
        if (x!=0) begin
            matrix_b[i][j][15] = x[31];
            matrix_b[i][j][14:10] = x[30:23] - 112;
            matrix_b[i][j][9:0] = (x[22:0] >> 13); 
        end
        else begin
            matrix_b[i][j]  = 0;
        end
      end
    end
    // 创建矩阵C
    for (int i = 0; i < 4; i++) begin
      for (int j = 0; j < 4; j++) begin
        reg [31:0]x = (i > j)? $shortrealtobits(0.1) : $shortrealtobits(0.0);
        if (x!=0) begin
            matrix_c[i][j][15] = x[31];
            matrix_c[i][j][14:10] = x[30:23] - 112;
            matrix_c[i][j][9:0] = (x[22:0] >> 13); 
        end
        else begin
            matrix_c[i][j]  = 0;
        end
      end
    end
    # 10;
    for (int i = 0; i < 4; i++) begin
        $display("%f %f %f %f", $bitstoshortreal(out[i][0]), $bitstoshortreal(out[i][1]), $bitstoshortreal(out[i][2]), $bitstoshortreal(out[i][3]));
    end
    end
endmodule
```
