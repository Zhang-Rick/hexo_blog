---
title: singlecycle设计及实现3(control unit控制单元)
date: 2020-03-24 01:06:04
categories: mips CPU design
tags:
- FPGA
- verilog
- CPU
---
### 简介
这篇博文主要介绍的是基于mips的 singlecycle CPU设计及实现的第三个单元control\_unit算术逻辑单元。
### RTL 图像
![][image-1]
接着上次博客对alu的介绍，本次本应该讲述memory储存的。涉及到储存，缓存就是性能提升的一个重要的步骤，但是由于缓存的内容比较多，所以打算单独放出来单独讨论。这次将讲解control\_unit控制单元。
### 控制单元详解
![][image-2]
依照上图，控制单元代表的是当完整的指令被解码的时候，具体要用哪些模块就会马上知道，所以控制这些模块的信号也会马上释放。

#### 举例
![][image-3]
例如我们解码的指令是ld rt，imm16(rs)
含义：取rs+imm16储存地址上的值，并把它赋给rt。
首先需要通过alu把实际地址算出来，然从储存读值，最后把值储存到寄存器中特定的地址。
1. 数据输入alu逻辑算术单元，busA不变，busB为带符号的立即执行，所以选择{16\*最高位,原来16位数字}。ALUsrc = signExT
2. 加法计算实际地址，ALUop = add
3. 从memory储存中读取，不是写。MemWr = 0，DataIn 无所谓
4. 选取memory输出的值。MemtoReg = 1，选储存出来的值
5. 对寄存器进行写入。RegWr = 1
6. 写入目标地址为rt。RegDst= rt
所以以上就是解码这个指令得到的所有的控制信号。所以如果要正确的处理所有的信号，需要理解所有信号的作用及自己构建的CPU的功能。

以下的是这个CPU所有需要包括的指令。
```c
`MIPS Instruction Set Architecture

---------------------<reserved registers>------------------------
$0                   zero
$1                   assembler temporary
$29                  stack pointer
$31                  return address
---------------------<R-type Instructions>-----------------------
ADDU   $rd,$rs,$rt   R[rd] <= R[rs] + R[rt] (unchecked overflow)
ADD    $rd,$rs,$rt   R[rd] <= R[rs] + R[rt]
AND    $rd,$rs,$rt   R[rd] <= R[rs] AND R[rt]
JR     $rs           PC <= R[rs]
NOR    $rd,$rs,$rt   R[rd] <= ~(R[rs] OR R[rt])
OR     $rd,$rs,$rt   R[rd] <= R[rs] OR R[rt]
SLT    $rd,$rs,$rt   R[rd] <= (R[rs] < R[rt]) ? 1 : 0
SLTU   $rd,$rs,$rt   R[rd] <= (R[rs] < R[rt]) ? 1 : 0
SLLV   $rd,$rs,$rt   R[rd] <= R[rt] << [0:4] R[rs]
SRLV   $rd,$rs,$rt   R[rd] <= R[rt] >> [0:4] R[rs]
SUBU   $rd,$rs,$rt   R[rd] <= R[rs] - R[rt] (unchecked overflow)
SUB    $rd,$rs,$rt   R[rd] <= R[rs] - R[rt]
XOR    $rd,$rs,$rt   R[rd] <= R[rs] XOR R[rt]
---------------------<I-type Instructions>-----------------------
ADDIU  $rt,$rs,imm   R[rt] <= R[rs] + SignExtImm (unchecked overflow)
ADDI   $rt,$rs,imm   R[rt] <= R[rs] + SignExtImm
ANDI   $rt,$rs,imm   R[rt] <= R[rs] & ZeroExtImm
BEQ    $rs,$rt,label PC <= (R[rs] == R[rt]) ? npc+BranchAddr : npc
BNE    $rs,$rt,label PC <= (R[rs] != R[rt]) ? npc+BranchAddr : npc
LUI    $rt,imm       R[rt] <= {imm,16b'0}
LW     $rt,imm($rs)  R[rt] <= M[R[rs] + SignExtImm]
ORI    $rt,$rs,imm   R[rt] <= R[rs] OR ZeroExtImm
SLTI   $rt,$rs,imm   R[rt] <= (R[rs] < SignExtImm) ? 1 : 0
SLTIU  $rt,$rs,imm   R[rt] <= (R[rs] < SignExtImm) ? 1 : 0
SW     $rt,imm($rs)  M[R[rs] + SignExtImm] <= R[rt]
LL     $rt,imm($rs)  R[rt] <= M[R[rs] + SignExtImm]; rmwstate <= addr
SC     $rt,imm($rs)  if (rmw) M[R[rs] + SignExtImm] <= R[rt], R[rt] <= 1 else R[rt] <= 0
XORI   $rt,$rs,imm   R[rt] <= R[rs] XOR ZeroExtImm
---------------------<J-type Instructions>-----------------------
J      label         PC <= JumpAddr
JAL    label         R[31] <= npc; PC <= JumpAddr
---------------------<Other Instructions>------------------------
HALT
---------------------<Pseudo Instructions>-----------------------
PUSH   $rs           $29 <= $29 - 4; Mem[$29+0] <= R[rs] (sub+sw)
POP    $rt           R[rt] <= Mem[$29+0]; $29 <= $29 + 4 (add+lw)
NOP                  Nop
-----------------------------------------------------------------
-----------------------------------------------------------------
org  Addr         Set the base address for the code to follow 
chw  #            Assign value to half word memory
cfw  #            Assign value to word of memory
```
`所以根据以上的指令集进行了归类，主要注意的是ALUsrc需要哪些input，做出了以下的表格进行整理。
```c
`/*
	|    Var    |    00     |    01     |    10     |    11    |

	|  ALUSrc   |    rs     |  unsign   |    sll    |   sign   |

	|  RegDst   |    rt     |    rd     |    31     |   zero   |

	|  PCSrc    |    no     |  Branch   |    JR     |  J/JAL   |

	|  DAtaSrc  |  portout  | dmemload  |    lui    |   PC+4   |
*/

```
`所以看了一下，数据线B总共有datab，没符号立即数，有符号立即数以及SLL这种只取寄存器其中的五位。
JAL这个指令需要跳转，同时把跳转回来的程序地址储存到31号寄存器中。
##### 主代码control\_unit.sv
```verilog
``include "cpu_types_pkg.vh"


`include "control_unit_if.vh"

module control_unit(
	control_unit_if.ctr cuif
);

/*
	|    Var    |    00     |    01     |    10     |    11    |

	|  ALUSrc   |    rs     |  unsign   |    slt    |   sign   |

	|  RegDst   |    rt     |    rd     |    31     |   zero   |

	|  PCSrc    |    no     |  Branch   |    JR     |  J/JAL   |

	|  DAtaSrc  |  portout  | dmemload  |    lui    |   PC+4   |
*/

import cpu_types_pkg::*;
 logic equFlag;
always_comb begin
	equFlag = 0;//beq bne flag

	cuif.PCSrc = 0;//pc mux
	cuif.DataSrc = 0;//memory to register mux
	cuif.ALUSrc = 0;//data bus 2 mux
	cuif.RegDest = 0;//destination register mux
	
	cuif.memREN = 0;
	cuif.memWEN = 0;
	cuif.RegWEN = 0;
	cuif.halt = 0;
    cuif.branch = 0;

	cuif.ALUOP = ALU_ADD;
	case(cuif.opcode)
		RTYPE: begin
			case(cuif.funct)
				SLLV: begin
					cuif.ALUOP = ALU_SLL;
					cuif.RegWEN = 1;
					cuif.RegDest = 1;
					cuif.ALUSrc = 2'b10;
				end
				SRLV: begin
					cuif.ALUOP = ALU_SRL;
					cuif.RegWEN = 1;
					cuif.RegDest = 1;
					cuif.ALUSrc = 2'b10;
				end
				JR:  begin
					cuif.PCSrc = 2'b10;
				end
				ADD: begin
					cuif.ALUOP = ALU_ADD;
					cuif.RegWEN = 1;
					cuif.RegDest = 1;
				end
				ADDU: begin
					cuif.ALUOP = ALU_ADD;
					cuif.RegWEN = 1;
					cuif.RegDest = 1;
				end
				SUB: begin
					cuif.ALUOP = ALU_SUB;
					cuif.RegWEN = 1;
					cuif.RegDest = 1;
				end
				SUBU: begin
					cuif.ALUOP = ALU_SUB;
					cuif.RegWEN = 1;
					cuif.RegDest = 1;
				end
				AND: begin
					cuif.ALUOP = ALU_AND;
					cuif.RegWEN = 1;
					cuif.RegDest = 1;
				end
				OR: begin
					cuif.ALUOP = ALU_OR;
					cuif.RegWEN = 1;
					cuif.RegDest = 1;
				end
				XOR: begin
					cuif.ALUOP = ALU_XOR;
					cuif.RegWEN = 1;
					cuif.RegDest = 1;
				end
				NOR: begin
					cuif.ALUOP = ALU_NOR;
					cuif.RegWEN = 1;
					cuif.RegDest = 1;
				end
				SLT: begin
					cuif.ALUOP = ALU_SLT;
					cuif.RegWEN = 1;
					cuif.RegDest = 1;
				end
				SLTU: begin
					cuif.ALUOP = ALU_SLTU;
					cuif.RegWEN = 1;
					cuif.RegDest = 1;
				end
		endcase
		end
	//j type	
	J: begin
		cuif.PCSrc = 2'b11;
	end
	JAL: begin
		cuif.PCSrc = 2'b11;
		cuif.RegDest = 2'b10;
		cuif.RegWEN = 1;	
		cuif.DataSrc = 2'b11;
	end

	//itype
	BEQ: begin
		equFlag = 1;
		cuif.ALUOP = ALU_SUB;
		cuif.PCSrc = 1;
		cuif.branch=cuif.zerFlag & equFlag;
	end
	BNE: begin
		equFlag = 1;
		cuif.ALUOP = ALU_SUB;
		cuif.PCSrc = 1;
		cuif.branch=cuif.zerFlag ^ equFlag;
	end		
	ADDI: begin
		cuif.RegWEN=1'b1;
		cuif.ALUSrc=2'b11;
		cuif.ALUOP=ALU_ADD;	
	end
	ADDIU: begin
		cuif.RegWEN=1'b1;
		cuif.ALUSrc=2'b11;
		cuif.ALUOP=ALU_ADD;	
	end			
	ORI: begin
		cuif.RegWEN=1'b1;
		cuif.ALUSrc=2'b1;
		cuif.ALUOP=ALU_OR;	
	end
	XORI: begin
	 	cuif.RegWEN=1'b1;
		cuif.ALUSrc=2'b1;
		cuif.ALUOP=ALU_XOR;	
	end	
	LUI: begin
		cuif.RegWEN=1'b1;
		cuif.DataSrc=2'b10;
	end
	LW: begin
	   cuif.ALUSrc=2'b11;
	   cuif.memREN=1'b1;
	   cuif.DataSrc=2'b1;
	   cuif.ALUOP=ALU_ADD;
	   cuif.RegWEN=1'b1;
	end			
	SLTI: begin
		cuif.RegWEN=1'b1;
		cuif.ALUSrc=2'b1;
		cuif.ALUOP=ALU_SLT;	
	end
	SLTIU: begin
		cuif.RegWEN=1'b1;
		cuif.ALUSrc=2'b1;
		cuif.ALUOP=ALU_SLTU;
	end		
	ANDI: begin
		cuif.RegWEN=1'b1;
		cuif.ALUSrc=2'b11;
		cuif.ALUOP=ALU_AND;	
	
	end
	SW: begin
	   cuif.ALUSrc=2'b11;
	   cuif.memWEN=1'b1;
	   cuif.DataSrc=2'b1;
	   cuif.ALUOP=ALU_ADD;
	   cuif.RegWEN=1'b0;
	end						
	HALT: begin
		cuif.halt=1'b1;
	end
endcase
end
endmodule			

```
`
##### 接口代码control\_unit\_if.vh
这次输入输出信号非常的多，所以写了接口代码。
```verilog
`
`ifndef CONTROL_UNIT_IF_VH
`define CONTROL_UNIT_IF_VH

// typedefs
`include "cpu_types_pkg.vh"

interface control_unit_if;
  // import types
  import cpu_types_pkg::*;
	
	opcode_t opcode;
	funct_t  funct;
    logic [1:0] PCSrc,DataSrc,ALUSrc,RegDest;
	logic memREN, memWEN, RegWEN, halt, zerFlag, branch, dhit, ihit;	
	logic [3:0] ALUOP;

modport ctr (
	input opcode, funct, zerFlag,dhit,ihit,
	output PCSrc,DataSrc,ALUSrc, memREN, memWEN, RegWEN, ALUOP, RegDest, halt, branch	
);

modport tb (
	input PCSrc,DataSrc,ALUSrc, memREN, memWEN, RegWEN, ALUOP, RegDest, halt, branch,
	output opcode, funct, zerFlag, dhit
);

endinterface
`endif


```
`
##### 接口代码control\_unit\_tb.sv
这次测试代码按照规范一个个写了assert来验证，但是很奇怪有些的assert就是对不上。
```verilog
`
`include "control_unit_if.vh"
`include "cpu_types_pkg.vh"
`timescale 1 ns / 1 ns
import cpu_types_pkg::*;
module control_unit_tb;
  
   parameter PERIOD = 10;
  
   logic CLK = 0, nRST;
  
   // clock
   always #(PERIOD/2) CLK++;

   //interface
   control_unit_if cuif();
	test21 prog(CLK,nRst,cuif); 
   //test program
   //test PROG (CLK, nRST, cuif);
`ifndef MAPPED
   control_unit DUT(cuif);
`endif
endmodule

program test21 (input logic CLK, 
	      output logic nRST,
	      control_unit_if.tb cuif);
int testcase =0;
initial begin
	cuif.opcode=RTYPE;
	cuif.funct=SLLV;
	cuif.zerFlag=1'b0;
        //test reset
	@(posedge CLK)
        testcase++;//test J type
	cuif.opcode=J;//J
	//cuif.funct=SLLV;
	cuif.zerFlag=1'b0;	
	@(posedge CLK)
	assert (cuif.PCSrc == 2'b11)else  $display("It's gone wrong addtime %00g ns",$time);
	@(posedge CLK)
        testcase++;//test JAL type	
	cuif.opcode=JAL;//JAL
	//cuif.funct='0;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
	assert (cuif.PCSrc == 2'b11 && cuif.RegDest == 2'b10 && cuif.RegWEN == 1 && cuif.DataSrc == 2'b11) else $display("It's gone wrong addtime %00g ns",$time);
	@(posedge CLK)
        testcase++;//test itype	
	cuif.opcode=BEQ;//BEQ
	//cuif.funct='0;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
	assert ( cuif.PCSrc == 1 && cuif.branch == 0) else $display("It's gone wrong addtime %00g ns",$time);
	@(posedge CLK)
        testcase++;//test itype	
	cuif.opcode=BNE;//BNE
	//cuif.funct='0;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert ( cuif.PCSrc == 1 && cuif.branch==1) else $display("It's gone wrong addtime %00g ns",$time);
	@(posedge CLK)
        testcase++;//test itype	
	cuif.opcode=ADDI;//ADDI
	//cuif.funct='0;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert (cuif.RegWEN==1'b1 && cuif.ALUSrc==2'b1) else $display("It's gone wrong addtime %00g ns",$time);
	@(posedge CLK)
        testcase++;//test itype	
	cuif.opcode=ANDI;//ANDI
	//cuif.funct='0;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert ( cuif.RegWEN==1'b1 && cuif.ALUSrc==2'b11) else $display("It's gone wrong addtime %00g ns",$time);
        @(posedge CLK)
        testcase++;//test itype	
	cuif.opcode=ORI;//ORI
	//cuif.funct='0;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert ( cuif.RegWEN==1'b1 && cuif.ALUSrc==2'b11)else $display("It's gone wrong addtime %00g ns",$time);
        @(posedge CLK)        
	testcase++;//test itype	
	cuif.opcode=LUI;//LUI
	//cuif.funct='0;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert (cuif.RegWEN==1'b1 && cuif.DataSrc==2'b10) else $display("It's gone wrong addtime %00g ns",$time);
	@(posedge CLK)
        testcase++;//test itype	
	cuif.opcode=HALT;//HALT
	//cuif.funct='0;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert (cuif.halt==1'b1)  else $display("It's gone wrong addtime %00g ns",$time);	
	@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=SLLV;//SLL
	cuif.opcode=RTYPE;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert ( cuif.RegWEN == 1 && cuif.RegDest == 1 && cuif.ALUSrc == 2'b10)else $display("It's gone wrong addtime %00g ns",$time);	
	@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=SRLV;//SLL
	cuif.opcode=RTYPE;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert ( cuif.RegWEN ==1 && cuif.RegDest == 1)else $display("It's gone wrong addtime %00g ns",$time);	
	@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=ADD;//SLL
	cuif.opcode=RTYPE;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert ( cuif.RegWEN == 1 && cuif.RegDest == 1 )else $display("It's gone wrong addtime %00g ns",$time);	
@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=ADDU;//SLL
	cuif.opcode=RTYPE;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert ( cuif.RegWEN == 1 && cuif.RegDest == 1 )else $display("It's gone wrong addtime %00g ns",$time);	
@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=SUBU;//SLL
	cuif.opcode=RTYPE;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert ( cuif.RegWEN == 1 )else $display("It's gone wrong addtime %00g ns",$time);	
@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=SUB;//SLL
	cuif.opcode=RTYPE;
	cuif.zerFlag=1'b0;

	@(posedge CLK)
assert ( cuif.RegWEN == 1 )else $display("It's gone wrong addtime %00g ns",$time);	
@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=AND;//SLL
	cuif.opcode=RTYPE;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert ( cuif.RegWEN == 1 )else $display("It's gone wrong addtime %00g ns",$time);	
@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=OR;//SLL
	cuif.opcode=RTYPE;
	cuif.zerFlag=1'b0;

	@(posedge CLK)
assert ( cuif.RegWEN == 1 && cuif.RegDest == 1 )else $display("It's gone wrong addtime %00g ns",$time);	
@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=XOR;//SLL
	cuif.opcode=RTYPE;
	cuif.zerFlag=1'b0;

	@(posedge CLK)
assert ( cuif.RegWEN == 1 && cuif.RegDest == 1 )  $display("It's gone wrong addtime %00g ns",$time);	
@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=NOR;//SLL
	cuif.opcode=RTYPE;
	cuif.zerFlag=1'b0;

	@(posedge CLK)
assert ( cuif.RegWEN == 1 && cuif.RegDest == 1 ) else $display("It's gone wrong addtime %00g ns",$time);	
@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=SLT;//SLL
	cuif.opcode=RTYPE;
	cuif.zerFlag=1'b0;

	@(posedge CLK)
assert ( cuif.RegWEN == 1 && cuif.RegDest == 1 ) else $display("It's gone wrong addtime %00g ns",$time);	
@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=SLTU;//SLL
	cuif.opcode=RTYPE;
	cuif.zerFlag=1'b0;

	@(posedge CLK)
if ( ~(cuif.RegWEN == 1 && cuif.RegDest == 1 && cuif.ALUSrc == 2'b10))  $display("It's gone wrong addtime %00g ns",$time);	
	@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=JR;//JR
	cuif.opcode=RTYPE;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert (cuif.PCSrc == 2'b10)   $display("It's gone wrong addtime %00g ns",$time);	
	@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=SUB;//SUB
	cuif.opcode=RTYPE;
	cuif.zerFlag=1'b0;
@(posedge CLK)
assert (cuif.PCSrc == 2'b1 && cuif.RegWEN == 1 ) else $display("It's gone wrong addtime %00g ns",$time);	
	@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=SUB;//SUB
	cuif.opcode=ADDIU;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert (cuif.PCSrc == 2'b1)  $display("It's gone wrong addtime %00g ns",$time);	

	@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=SUB;//SUB
	cuif.opcode=SLTI;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert (cuif.PCSrc == 2'b1)  $display("It's gone wrong addtime %00g ns",$time);	
	@(posedge CLK)	
	@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=SUB;//SUB
	cuif.opcode=SLTIU;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert (cuif.PCSrc == 2'b1) else $display("It's gone wrong addtime %00g ns",$time);	
	@(posedge CLK)	
	@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=SUB;//SUB
	cuif.opcode=XORI;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert (cuif.ALUSrc == 2'b11)  $display("It's gone wrong addtime %00g ns",$time);	
	@(posedge CLK)	
	@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=SUB;//SUB
	cuif.opcode=LW;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert (cuif.ALUSrc == 2'b11 &&  cuif.DataSrc == 1 )  $display("It's gone wrong addtime %00g ns",$time);	
	@(posedge CLK)
	@(posedge CLK)
        testcase++;//test Rtype	
	cuif.funct=SUB;//SUB
	cuif.opcode=SW;
	cuif.zerFlag=1'b0;
	@(posedge CLK)
assert (cuif.ALUSrc == 2'b11 &&  cuif.DataSrc == 1)  $display("It's gone wrong addtime %00g ns",$time);	

	@(posedge CLK)
		cuif.zerFlag=1'b1;
	@(posedge CLK)
		cuif.zerFlag=1'b0;	
	@(posedge CLK)
		cuif.ihit=1'b0;	
		cuif.dhit=1'b0;	
	@(posedge CLK)
		cuif.ihit=1'b1;	
		cuif.dhit=1'b1;
	@(posedge CLK)
$finish;			       
end
endprogram
```
`以上就是控制单元的代码，接下来就会是一个总的单元用于处理所有小的模块和前面写过的模块。

[image-1]:	https://res.cloudinary.com/djyodckal/image/upload/v1584759605/datapath_jchv0c.png
[image-2]:	https://res.cloudinary.com/djyodckal/image/upload/v1585026711/image001_mtevmx.jpg
[image-3]:	https://res.cloudinary.com/djyodckal/image/upload/v1585027156/%E5%9B%BE%E7%89%87243_vc15c2.png