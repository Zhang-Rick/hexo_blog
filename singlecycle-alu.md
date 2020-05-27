---
title: singlecycle设计及实现2 (ALU算术逻辑单元)
date: 2020-03-21 08:08:16
categories: mips CPU design
tags: 
- FPGA
- verilog
- CPU
---
### 简介
这篇博文主要介绍的是基于mips的 singlecycle CPU设计及实现的第二个单元ALU算术逻辑单元。
### 详解
![][image-1]
衔接上篇的寄存器的讨论，这次将对ALU算术逻辑单元进行编写。mips支持的指令有很多指令形式(instruction format)，如
R-type：Op, rs, rt, rd 单纯只用到了寄存器的值，
I-type：OP rt, IMM(rs) 或 OP  rs, rt, IMM 用到了立即值，(这里的立即值是16位)
J-type：Op, address 用到了地址。(这里的地址是26位)。
为什么要进行分类呢，因为根据不同的类型，ALU 算术逻辑单元的数据总线的值可能会有不同。
例如：
R-type ADD：bus1 就是rt, bus2 就是rd
I-type LD：     bus1 就是rs, bus2就是 IMM
I-type BEQ：  bus1 就是rs, bus2就是 IMM(减法判断结果是不是0)
j-type JAL：   不需要ALU，PC单元能够单独解决。

算术逻辑单元只能实现一些比较简单的算术，但是mips却支持很多的指令。因为这些复杂的指令可以通过简单的逻辑算术和别的模块解决。

但是为什么不能直接根据opcode直接进行运算，像 opcode == add 把rt + rd的值放入rs。直接把多个模块需要合作解决的和在一起。 因为通过简单的逻辑指令和别的模块的合作，能灵活的实现复杂的指令，最重要的是能省下不少晶体管。

那为什么算术逻辑单元的输出还有负数符号，溢出符号。但是在总的singlecycle CPU设计上没有这些输出信号，因为mips不关心溢出和负数符号，但是零符号是需要用来检测分支的。
#### ALU 算术逻辑单元
![][image-2]
如图所示上图展示了算术逻辑单元的一些输入及输出。
输入:
PortA(31:0) 32位的bus1输入
PortB(31:0) 32位的bus2输入
ALUOP(3:0) 算术逻辑单元指令选择
输出:
Negative 负符号
Overflow 溢出符号
Zero 零符号
Output Port(31:0) 算术逻辑单元输出

这次因为接口比较少，所以偷了懒没有写接口文件，但是写了FPGA文件。
##### 主代码alu.sv
 ```verilog
`	`include "cpu_types_pkg.vh"
	module alu
	(
		input logic [31:0] portA,
		input logic [31:0] portB,
		input logic [3:0] aluOP,
		output logic negFlag,
		output logic oveFlag,
		output logic zerFlag,
		output logic [31:0] outputPort
	);
	import cpu_types_pkg::*;
	always_comb begin
		negFlag = 0;
		oveFlag = 0;
		zerFlag = 0;
		outputPort = '0;
		if (aluOP == ALU_SLL)//向左移
			outputPort =  portB << portA;
		else if (aluOP == ALU_SRL))//向右移
			outputPort =  portB >> portA;
		else if (aluOP == ALU_ADD) begin)//加
			outputPort = portA + portB;
			if (outputPort[31] != (portA[31] ^ portB[31]))
				oveFlag = 1;
			end
		else if (aluOP == ALU_SUB) begin//减
			outputPort = portA - portB;
			if (outputPort[31] != (portA[31] ^ portB[31]))
				oveFlag = 1;
			end
		else if (aluOP == ALU_AND)//和
			outputPort = portA & portB;
		else if (aluOP == ALU_OR)//或
			outputPort = portA | portB;
		else if (aluOP == ALU_XOR)//异或
			outputPort = portA ^ portB;
		else if (aluOP == ALU_NOR)//非或
			outputPort = ~(portA | portB);
		else if (aluOP == ALU_SLTU)//小于不考虑符号
			outputPort = portA < portB;
		else if (aluOP == ALU_SLT) begin//小于考虑符号 这段代码应该写错了 最后评分就这段没过测试
			if (portA[31] == 1 && portB[31] == 0)
				outputPort = 1;
			else if (portA[31] == 0 && portB[31] == 1)
				outputPort = 0;
			else
				outputPort = portA < portB;
	    end
	 //单独判断零和负符号
		if (outputPort == 0)
			zerFlag = 1;
		if (outputPort[31] == 1)
			negFlag = 1;
		end
	endmodule
	
```
`
这次测试代码直接用最简单粗暴的方式测试了，但是覆盖率还是100%。(根据questasim的report判断的)
##### 测试代码alu\_tb.sv
```verilog
`	`include "cpu_types_pkg.vh"
	// mapped timing needs this. 1ns is too fast
	`timescale 1 ns / 1 ns
	module alu_tb();
	
	  parameter PERIOD = 10;
	
	  logic CLK = 0, nRST;
	
	  logic [31:0] portA;
	  logic [31:0] portB;
	  logic [3:0] aluOP;
	  logic negFlag;
	  logic oveFlag;
	  logic zerFlag;
	  logic [3:0] i;
	  logic [31:0] outputPort;
	  always #(PERIOD/2) CLK++;
	
	  import cpu_types_pkg::*;
	  // test program
	  //test PROG (portA,portB,aluOP,negFlag,oveFlag,negFlag,zerFlag,outputPort);
	  // DUT
	 
	  alu DUT(portA,portB,aluOP,negFlag,oveFlag,zerFlag,outputPort);
	  initial begin
		portA = '0;
		portB = '0;
		aluOP = '0;
	
		for (i = 0; i < 14; i = i + 1) begin
		@(posedge CLK)
		portA = '1;
		portB = '1;
		aluOP = i;
		
		@(posedge CLK)
		portA = 0;
		portB = 0;
		aluOP = i;
		
		@(posedge CLK)
		portA = 1;
		portB = 0;
		aluOP = i;
	
		@(posedge CLK)
		portA = 0;
		portB = 1;
		aluOP = i;
	
		@(posedge CLK)
		portA = '0;
		portB = '0;
		aluOP = i;
			
	
	
	
		end	
		@(posedge CLK)
		portA = '0;
		portB = '0;
		aluOP = '1;
		@(posedge CLK)
		portA = '0;
		portB = '0;
		aluOP = 0;
		@(posedge CLK)
		
	
		$stop;
		end
	endmodule
	
	
	
```
`
##### FPGA代码alu\_fpga.sv
```verilog
`	/*
	  Eric Villasenor
	
	  register file fpga wrapper
	*/
	
	// interface
	`include "cpu_types_pkg.vh"
	
	module alu_fpga (
	  input logic CLOCK_50,
	  input logic [3:0] KEY,
	  input logic [17:0] SW,
	  output logic [17:0] LEDR,
	  output logic [6:0]  HEX0,
	  output logic [6:0]  HEX1,
	  output logic [6:0]  HEX2,
	  output logic [6:0]  HEX3
	);
		import cpu_types_pkg::*;
		logic negFlag;
		logic oveFlag;
		logic zerFlag;
		logic [31:0] output1;
		reg [31:0] register;
		reg [31:0] nregister;
		logic [31:0] portA;
		assign portA = {16*{SW[16]},SW[15:0]};
		always_ff @(posedge CLOCK_50) begin
			if (SW[17])
				register <= {16*{SW[16]},SW[15:0]};
			else
				register <= register;
		end
		
		alu DUT(.portA(portA),.portB(register),.aluOP(~KEY),.negFlag(negFlag),.oveFlag(oveFlag),.zerFlag(zerFlag),.outputPort(output1));//map 版本
	
		always_comb
		begin
	 	unique casez (output1[3:0])//led显示设置
	      'h0: HEX0 = 7'b1000000;
	      'h1: HEX0 = 7'b1111001;
	      'h2: HEX0 = 7'b0100100;
	      'h3: HEX0 = 7'b0110000;
	      'h4: HEX0 = 7'b0011001;
	      'h5: HEX0 = 7'b0010010;
	      'h6: HEX0 = 7'b0000010;
	      'h7: HEX0 = 7'b1111000;
	      'h8: HEX0 = 7'b0000000;
	      'h9: HEX0 = 7'b0010000;
	      'ha: HEX0 = 7'b0001000;
	      'hb: HEX0 = 7'b0000011;
	      'hc: HEX0 = 7'b0100111;
	      'hd: HEX0 = 7'b0100001;
	      'he: HEX0 = 7'b0000110;
	      'hf: HEX0 = 7'b0001110;
		default: HEX0 = 7'b1000000; // "0" 
	 	endcase
	
	 	unique casez (output1[7:4])
	      'h0: HEX1 = 7'b1000000;
	      'h1: HEX1 = 7'b1111001;
	      'h2: HEX1 = 7'b0100100;
	      'h3: HEX1 = 7'b0110000;
	      'h4: HEX1 = 7'b0011001;
	      'h5: HEX1 = 7'b0010010;
	      'h6: HEX1 = 7'b0000010;
	      'h7: HEX1 = 7'b1111000;
	      'h8: HEX1 = 7'b0000000;
	      'h9: HEX1 = 7'b0010000;
	      'ha: HEX1 = 7'b0001000;
	      'hb: HEX1 = 7'b0000011;
	      'hc: HEX1 = 7'b0100111;
	      'hd: HEX1 = 7'b0100001;
	      'he: HEX1 = 7'b0000110;
	      'hf: HEX1 = 7'b0001110;
		default: HEX1 = 7'b1000000; // "0" 
	 	endcase
	
	 	unique casez (output1[11:8])
	      'h0: HEX2 = 7'b1000000;
	      'h1: HEX2 = 7'b1111001;
	      'h2: HEX2 = 7'b0100100;
	      'h3: HEX2 = 7'b0110000;
	      'h4: HEX2 = 7'b0011001;
	      'h5: HEX2 = 7'b0010010;
	      'h6: HEX2 = 7'b0000010;
	      'h7: HEX2 = 7'b1111000;
	      'h8: HEX2 = 7'b0000000;
	      'h9: HEX2 = 7'b0010000;
	      'ha: HEX2 = 7'b0001000;
	      'hb: HEX2 = 7'b0000011;
	      'hc: HEX2 = 7'b0100111;
	      'hd: HEX2 = 7'b0100001;
	      'he: HEX2 = 7'b0000110;
	      'hf: HEX2 = 7'b0001110;
		default: HEX2 = 7'b1000000; // "0" 
	 	endcase
	
	 	unique case(output1[15:12])
	      'h0: HEX3 = 7'b1000000;
	      'h1: HEX3 = 7'b1111001;
	      'h2: HEX3 = 7'b0100100;
	      'h3: HEX3 = 7'b0110000;
	      'h4: HEX3 = 7'b0011001;
	      'h5: HEX3 = 7'b0010010;
	      'h6: HEX3 = 7'b0000010;
	      'h7: HEX3 = 7'b1111000;
	      'h8: HEX3 = 7'b0000000;
	      'h9: HEX3 = 7'b0010000;
	      'ha: HEX3 = 7'b0001000;
	      'hb: HEX3 = 7'b0000011;
	      'hc: HEX3 = 7'b0100111;
	      'hd: HEX3 = 7'b0100001;
	      'he: HEX3 = 7'b0000110;
	      'hf: HEX3 = 7'b0001110;
		default: HEX3 = 7'b1000000; // "0" 
	 	endcase
		end
	
	assign LEDR[17:14] = portA[3:0];
	
	assign LEDR[13:10] = register[3:0];
	assign LEDR[0] = negFlag;
	assign LEDR[1] = oveFlag;
	assign LEDR[2] = zerFlag;
	
	
	
	endmodule
	
```
`借此alu逻辑算术单元也算完成了。

[image-1]:	https://res.cloudinary.com/djyodckal/image/upload/v1584759605/datapath_jchv0c.png
[image-2]:	https://res.cloudinary.com/djyodckal/image/upload/v1584871537/lab2__1_uwlege.png