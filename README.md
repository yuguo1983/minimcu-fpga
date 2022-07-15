---
typora-root-url: ./
typora-copy-images-to: images
---

# mini-mcu简介

为了快速在FPGA上开发一些应用，且占用资源较少，所以设计和xilinx退出的KSPMS6指令基本一样的软核，并且实现指令可裁剪。项目仿真使用的是`iverilog和gtkwave`，指令设计和下面一样，这样自己就不用开发编译软件或者脚本！个人计划是设计这个`mini-mcu`,解决fpga上一些低速传感器的数据采集和简单的外设控制的，为了方便移植，设计的所有结构都是使用逻辑资源做的。



我使用bash写了自动根据编写的代码裁剪`mini-mcu`，也就是说，最终`mini-mcu`只会生成你是用过的命令的对应电路，这样做可以减少逻辑资源的开销，自己这次并不打算为其设计外部的外设和数据总线，只是用其做简单的模块，以加快项目开发。





# 计划指令集

| 指     令            | 描述                                                         | 功能                                     | 标志位                 | 备注                            |
| -------------------- | ------------------------------------------------------------ | ---------------------------------------- | ---------------------- | ------------------------------- |
| ADD sX,  kk          | 寄存器sX加立即数 kk                                          | sX = sX + kk                             | 影响进位C和结果标志位Z | kk:一个字节                     |
| ADD sX,  sY          | 寄存器sX和sY相加                                             | sX=sX + sY                               | 影响进位C和结果标志位Z | X,Y为0-F，表示16个寄存器        |
| ADDCY sX,  kk        | 寄存器sX加立即数 kk和进位位                                  | sX = sX + kk + C                         | 影响进位C和结果标志位Z | 进位标志位：C                   |
| ADDCY sX,  xY        | 寄存器sX，sY，进位相加                                       | sX=sX + sY +C                            | 影响进位C和结果标志位Z |                                 |
| AND sX, kk           | sX与立即数kk逻辑与                                           | sX = sX & kk                             | C=0,影响结果标志Z      | kk:一个字节常数                 |
| AND sX, sY           | sX与sY逻辑与                                                 | sX = sX & sY                             | C=0,影响结果标志Z      | X,Y为0-F，表示16个寄存器        |
| CALL  aaa            | 无条件调用地址为aaa的子程序                                  | TOS=PC, PC=aaa                           | 不影响当前标志位       | aaa为000-3FF，10位地址          |
| CALL C, aaa          | 如果进位标志C=1，调用地址为aaa的子程序                       | 如果C==1，{TOS=PC, PC=aaa}               | 不影响当前标志位       | aaa为000-3FF，10位地址          |
| CALL NC, aaa         | 如果进位标志C=0，调用地址为aaa的子程序                       | 如果C==0，{TOS=PC, PC=aaa}               | 不影响当前标志位       | aaa为000-3FF，10位地址          |
| CALL NZ, aaa         | 如果结果标志Z=0，调用地址为aaa的子程序                       | 如果Z==0，{TOS=PC, PC=aaa}               | 不影响当前标志位       | aaa为000-3FF，10位地址          |
| CALL Z, aaa          | 如果结果标志Z=1，调用地址为aaa的子程序                       | 如果Z==1，{TOS=PC, PC=aaa}               | 不影响当前标志位       | aaa为000-3FF，10位地址          |
| COMPARE sX, kk       | sX与立即数kk比较                                             | 如果sX==kk，ZERO=1,如果sX<kk,C = 0       | 影响标志位             |                                 |
| COMPARE sX, sY       | sX与sY比较                                                   | 如果sX==sY，ZERO=1,如果sX<sY,C = 0       | 影响标志位             |                                 |
| DISABLE INTERRUP     | 禁止中断响应                                                 | 中断使能标志清零                         | 不影响当前标志位       |                                 |
| ENAABLE INTERRUP     | 使能中断响应                                                 | 中断标志设为1                            | 不影响当前标志位       |                                 |
| INTERRUPT Event      | 异步中断输入，保护标志和程序计数器，清除INTERUPT_ENABLE标志。跳转到3FF的中断矢量地址 | 当下状态入栈，PC=3FF                     | 不影响当前标志位       |                                 |
| FETCH sX, (sY)       | sY所指的中间寄存器内容读取到sX的寄存器内                     | sX = RAM[(sY)]                           | 不影响当前标志位       |                                 |
| FETCH sx， ss        | 6位地址ss所指定的中间寄存器内容度发哦sX寄存器内              | sX=RAM[ss]                               | 不影响当前标志位       | ss表示6位地址，可访问64个寄存器 |
| **INPUT sX，（sY）** | sY寄存器所指定的输入口地址的内容读到sX                       | PORT_ID=sY,sX =IN_PORT                   | 不影响当前标志位       | sY表示8位IO的地址，最多256个    |
| INPUT sX，pp         | 输入口地址为pp的内容读到sX寄存器内                           | PORT_ID=pp,sX =IN_PORT                   | 不影响当前标志位       | pp表示8位IO的地址，最多256个    |
| JUMP aaa             | 无条件跳转到aaa的程序                                        | PC=aaa                                   | 不影响当前标志位       | aaa为000-3FF,10位地址           |
| JUMP C,aaa           | 如果进位C为1，跳转到地址为aaa的程序                          | 如果C==1，PC=aaa                         | 不影响当前标志位       | aaa为000-3FF,10位地址           |
| JUMP NC,aaa          | 如果进位C为0，跳转到地址为aaa的程序                          | 如果C==0，PC=aaa                         | 不影响当前标志位       | aaa为000-3FF,10位地址           |
| JUMP NZ,aaa          | 如果结果标志位为0，跳转到地址为aaa的程序                     | 如果Z==0,PC=aaa                          | 不影响当前标志位       | aaa为000-3FF,10位地址           |
| JUMP Z,aaa           | 如果结果标志位为1，跳转到地址为aaa的程序                     | 如果Z==1,PC=aaa                          | 不影响当前标志位       | aaa为000-3FF,10位地址           |
| LOAD sX，kk          | 立即数kk送到sX寄存器                                         | sX=kk                                    | 不影响当前标志位       | kk为一个字节                    |
| LOAD sX，sY          | sY寄存器的值送到sX寄存器                                     | sX=sY                                    | 不影响当前标志位       |                                 |
| OR sX，kk            | sX与立即数kk逻辑或                                           | sX=sX \| kk                              | C=0，影响结果标志位Z   | kk为一个字节                    |
| OR sX，sY            | sX与立sY逻辑或                                               | sX=sX \| sY                              | C=0，影响结果标志位Z   | X,Y为0-F，代表16个寄存器        |
| OUTPUT sX，（sY）    | sX寄存器的内容送到输出端口地址为sY上                         | PORT_ID = sY，OUT_PORT=sX                | 不影响当前标志位       | sY表示8位地址                   |
| OUTPUT sX，pp        | sX寄存器的内容送到指定地址上                                 | PORT_ID=pp，OUT_PORT=sX                  | 不影响当前标志位       |                                 |
| RETURN               | 从子程序无条件返回                                           | PC=TOS+1                                 | 不影响当前标志位       |                                 |
| RETURN C             | 如果进位标志C为1，从子程序返回                               | 如果C==1，PC=TOS+1                       | 不影响当前标志位       |                                 |
| RETURN NC            | 如果进位标志C为0，从子程序返回                               | 如果C==0，PC=TOS+1                       | 不影响当前标志位       |                                 |
| RETURN NZ            | 如果结果标志Z为0，从子程序返回                               | 如果Z==0，PC=TOS+1                       | 不影响当前标志位       |                                 |
| RETURN Z             | 如果结果标志Z为1，从子程序返回                               | 如果Z==1，PC=TOS+1                       | 不影响当前标志位       |                                 |
| RETURNI ENABLE       | **从中断服务程序返回，同时重新使能中断**                     | PC=TOS,恢复标志Z,C,中断使能标志为1       | 影响Z和C               |                                 |
| RETURNI DISABLE      | **从中断服务程序返回，同时禁用中断**                         | PC=TOS,恢复标志Z,C,中断使能标志为0       | 影响Z和C               |                                 |
| RL sX                | 循环左移sX寄存器                                             | sX = {sX[6:0],sX[7]}, C=sX[7]            | 影响C和Z               |                                 |
| RR sX                | 循环右移sX寄存器                                             | sX = {sX[0],sX[7:1]},  C=sX[0]           | 影响C和Z               |                                 |
| SL0 sX               | 左移sX寄存器，最低位置0                                      | sX = {sX[6:0],0}, C=sX[7]                | 影响C和Z               |                                 |
| SL1 sX               | 左移sX寄存器，最低位置1                                      | sX = {sX[6:0],1}, C=sX[7]                | 影响C和Z               |                                 |
| SLA sX               | **sX寄存器含进位位循环左移**                                 | sX = {sX[6:0],C}, C=sX[7]                | 影响C和Z               |                                 |
| SLX sX               | sX寄存器循环左移，最低位维持                                 | sX = {sX[6:0],sX[0]}, C=sX[7]            | 影响C和Z               |                                 |
| SR0 sX               | 右移sX寄存器，最高位置0                                      | sX={0,sX[7:1]}, C=sX[0]                  | 影响C和Z               |                                 |
| SR1 sX               | 右移sX寄存器，最高位置1                                      | sX={1,sX[7:1]}, C=sX[0]                  | 影响C和Z               |                                 |
| SRA sX               | 含进位标志右移sX寄存器                                       | sX={C,sX[7:1]}, C=sX[0]                  | 影响C和Z               |                                 |
| SRX, sX              | 右移sX寄存器，最高位维持                                     | sX={sX[7],sX[7:1]}, C=sX[0]              | 影响C和Z               |                                 |
| STORE sX, ss         | 将sX寄存器的内容写到6位地址ss所指定的中间寄存器              | RAM[ss] = sX                             | 不影响当前标志位       | ss表示6位地址，可访问64个寄存器 |
| STORE sX, (sY)       | 将sX寄存器的内容写到寄存器sY所指向的中间寄存器               | RAM[(sY)] = sX                           | 不影响当前标志位       |                                 |
| SUB sX, kk           | sX减去立即数kk                                               | sX =sX-kk                                | 影响C和Z               | kk为8位常数                     |
| SUB sX, sY           | sX减去sY                                                     | sX=sX-sY                                 | 影响C和Z               |                                 |
| SUBCY sX, kk         | sX减去立即数kk和进位位                                       | sX=sX-kk-C                               | 影响C和Z               | kk为8位常数                     |
| SUBCY sX, sY         | sX减去sY和进位位                                             | sX=sX-sY-C                               | 影响C和Z               |                                 |
| TEST sX, kk          | 测试寄存器sX的内容是否与立即数kk相反                         | 如果(sX & kk) ==0,ZERO=1,C=奇数(sX & kk) | 影响C和Z               | 仅仅影响标志位                  |
| TEST sX, sY          | 测试寄存器sX的内容是否sY相反                                 | 如果(sX & sY) ==0,ZERO=1,C=奇数(sX & sY) | 影响C和Z               | 仅仅影响标志位                  |
| XOR sX，kk           |                                                              | sX=sX ^ kk                               | C=0,影响Z              |                                 |
| XOR sX，sY           |                                                              | sX=sX ^ sY                               | C=0,影响Z              |                                 |
|                      |                                                              |                                          |                        |                                 |



![image-20201125221152759](/images/image-20201125221152759.png)



![](/images/image-20201125221130630.png)



上述指令并没有全部实现,之后有时间在实现！

# 文件结构

```
.
├── images 放图片
├── mcu.v  设计的微控制器
├── README.md 说明
├── rom.v  程序存储器
├── run 命令仿真脚本
├── sim 仿真生成的文件夹
│   ├── wave
│   └── wave.lxt2
├── software 为项目开发的汇编程序
│   └── led_water.psm
├── tags 
├── tb.v 仿真testbech文件
└── tools 项目需要的一些基本工具
    ├── bin 一些自己写的脚本或者其他可执行文件
    │   └── msim
    └── kcpsm 汇编编译工具

```



为了确保脚本正常运行，你需要保证项目路径下tools文件夹存在，software文件夹下有语法没毛病的psm文件，还有项目根目录下存在rrun脚本。根目录下的tmp文件夹是仿真时自动生成的！



# 关于仿真

项目仿真使用小巧的iverilog工具，同样我为其写了脚本，使得开发支持一条命令跑到底的方式，根目录下的verilog文件包含`tb.v,mcu.v,rom.v,head.v`这几个文件，sofaware文件夹用于存放psm代码，为了开发一个小项目，使用这个项目的流程为：

假设我要为`step-fpga mox2 -c `开发一个按键检测项目，实现按键按一次，led灯翻转一次：

## step1: 编写汇编代码

在software一级目下简历一个psm文件，编写汇编代码，比如我的按键检测代码：

```assembly
;系统时钟为12MHz
;目标硬件为 小脚丫FPGA step-maxo2-c，这个型号是U盘模式，流文件会下载到mcu，每次上电由mcu配置FPGA

;实现按键检测，按一下key1，led1将会翻转一次，程序简单，因此注意手不要抖，正常按法按
;输入
constant sw_port,00 ;定义按键四段拨码开关  【按键 ： 开关 】

;输出
constant seg_port,00 ;定义数码管地址
constant led_port,01 ;定义led_port为常量01
constant rgb_port,02  ; rgb灯


start:
	load sA,FF ;  led等控制
    output sA,led_port
    load sB,00 ; 初始化数码管显示 
    load sC,FF  ; ' rgb 灭
    output sC,rgb_port ;rgb
    input sD,sw_port ; 读一次io口
loop:
	input sF,sw_port
	AND sF,00010000'b ;'
	COMPARE sF,00
	JUMP Z,keycheck
	jump	loop ;循环

keycheck:
	CALL delay_200ms
	input sF,sw_port
	AND sF,00010000'b ;'
	COMPARE sF,00
	CALL Z,led_sh
	JUMP loop 

led_sh:
	XOR sA,00010000'b ;'
    OUTPUT sA,led_port
	RETURN 

delay_200ms: LOAD s2, 03      ; 
             LOAD s1, a9
             LOAD s0, 80
             jump software_delay	

delay_500ms: 	LOAD s2, 09      ;  500000us / (1/1.2us)  --> 计数次数
                LOAD s1, 27
                LOAD s0, c0
                jump software_delay	

software_delay: LOAD s0, s0             ;pad loop to make it 10 clock cycles (5 instructions), if clk 12MHz  --> 1/1.2 us
                SUB s0, 01
                SUBCY s1, 00
                SUBCY s2, 00
                JUMP NZ, software_delay
                RETURN 
```



然后将终端路径切换到项目的根目录下。

## step2: 编译并自动裁剪mcu

到根目录下，终端中执行`./run -c && ./run`命令，将会将汇编代码便以为十六进制的数据文件，这些十六进制数据就是机器执行指令，会被放到ROM中，但是这都不用担心，我使用bash shell编写了脚本可以自动生成，并自动裁剪。

执行命令后如果不出错，会在根目录下生成tmp文件夹，或者清空原来的重新生成文件，hex文件使我们需要的，脚本会将hex文件中内容直接写入到根目录下的`rom.v`文件，并分析文件，生成用于裁剪`mini-mcu`的宏定义文件，为`head.v`,如生成的`head.v`文件内容如下：

```verilog
`define _LOAD_REG2REG_
`define _LOAD_DAT2REG_
`define _AND_KK_
`define _XOR_KK_
`define _INPUT_PP_
`define _SUB_KK_
`define _SUB_CY_KK_
`define _COMPARE_KK_
`define _CALL_
`define _JUMP_AAA_
`define _RETURN_
`define _OUTPUT_PP_
`define _CALL_Z_
`define _JUMP_Z_
`define _JUMP_NZ_

```

`mcu.v`文件会根据这些宏定义选择性的被条件编译，从而实现裁剪`mini-mcu`，之后为完善指令，这些脚本同样可用，但是需要注意，添加指令的位置为：

```verilog
//---------------------- 定义一级指令编码 --------------------
// --------  SCRIPT_BEGIN         --> 脚本识别标签，勿动
localparam
//LOAD
    LOAD_REG2REG = 6'H00,               //LOAD sX, sY    装载数据 
    LOAD_DAT2REG = 6'H01,               //LOAD sX, kk    装载立即数到寄存器
//JUMP
    JUMP_AAA     = 6'H22,               //JUMP aaa       跳转到aaa地址执行  
    JUMP_Z       = 6'H32,               //JUMP Z,aaa    
    JUMP_NZ      = 6'H36,               //JUMP NZ,aaa      
    JUMP_C       = 6'H3A,               //JUMP C,aaa    
    JUMP_NC      = 6'H3E,               //JUMP NC,aaa 
//INPUT
    INPUT_REG    = 6'H08,               //INPUT sX, (sY)  
    INPUT_PP     = 6'H09,               //INPUT sX, pp
//OUTPUT
    OUTPUT_REG   = 6'H2C,               //OUTPUT sX,(sY)
    OUTPUT_PP    = 6'H2D,               //OUTPUT sX,pp
//SHIFT 
    SHIFT        = 6'H14,                //移位操作
// ADD
    ADD_REG      = 6'H10,               // ADD sX,sY
    ADD_KK       = 6'H11,               // ADD sX,kk
    ADD_CY_REG   = 6'H12,               // ADDCY sX,sX
    ADD_CY_KK    = 6'H13,               // ADDCY sX,kk
// SUB
    SUB_REG      = 6'H18,               // SUB sX,sY
    SUB_KK       = 6'H19,               // SUB sX,kk
    SUB_CY_REG   = 6'H1A,               // SUBCY sX,sY
    SUB_CY_KK    = 6'H1B,               // SUBCY sX,kk
// RETURN
    RETURN       = 6'H25,               // RETURN 无条件返回
    RETURN_Z     = 6'H31,               // RETURN Z    Z==0 则返回
    RETURN_NZ    = 6'H35,               // RETURN NZ   Z==1 则返回
    RETURN_C     = 6'H39,               // RETURN C    C==0 则返回
    RETURN_NC    = 6'H3D,               // RETURN NC   C==1 则返回
//CALL 
    CALL         =  6'H20,                // CALL aaa
    CALL_Z       =  6'H30,                // CALL Z, aaa
    CALL_NZ      =  6'H34,                // CALL NZ, aaa
    CALL_C       =  6'H38,                // CALL C, aaa
    CALL_NC      =  6'H3C,                // CALL NC, aaa
// COMPARE
    COMPARE_REG  =  6'H1C,                // COMPARE sX,sY   if(sX < kk or sX < sY) Z=X,C=1  ;  if(sX > kk or sX > sY) Z=X,C=0  ; if(sX == kk or sX == sY) Z=1,C=x ;
    COMPARE_KK   =  6'H1D,                // COMPARE sX,kk   if(sX < kk or sX < sY) Z=X,C=1  ;  if(sX > kk or sX > sY) Z=X,C=0  ; if(sX == kk or sX == sY) Z=1,C=x ;
    COMPARECY_REG=  6'H1E,               // COMPARECY sX,sY
    COMPARECY_KK =  6'H1F,               // COMPARECY sX,kk
// TSET
    TEST_REG     = 6'H0C,                // TEST sX,sY
    TEST_KK      = 6'H0D,                // TEST sX,kk
    TESTCY_REG   = 6'H0E,                // TESTCY sX,sY
    TESTCY_KK    = 6'H0F,                // TESTCY sX,kk
// LOGIC
    AND_REG      = 6'H02,                // AND sX,sY
    AND_KK       = 6'H03,                // AND sX,kk
    OR_REG       = 6'H04,                // OR sX,sY
    OR_KK        = 6'H05,                // OR sX,kk  
    XOR_REG      = 6'H06,                // XOR sX,sY
    XOR_KK       = 6'H07;                // XOR sX,kk     

// --------  SCRIPT_END         --> 脚本识别标签，勿动
```

要在上面的格式上进行指令扩展，脚本识别标签勿动，这是用来根据指令自动生成裁剪mcu宏定义的关键内容！

编译完毕，将会自动打开`gtkwave`显示波形。

![image-20201127030217652](/images/image-20201127030217652.png)





## step3: 拷贝文件到工程

设计好后，可以自行拷贝`rom.v,  mcu.v,   head.h `到项目，连接号个信号线即可。比如我项目中使用`mini-mcu`的顶层文件为：

```verilog
module top
(
	input clk_in,             //输入系统12MHz时钟
	//4bit拨码开关输入
	input [3:0] sw,
	input [3:0] key, //按键输入
	//数码管
	output [8:0] seg_led_1,
	output [8:0] seg_led_2,
	//rgb
	output reg[2:0]rgb,
	//led
	output led1,              
	output led2,
	output led3,
	output led4,
	output led5,
	output led6,
	output led7,
	output led8
); 

wire clk ,clko,rst;
reg [7:0] out;
assign {led8,led7,led6,led5,led4,led3,led2,led1} = out;
assign clk = clk_in;
reg rst_n_in;          //复位信号
reg [17:0]cnt ;
always @(posedge clk) begin
		if(cnt>=18'h3ffff)begin
			rst_n_in <= 1'b1;
		end else begin
		    cnt <= cnt +1;
			rst_n_in <= 0;
		end
end 
/*
divide #(
	.N(1)
) u1 (
	.clk(clko),
	.rst_n(rst_n_in),
	.clkout(clk)
); 
*/
//----------- mini-mcu 相关------------
wire [11:0]address;
wire [17:0] instruction;
wire bram_enable, read_strobe, write_strobe;
reg [7:0] in_port;
wire [7:0] port_id, out_port;
//----------- 数码管 相关------------
reg[3:0] seg_data_1, seg_data_2;
//输出引脚 
always @(posedge clk )begin
		if(write_strobe)begin
			case(port_id)
				8'h00:{seg_data_1,seg_data_2} <= out_port;//bcd编码的2个数码管
				8'h01:out <= out_port;  //LED控制
				8'h02:rgb <= out_port[2:0];  //rgb
				default:out <=out;
			endcase
		end else out <= out;
end
//输入引脚

always@(*)begin
	if(read_strobe) begin
		case(port_id)
				8'h00: in_port = {key[3:0],sw[3:0]};   //按键 4bit拨码开关输入
				
		endcase
	end 

end	

/**********************************
* 例化mini-mcu
**********************************/
mcu mcu(
    .clk(clk),            //系统时钟
    .rst_n(	rst_n_in),          //复位  0 --> 复位
    .address(	address),       //程序取址地址
    .instruction(	instruction),   //指令输入
    .bram_enable(	bram_enable),   //程序rom使能 1-->使能    
    .in_port(	in_port),        //输入口
    .read_strobe(	read_strobe),    //输入口使能     
    .port_id( port_id),        //io口地址
    .out_port(	out_port),       //输出口
    .write_strobe(	write_strobe)   //输出口写使能  
);

rom rom(
    .clk(	clk),
    .address(	address),       //程序取址地址
    .instruction( 	instruction),   //指令输入
    .enable( 	bram_enable)   //程序rom使能 1-->使能 
);
/**********************************
*数码管显示 是bcd码 
**********************************/
seg_display seg_display(
	.seg_data_1(seg_data_1),
	.seg_data_2(seg_data_2),
	.seg_led_1(seg_led_1),
	.seg_led_2(seg_led_2)
);

endmodule
```

## step4：使用和我一样的开发板

小脚丫FPGA  step- MOX2-C，不带TAGE，是U盘模式，拷贝文件就相当下载配置文件，使用这个是最方便的，因为我写了脚本，可以试试在根目录下执行`./run -g`试一试，这个命令将会编译汇编，生成rom文件，裁剪mini-mcu，拷贝有用文件到项目工程路径，工具综合，下载到开发板整个过程完成！只需要稍微调整脚本即可实现。

![image-20201127031143032](/images/image-20201127031143032.png)



![image-20201127031207445](/images/image-20201127031207445.png)



![image-20201127031303848](/images/image-20201127031303848.png)



# 设计记录

## 基本输入输出指令

**汇编代码**

```assembly
;系统时钟为50MHz
constant led_port,01 ;定义led_port为常量01
constant led_on,00000010'b ;
constant led_off,00000001'b ;
start:
	load	sF,	led_on;
	output	sF,led_port ;led亮
	load	sF,	led_off;
	output	sF,led_port ;led灭
	jump	start ;跳转的开始

```

**对应的ROM文件中的内容为:**

```verilog
`timescale 1ns / 1ps

module rom(
    input          clk,
    input [11:0]address,       //程序取址地址
    output[17:0]instruction,   //指令输入
    input       enable   //程序rom使能 1-->使能 
);
reg [17:0] y;
assign instruction = y;
always@(posedge clk) begin
    if(enable)begin
        case(address)
            12'd0 : y <= 18'h01F02;
            12'd1 : y <= 18'h2DF01;
            12'd2 : y <= 18'h01F01;
            12'd3 : y <= 18'h2DF01;
            12'd4 : y <= 18'h22000;
        endcase
    end
end



endmodule 
```

**仿真文件：**

```verilog
`timescale 1ns / 1ps
module tb ;
reg clk,rst_n;

//生成始时钟
parameter NCLK = 20;
initial begin
	clk=0;
	forever clk=#(NCLK/2) ~clk; 
end 

/****************** BEGIN ADD module inst ******************/
wire [11:0]address;
wire [17:0] instruction;
wire bram_enable, read_strobe, write_strobe;
reg [7:0] in_port;
wire [7:0] port_id, out_port;

mcu mcu(
    .clk(clk),            //系统时钟
    .rst_n(	rst_n),          //复位  0 --> 复位
    .address(	address),       //程序取址地址
    .instruction(	instruction),   //指令输入
    .bram_enable(	bram_enable),   //程序rom使能 1-->使能    
    .in_port(	in_port),        //输入口
    .read_strobe(	read_strobe),    //输入口使能     
    .port_id( port_id),        //io口地址
    .out_port(	out_port),       //输出口
    .write_strobe(	write_strobe)   //输出口写使能  
);

rom rom(
    .clk(	clk),
    .address(	address),       //程序取址地址
    .instruction( 	instruction),   //指令输入
    .enable( 	bram_enable)   //程序rom使能 1-->使能 
);
/****************** BEGIN END module inst ******************/

initial begin
    $dumpfile("wave.lxt2");
    $dumpvars(0, tb);   //dumpvars(深度, 实例化模块1，实例化模块2，.....)
end

initial begin
   	rst_n = 1;
    	#(NCLK) rst_n=0;
    	#(NCLK) rst_n=1; //复位信号

	repeat(10000) @(posedge clk)begin

	end
	$display("运行结束！");
	$dumpflush;
	$finish;
	$stop;	
end 
endmodule

```

**仿真波形**

![image-20201126003842859](/images/image-20201126003842859.png)

可以看到每个指令需要两个周期的时钟，结果符合的还不错。



