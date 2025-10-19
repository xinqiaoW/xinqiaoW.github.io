---
toc: true
tags: GPU
---
本文主要介绍如何通过 Verilog 设计一个简单的 Warp Scheduler，Warp Scheduler 是 GPU 中用来调度 Warp、 发射指令的模块。

### Warp 概念
Warp 可以理解为 一组线程（比如 32 个）。正常情况下，Warp 内的所有线程会执行同一条指令。那么我们为什么需要 Warp 呢？首先，管理 Warp 相比线程级的管理更加简单，同时在硬件层面上，如果为每一个线程分别设立一个独立的 PC 寄存器较为困难，不如统一用一个 PC 寄存器，这样可以大大降低成本与设计难度。

### SIMTStack
SIMT(Single Instruction Multiple Thread) 是指同一条指令由多个线程执行，即我们上面说到的 warp 执行方式（Warp 内的所有线程会执行同一条指令）。值得注意的是，某些时候，我们可能会遇到条件分支指令，同一个 Warp 内的不同线程可能进入到不同的分支。我们需要用到 Stack （栈）解决这个问题。

我们设计的策略可以简单描述如下:
```
每次遇到产生分支的指令
  将 {活跃掩码, 该分支结束后 汇合代码块 的 起始地址} 压入栈中
  将 {活跃掩码, 该 else 代码块 的 起始地址} 压入栈中
  将 {活跃掩码, 该 if 代码块 的 起始地址} 压入栈中

弹出栈顶元素，读取 线程活跃掩码 以及 if 代码块 的 起始地址 -> 执行;
if 代码块结束后，产生信号弹出栈顶元素，读取 线程活跃掩码 以及 else 代码块 的 起始地址-> 执行;

以此类推
（每个分支可能有不同的活跃掩码，一部分线程进入 if，另一部分进入 else）
```

Verilog 代码如下：
```
module SIMTStack #(
    parameter DataWidth = 32,
    parameter AddrWidth = 32,
    parameter ThreadNum = 32,
    parameter DEPTH = 8
) (
    input wire clk,
    input wire reset,
    input wire [AddrWidth + ThreadNum - 1:0] in_data,         // 压入栈的是 活跃掩码 以及 PC地址
    input wire push,
    input wire pop,
    output wire [AddrWidth + ThreadNum - 1:0] out_data
);
    integer sp;
    
    reg [AddrWidth + ThreadNum - 1:0] stack_mem [0:DEPTH-1];
    
    assign out_data = stack_mem[(sp > 0) ? (sp - 1) : 0];
    
    always @(posedge clk or posedge reset) begin
        if (reset) begin
            sp <= 0;
        end else begin
            if (push) begin
                stack_mem[sp] <= in_data;
                sp <= sp + 1;
            end else if (pop && sp > 0) begin
                sp <= sp - 1;
            end
        end
    end

endmodule
```
为了更清晰地描述基本功能，上述的代码没有栈满栈空检验。请测试的时候注意这个问题。

### Warp Scheduler
我们前面说到，Warp Scheduler 是用来分配 Warp，发射指令的。首先，我们需要一个起始 PC，以及起始控制指令，用来代表一个待分配的任务。
我们的实现中用 warp_cmd_pc、warp_cmd_valid、warp_cmd_ready 来控制。warp_cmd_ready 只有当存在空闲 warp 的时候为 1，只有这个时候我们可以为一个任务分配 warp 执行。
```
    assign has_idle = |warp_idle;
    assign warp_cmd_ready = has_idle; 
    // Assign Warp 
    if (warp_cmd_valid && warp_cmd_ready) begin
        warp_idle[idle_id] <= 1'b0;            
        warp_active[idle_id] <= 1'b1;          
        warp_pc[idle_id] <= warp_cmd_pc;       // set start pc
        warp_tmask[idle_id] <= warp_cmd_mask;  // set thread mask
    end
```
我们还需要控制是否弹栈的信号，warp_pop_valid、warp_pop_wid。当 warp_pop_valid 为 1 的时候，我们需要弹栈，并修改对应 Warp 的 PC 寄存器。
```
    genvar i;
    generate
        for (i = 0; i < WarpNum; i = i + 1) begin : simt_stacks
            SIMTStack #(
                .DataWidth(DataWidth),
                .AddrWidth(AddrWidth),
                .ThreadNum(ThreadNum)
            ) simt_stack (
                .clk(clk),
                .reset(reset),
                // branch?
                .in_data(branch_ctl_data),
                // push
                .push(branch_ctl_valid && (branch_ctl_wid == i)),
                // pop 
                .pop(warp_pop_valid && (warp_pop_wid == i)),
                // output data
                .out_data(simt_stack_out_data[i])
            );
        end
    endgenerate

    if (warp_pop_valid) begin
        warp_pc[warp_pop_wid] <= simt_stack_out_data[warp_pop_wid][AddrWidth-1:0]; 
        warp_tmask[warp_pop_wid] <= simt_stack_out_data[warp_pop_wid][AddrWidth + ThreadNum - 1:AddrWidth];
    end
```
我们通过 inst_fetch 发射指令（PC Mask 以及 Warp ID）。
```
    // instruction fetched signal
    assign inst_fetch_valid = has_active;
    assign inst_fetch_pc = has_active ? warp_pc[active_id] : {AddrWidth{1'b0}};
    assign inst_fetch_mask = has_active ? warp_tmask[active_id] : {ThreadNum{1'b0}};
    assign inst_fetch_wid = has_active ? active_id : {$clog2(WarpNum){1'b0}};
```
这里我们使用的是 组合逻辑，在更改 PC Mask active_id 后会立刻影响 inst_fetch，从而确保我们可以取出正确的指令。

我们需要一个 end_ctl 来结束某一个 Warp。
```
 // End
            if (end_ctl_valid && end_ctl_ready) begin
                warp_idle[end_ctl_wid] <= 1'b1;        
            end
```

除此之外，我们还需要激活某个不活跃的 Warp。warp_active 和 warp_idle 是两个概念，只要某个 warp 正在执行程序，我们就将其对应的 warp_idle 设立为 0。但是它并不一定处于活跃状态。因为我们不可能总让某一个 warp 发射指令，所以有时候我们会将某些 warp 的活跃状态设置为 0，避免由其发射指令。
为了使得这个 warp 可以重新执行，我们需要重新激活它（我们的实现中采用 warp_ctl 来控制）。
```
// ReActive a certain warp
            if (warp_ctl_valid && warp_ctl_ready) begin
                warp_active[warp_ctl_wid] <= warp_ctl_active;
            end
```
以上就是我们设计的大部分细节。全部代码附录如下:
```
`default_nettype none
`timescale 1ns/1ps

module WarpScheduler #(
    parameter WarpNum     = 8,      
    parameter ThreadNum   = 32,     
    parameter AddrWidth   = 32,    
    parameter DataWidth   = 32      
) (
    input  wire clk,                // clock
    input  wire reset,              // reset signal
    
    // Warp command
    input  wire warp_cmd_valid,     
    output wire warp_cmd_ready,     
    input  wire [AddrWidth-1:0] warp_cmd_pc,   
    input  wire [ThreadNum-1:0] warp_cmd_mask,  
    
    // warp control signal
    input  wire warp_ctl_valid,     
    output wire warp_ctl_ready,     
    input  wire [$clog2(WarpNum)-1:0] warp_ctl_wid,  // Warp ID
    input  wire warp_ctl_active,    
    //////////////////////////////////////////////////////////////////////
    input  wire branch_ctl_valid,   
    output wire branch_ctl_ready,   
    input  wire [$clog2(WarpNum)-1:0] branch_ctl_wid,  // Warp ID
    input  wire [AddrWidth + ThreadNum - 1:0] branch_ctl_data,
    
    // end control
    input  wire end_ctl_valid,     
    output wire end_ctl_ready,     
    input  wire [$clog2(WarpNum)-1:0] end_ctl_wid,  // Warp ID
    // pop control
    input  wire warp_pop_valid,                      
    input  wire [$clog2(WarpNum)-1:0] warp_pop_wid,  

    
    output wire inst_fetch_valid,   
    input  wire inst_fetch_ready,   
    output wire [AddrWidth-1:0] inst_fetch_pc,  // PC Register
    output wire [ThreadNum-1:0] inst_fetch_mask,// Thread Mask
    output wire [$clog2(WarpNum)-1:0] inst_fetch_wid,  // Warp ID
    
    //////////////////////////////////////////////////////////////
    // output for simulation
    output reg [(WarpNum)-1:0] idle_id_out,
    output reg [(WarpNum)-1:0] active_id_out,
    output wire [AddrWidth + ThreadNum - 1:0] pop_data_out
    //////////////////////////////////////////////////////////////
);

    reg [WarpNum-1:0] warp_idle;        // 1-idle
    reg [WarpNum-1:0] warp_active;     // 1-active
    reg [AddrWidth-1:0] warp_pc [0:WarpNum-1];         
    reg [ThreadNum-1:0] warp_tmask [0:WarpNum-1];      
    
    wire has_idle;                      
    wire has_active;                   
    wire [$clog2(WarpNum)-1:0] idle_id; 
    wire [$clog2(WarpNum)-1:0] active_id;
    
    
    genvar i;
    generate
        for (i = 0; i < WarpNum; i = i + 1) begin : simt_stacks
            SIMTStack #(
                .DataWidth(DataWidth),
                .AddrWidth(AddrWidth),
                .ThreadNum(ThreadNum)
            ) simt_stack (
                .clk(clk),
                .reset(reset),
                // branch?
                .in_data(branch_ctl_data),
                // push
                .push(branch_ctl_valid && (branch_ctl_wid == i)),
                // pop 
                .pop(warp_pop_valid && (warp_pop_wid == i)),
                // output data
                .out_data(simt_stack_out_data[i])
            );
        end
    endgenerate
    
    //SIMTStack output
    wire [AddrWidth + ThreadNum - 1:0] simt_stack_out_data [0:WarpNum-1];
   
    PriorityEncoder #(
        .WIDTH(WarpNum)
    ) idle_encoder (
        .in(warp_idle),
        .out(idle_id),
        .valid() 
    );
    
    PriorityEncoder #(
        .WIDTH(WarpNum)
    ) active_encoder (
        .in(warp_active),
        .out(active_id),
        .valid() 
    );
    
    assign active_id_out = warp_active;
    assign idle_id_out = warp_idle;
    assign pop_data_out = simt_stack_out_data[warp_pop_wid];
    
    assign has_idle = |warp_idle;
    assign has_active = |warp_active;
    
    // signal
    assign warp_cmd_ready = has_idle;  
    assign warp_ctl_ready = 1'b1;       
    assign branch_ctl_ready = 1'b1;    
    assign end_ctl_ready = 1'b1;        

    // instruction fetched signal
    assign inst_fetch_valid = has_active;
    assign inst_fetch_pc = has_active ? warp_pc[active_id] : {AddrWidth{1'b0}};
    assign inst_fetch_mask = has_active ? warp_tmask[active_id] : {ThreadNum{1'b0}};
    assign inst_fetch_wid = has_active ? active_id : {$clog2(WarpNum){1'b0}};
    
    always @(posedge clk or posedge reset) begin
        integer w, t;
        if (reset) begin
            // 复位初始化
            warp_idle <= {WarpNum{1'b1}};      
            warp_active <= {WarpNum{1'b0}};    
            
            // initialization
            for (w = 0; w < WarpNum; w = w + 1) begin
                warp_pc[w] <= {AddrWidth{1'b0}};
                for (t = 0; t < ThreadNum; t = t + 1) begin
                    warp_tmask[w][t] <= 1'b0;
                end
            end
            
        end else begin
            
            // Assign Warp 
            if (warp_cmd_valid && warp_cmd_ready) begin
                warp_idle[idle_id] <= 1'b0;            
                warp_active[idle_id] <= 1'b1;          
                warp_pc[idle_id] <= warp_cmd_pc;       // set start pc
                warp_tmask[idle_id] <= warp_cmd_mask;  // set thread mask
            end
            
            // End
            if (end_ctl_valid && end_ctl_ready) begin
                warp_idle[end_ctl_wid] <= 1'b1;        
            end
            
            // ReActive a certain warp
            if (warp_ctl_valid && warp_ctl_ready) begin
                warp_active[warp_ctl_wid] <= warp_ctl_active;
            end
    
            
            // SIMTStack pop
            if (warp_pop_valid) begin
                warp_pc[warp_pop_wid] <= simt_stack_out_data[warp_pop_wid][AddrWidth-1:0]; 
                warp_tmask[warp_pop_wid] <= simt_stack_out_data[warp_pop_wid][AddrWidth + ThreadNum - 1:AddrWidth];
            end
            
            // instruction fetched
            if (inst_fetch_valid && inst_fetch_ready) begin
                warp_active[active_id] <= 1'b0;       
                warp_pc[active_id] <= warp_pc[active_id] + 4; // PC <- PC + 4
            end
        end
    end

endmodule

// ====================Naive Priority Encoder ====================
module PriorityEncoder #(
    parameter WIDTH = 8
) (
    input wire [WIDTH-1:0] in,          
    output reg [$clog2(WIDTH)-1:0] out, 
    output wire valid                   
);
    
    assign valid = |in;

    always @(*) begin
        out = {$clog2(WIDTH){1'b0}};   
        for (integer i = 0; i < WIDTH; i = i + 1) begin
            if (in[i]) begin
                out = i;               
            end
        end
    end
    
endmodule

// ==================== SIMTStack ====================
module SIMTStack #(
    parameter DataWidth = 32,
    parameter AddrWidth = 32,
    parameter ThreadNum = 32,
    parameter DEPTH = 8
) (
    input wire clk,
    input wire reset,
    input wire [AddrWidth + ThreadNum - 1:0] in_data,         // 压入栈的是指令地址
    input wire push,
    input wire pop,
    output wire [AddrWidth + ThreadNum - 1:0] out_data
);
    integer sp;
    
    reg [AddrWidth + ThreadNum - 1:0] stack_mem [0:DEPTH-1];
    
    assign out_data = stack_mem[(sp > 0) ? (sp - 1) : 0];
    
    always @(posedge clk or posedge reset) begin
        if (reset) begin
            sp <= 0;
        end else begin
            if (push) begin
                stack_mem[sp] <= in_data;
                sp <= sp + 1;
            end else if (pop && sp > 0) begin
                sp <= sp - 1;
            end
        end
    end

endmodule
```

### Context Switching
我们的代码目前是没有实现 Context Switching 的。如果要实现的话，一个比较简单的思路可能是对于每一个需要的元件，设置多套寄存器。执行某个 Warp 的时候根据特定的一套寄存器进行计算即可。
