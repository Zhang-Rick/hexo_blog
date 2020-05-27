---
title: singlecycle设计及实现1(register file寄存器)
date: 2020-03-20 22:50:52
Categories: mips CPU design
tags: 
- FPGA
- verilog
- CPU
---

### 简介
这篇博文主要介绍的是基于mips的 singlecycle CPU设计及实现。
### RTL 图像
![][image-1]
因为刚开始写的时候只考虑了mips最基础的指令，没有考虑lui和跳转等指令，所以这是一个非常粗略的图像。主要由几个大块组成，register file寄存器，ALU算术逻辑单元，memory储存，control unit控制单元，instruction fetch指令获取单元，PC 程序计数器和decoder。由于decoder已经编写，所以这次设计直接从储存中读取opcode来识别指令，iload来识别各个寄存器的地址。接下来一一介绍每个单元。
#### register file 寄存器
![][image-2]
如图所示上图展示了寄存器的一些输入及输出。
输入:
WDAT(31:0) 32位的输入
WSEL(4:0) 寄存器地址选择
WEN 寄存器开关 1:开 0:关
CLK 寄存器时钟
RSEL1(4:0) 数据总线1 地址选择
RSEL2(4:0) 数据总线2 地址选择
输出:
RDAT1(31:0) 数据总线1值
RDAT2(31:0) 数据总线2值

以前写verilog代码的时候，并没有用过接口，这次因为是要求，所以更具接口的方式来编写。
##### 主代码register\_file.sv
```verilog
`	`include "cpu_types_pkg.vh”//cpu总接口文件
	`include "register_file_if.vh”//寄存器接口文件
	module register_file
	
	(
	input logic CLK, nRST,
	register_file_if.rf rfif//接口中 rf端口
	);
	
	//mips有32个32位的寄存器，并且需要放入reg
	reg \[31:0]\[31:0] register;//输出的寄存器
	reg \[31:0]\[31:0] nregister;//输入的寄存器
	always\_comb begin
	nregister = register;
	if (rfif.WEN && rfif.wsel != 0) begin//mips 0寄存器不能修改并且要在打开的情况才能修改
	    nregister[rfif.wsel] = rfif.wdat;
	end
	rfif.rdat1 = register[rfif.rsel1];
	rfif.rdat2 = register[rfif.rsel2];   
	end
	
	always\_ff @(posedge CLK, negedge nRST) begin//异步同步
	if (!nRST)
	    register <= '0;
	else
	    register <= nregister;
	end
	
	endmodule
```
`
##### 接口代码register\_file\_if.vh
 ```verilog
`/*
  Eric Villasenor

  register file interface
*/
`ifndef REGISTER_FILE_IF_VH
`define REGISTER_FILE_IF_VH

// all types
`include "cpu_types_pkg.vh"

interface register_file_if;
  // import types
  import cpu_types_pkg::*;

  logic     WEN;
  regbits_t wsel, rsel1, rsel2;//在cpu_types_pkg.vh中定义 5，2^5 = 32
  word_t    wdat, rdat1, rdat2;//在cpu_types_pkg.vh中定义32

  // register file ports
  modport rf (
    input   WEN, wsel, rsel1, rsel2, wdat,
    output  rdat1, rdat2
  );
  // register file tb
  modport tb (
    input   rdat1, rdat2,
    output  WEN, wsel, rsel1, rsel2, wdat
  );
endinterface

`endif //REGISTER_FILE_IF_VH

 ```
`
##### 测试代码register\_file\_tb.sv
```verilog
`	/\*
	 Eric Villasenor
	
	 register file test bench
	\*/
	
	// mapped needs this
	\`include "register\_file\_if.vh"
	
	// mapped timing needs this. 1ns is too fast
	\`timescale 1 ns / 1 ns
	
	module register\_file\_tb;
	
	 parameter PERIOD = 10;
	
	 logic CLK = 0, nRST;
	
	 // test vars
	 int v1 = 1;
	 int v2 = 4721;
	 int v3 = 25119;
	
	 // clock
	 always #(PERIOD/2) CLK++;
	
	 // interface
	 register\_file\_if rfif ();
	 // test program
	 test PROG (CLK, nRST,rfif,v1,v2,v3);
	 // DUT
	\`ifndef MAPPED
	 register\_file DUT(CLK, nRST, rfif);
	\`else
	 register\_file DUT(
	.\rfif.rdat2 (rfif.rdat2),
	.\rfif.rdat1 (rfif.rdat1),
	.\rfif.wdat (rfif.wdat),
	.\rfif.rsel2 (rfif.rsel2),
	.\rfif.rsel1 (rfif.rsel1),
	.\rfif.wsel (rfif.wsel),
	.\rfif.WEN (rfif.WEN),
	.\nRST (nRST),
	.\CLK (CLK)
	 );
	\`endif
	
	endmodule
	
	program test
	(
	input logic CLK,
	output logic nRST,
	
	register\_file\_if.tb rfif,
	input logic \[31:0] v1,
	input logic \[31:0] v2,
	input logic \[31:0] v3
	);
	reg [5:0] i;
	  reg \[3:0] testcase = 0;
	  initial begin
	rfif.wsel = 0;
	rfif.wdat = '0;
	rfif.WEN = 0;
	rfif.rsel1 = 0;
	rfif.rsel2 = 0;      
	@(posedge CLK)
	nRST = 1'b0;
	
	
	@(posedge CLK)
	nRST = 1'b0;
	
	@(posedge CLK)
	nRST = 1'b1;
	rfif.wsel = 0;
	rfif.WEN = 1;
	rfif.wdat = '1;
	rfif.rsel1 = 0;
	rfif.rsel2 = 1;
	testcase ++;
	@(posedge CLK)
	nRST = 1'b0;
	@(posedge CLK)
	nRST = 1'b1;
	for (i = 0; i < 32; i = i + 1)
	begin
	    @(posedge CLK)
	    rfif.wsel = i;
	    @(posedge CLK)
	    rfif.rsel1 = i;
	    rfif.rsel2 = 31-i;
	end
	@(posedge CLK)
	nRST = 1'b1;
	rfif.wsel = 1;
	rfif.WEN = 1;
	rfif.wdat = '0;
	rfif.rsel1 = 0;
	rfif.rsel2 = 1;
	testcase ++;
	@(posedge CLK)
	nRST = 1'b0;
	@(posedge CLK)
	nRST = 1'b1;
	for (i = 0; i < 32; i = i + 1)
	begin
	    @(posedge CLK)
	    rfif.wsel = i;
	    @(posedge CLK)
	    rfif.rsel1 = i;
	    rfif.rsel2 = 31-i;
	end
	@(posedge CLK)
	nRST = 1'b0;
	@(posedge CLK)
	nRST = 1'b1;
	rfif.wsel = 1;
	rfif.WEN = 1;
	rfif.wdat = '1;
	rfif.rsel1 = 1;
	rfif.rsel2 = 0;
	testcase ++;
	@(posedge CLK)
	nRST = 1'b0;
	@(posedge CLK)
	nRST = 1'b1;
	for (i = 0; i < 32; i = i + 1)
	begin
	    @(posedge CLK)
	    rfif.wsel = i;
	    @(posedge CLK)
	    rfif.rsel1 = i;
	    rfif.rsel2 = 31-i;
	end
	@(posedge CLK)
	@(posedge CLK)
	nRST = 1'b0;
	@(posedge CLK)
	@(posedge CLK)
	$finish;
	end
	endprogram
```
`测试文件用questasim来验证，本来应该用assert更规范的测试，第一次懒惰了。
 以上是singlecycle中寄存器文件的

[image-1]:	https://res.cloudinary.com/djyodckal/image/upload/v1584759605/datapath_jchv0c.png
[image-2]:	https://res.cloudinary.com/djyodckal/image/upload/v1584760463/registerDiagram_djizyf.png