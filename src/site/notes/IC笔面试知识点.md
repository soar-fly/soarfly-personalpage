---
{"dg-publish":true,"dg-home":true,"permalink":"/IC笔面试知识点/","tags":["gardenEntry"],"dgPassFrontmatter":true,"dg-note-properties":{}}
---


### 跨时钟域数据处理
对于单bit信号，简单的由慢时钟域到快时钟域，用两个D触发器打两拍即可。对于一般性的处理方法，使用脉冲展宽处理（握手法）。具体的思路是req_a在bit_a升高时拉高；req_b_r打两拍同步req_a，且bit_b=~req_b_r1 & req_b识别上升沿；ack_a_r打两拍同步req_b_r；最后req_a在收到ack_a_r的应答信号后拉低
```verilog
module Sync_Pulse
(
    input   clka,
    input   clkb,
    input   rst_n,
    input   bit_a,
    output  bit_b
);
reg         req_a;
reg         ack_a;
reg         ack_a_r1;
reg         req_b;
reg         req_b_r1;

//--    a时钟域生成展宽信号
always @(posedge clka or negedge rst_n)begin
    if(!rst_n)begin
        req_a <= 1'b0;
    end
    else if(bit_a) begin         //检测到脉冲
        req_a <= 1'b1;           //进行展宽
    end
    else if(ack_a_r1) begin      //同步到b时钟域后得到应答
        req_a <= 1'b0;           //展宽使命完成
    end
end

//--    展宽信号同步到b时钟域
always @(posedge clkb or negedge rst_n)begin
    if(!rst_n)begin
        req_b    <= 1'b0;
        req_b_r1 <= 1'b0;
    end
    else begin
        req_b    <= req_a;
        req_b_r1 <= req_b;
    end
end

//--    展宽信号同步回b时钟域，作为应答
always @(posedge clka or negedge rst_n)begin
    if(!rst_n)begin
        ack_a    <= 1'b0;
        ack_a_r1 <= 1'b0;
    end
    else begin
        ack_a    <= req_b_r1;
        ack_a_r1 <= ack_a;
    end
end

//--    脉冲信号输出,上升沿检测
assign bit_b = ~req_b_r1 & req_b;
 
endmodule
```
对于多bit的跨时钟域信号同步，使用异步FIFO即可

### 手撕FIFO-verilog
```verilog
module FIFO #(
	parameter	WIDTH = 8,
	parameter 	DEPTH = 16
)(
	input 					clk		, 
	input 					rst_n	,
	input 					winc	,
	input 			 		rinc	,
	input 		[WIDTH-1:0]	wdata	,

	output reg				wfull	,
	output reg				rempty	,
	output wire [WIDTH-1:0]	rdata
);

	parameter ADDR_WIDTH = $clog2(DEPTH);
	//指针更新逻辑
	reg [ADDR_WIDTH:0] wptr_bin;
	reg [ADDR_WIDTH:0] rptr_bin;
	wire [ADDR_WIDTH:0] wptr_bin_next;
	wire [ADDR_WIDTH:0] rptr_bin_next;
	
	assign wptr_bin_next = wptr_bin + (winc & !wfull);
	assign rptr_bin_next = rptr_bin + (rinc & !rempty);
	
	always @ (posedge clk or negedge rst_n) begin
		if (!rst_n) begin
			wptr_bin <= 'd0;
			rptr_bin <= 'd0;
		end
		else begin
			wptr_bin <= wptr_bin_next;
			rptr_bin <= rptr_bin_next;
		end
	end
	
	//空满判断逻辑
	assign wfull = (wptr_bin == {~rptr_bin[ADDR_WIDTH],rptr_bin[ADDR_WIDTH-1 : 0]}) ? 1'b1 : 1'b0;
	assign rempty = (wptr_bin == rptr_bin) ? 1'b1 : 1'b0;

	//例化RAM
    wire wenc, renc;
    wire [ADDR_WIDTH-1:0] raddr, waddr;
    assign wenc = winc & !wfull;
    assign renc = rinc & !rempty;
    assign raddr = rptr_bin[ADDR_WIDTH-1:0];
    assign waddr = wptr_bin[ADDR_WIDTH-1:0];

    dual_port_RAM #(.DEPTH(DEPTH), .WIDTH(WIDTH)) 
                u_dual_port_RAM (
                .wclk(clk), 
                .rclk(clk), 
                .wenc(wenc), 
                .renc(renc),
                .raddr(raddr),
                .waddr(waddr),
                .wdata(wdata),
                .rdata(rdata)
                );
endmodule
******************************/
module dual_port_RAM #(parameter DEPTH = 16,
					   parameter WIDTH = 8)(
	input wclk，
	input wenc，
	input [$clog2(DEPTH)-1:0] waddr  ，
	input [WIDTH-1:0] wdata  ，    	
	input rclk，
	input renc，
	input [$clog2(DEPTH)-1:0] raddr  ，
	output reg [WIDTH-1:0] rdata 		
);

reg [WIDTH-1:0] RAM_MEM [0:DEPTH-1];

always @(posedge wclk) begin
	if(wenc)
		RAM_MEM[waddr] <= wdata;
end 

always @(posedge rclk) begin
	if(renc)
		rdata <= RAM_MEM[raddr];
end 

endmodule  
``````



### SystemVerilog基础
###### 如何使用功能覆盖率测试中断的功能？
1.确定功能覆盖率的目标：中断延迟、中断频率、中断优先级等
2.为interrupt写一个testbench，检测DUT里中断的响应
3.实现functional coverage，使用sv构造covergroups，coverpoints，bins来定义和追踪功能覆盖率
4.分析functional coverage的结果，再根据结果反向调整testbench以改进

###### sv中::操作符的用途？
指定标识符的定义范围或搜索范围。
1.获取继承关系内部的module或变量
2.解决不同模块的命名重复问题
3.获取静态变量和函数
4.获取package中的item，利用import

###### sv中如何应用randc？
randc会先穷举所有组合，再重复某个值，而rand可能在穷举完所有组合前就有值重复

###### 什么是DPI，解释DIP的export和import？
Direct Programming Interface，是一种集成sv设计和外部C/C++代码的机制，使用export可以把C代码变成sv中的一个tast或者function

###### semaphore是什么？scenario用在什么上？
semaphore是一种控制对共享资源访问的同步机制。避免同时访问导致冲突和数据不一致，产生意外行为

###### deep copy和shallow copy的区别？
deep copy指嵌套类对象的内容也被完全复制到新的类对象中，而shallow copy只会简单的分配句柄

###### 如何关闭约束的使能？
```systemverilog
class ABC;
	constraint c_data{...}
endclass
ABC m_abc = new;
m_abc.c_data.constraint_mode(0);
```

###### code coverage和functional coverage的区别？
code coverage 衡量testbench对RTL级代码的测试成都，哪些代码被执行，分支、循环语句被执行。通常以语句覆盖率、分支覆盖率、条件覆盖率来衡量。而functional coverage是指具体的测试场景中哪些所需求的功能被覆盖到，逆流一个接口的正确传输和接收的数据量。

###### 验证中的层架构是什么？
testbench layer-verification IP layer(VIP)-functional block layer

###### structure和class的区别？
二者都能定义自定义的数据类型，及其特征和功能。
1.class支持私有，公开和被保护的获取修改数据，而structure永远是公开的
2.class有继承属性
3.class有构造函数和析构函数，可以在对象创建和销毁时使用，而structure是一直存在的
4.class可以有functions和tasks

###### integer和int的区别？
都能存储32bit的signed有符号数，但是integer可代表4种状态，int只能表示0/1两种状态

###### 如何在monitor和scoreboard之间建立联系？
monitor观察设计的信号并生成transaction，而scoreboard将这些transaction与预期值进行比较并生成结果，它们可以通过mailbox进行连接。

###### 如果functional coverage是100%而code coverage太低应该怎么办？
这种情况意味着在测试例中有很多代码没有被执行。
1.检查代码并确定未覆盖的地方
2.添加新的测试例
3.修改现有的测试例
4.使用代码覆盖率指标，为特定的覆盖率（分支、语句）设置目标指标
5.使用覆盖驱动验证和约束随机测试等奇数

###### 如何找到与associative array项相关的索引？
```systemverilog
module tb;
	int fruit_hash [string];
	string idx_q [$];
	
	initial begin
		fruit_hash["apple"] = 5;
		fuite_hash["pear"] = 3;
		fruit_hash["mango"] = 9;
		idx_q = fruit_hash.find_index with (1);
		$display("idx_q=%p",idx_q);
	end
endmodule
```

###### 什么是pass-by-value和pass-by-reference方法？
两种函数传参的方式。pass-by-value只穿递参数的复制值，函数内部对参数的改变不影响其原始定义，而pass-by-reference传递参数的存储器位置

###### code coverage包括什么？
statements,branches,expressions,toggle,assertions,FSM

###### 写一个在8bit sequence中检测奇数元素的constraint？
```systemverilog
class ABC
	rand bit [7:0] data;
	constraint c_data {$countones(data)%2!=0;}
endclass

module tb
	initial begin
		ABC m_abc = new;
		for(byte i=0; i<20; i++) begin
			m_abc=randomize();
			$display("data=0b%0b", m_abc.data);
		end
	end
endmodule
```

###### OOP概念在验证中的作用？
1.模块化设计：每个模块/类代表设计的不同方面，通过复用和集成这些模块化组件，可以更容易地开发testbench，节省时间精力，并使得系统更具可扩展性
2.封装：对任何数据对象执行的操作都可以在同一class中定义
3.继承：允许创建基类并使用进一步的派生类对其进行扩展，通常在基类中定义公共的属性和方法，以便所有子类可以访问
4.多态性：允许多种不同方法或函数实现，多态性有助于帮助实现不同的测试用例场景，同时不对代码其他部分产生显著影响