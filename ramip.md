# RAMIP  
***   
##  设计思路与原理
```
Xilinx7系列器件具有嵌入式存储器结构
由一系列BRAM存储器模块组成
通过对BRAM存储器模块进行 配置，可以实现各种存储器的功能
Vivado软件自带BMG P核 block memory generator 块RAM生成器 可以生成RAM ROM
配置成RAM或ROM使用的资源的批示FPGA内部的BRAM 只不过配置成ROM只使用到嵌入式BRAM的读数据端口
Xilinx7系列器件内部的BRAM全部都是真双端口RAM TDP ture dual port ram
这两个端口都可以独立地对BRAM进行读写
但也可以配置成伪双端口 SDP simple dual port ram 一只读一只写
或者单端口RAM 读写只在一端口进行
单端口RAM只有一组数据总线、地址总线、时钟信号以及其他控制信号
双端口RAM具有两组数据总线、地址总线、时钟信号以及其他控制信号
如单端口RAM器件框图
单端口RAM器件各个端口描述：
DINA写数据信号
ADDRA读写地址信号
WEA写使能信号 高写入 低读出
ENA使能信号 高使能 可选信号
RSTA复位信号 可选信号
REGCEA输出寄存器使能信号 高保持最后一次输出数据 可选信号
CLKA时钟信号
DOUTA读出数据
```
***   
## 设计过程  
**新建工程**  
板子型号*xc7z020clg400-2*  
**设计输入输出**  
sys_clk *系统时钟输入* 16位  
sys_rst_n *系统复位输入*16位  
**设计IP核**  
IP Catalog->block memory generator  
basic->interface:native（标准接口）  
basic->memory type:single port ram（单端口RAM）  
basic->ecc options:error correction capability（单端口不支持ECC）  
basic->wirte enable:不使能  
basic->algorithm options:minimum area（最小面积）  
port a options->write width:8  
port a options->read width:8  
port a options->write depth:32  
port a options->read depth:32  
port a options->operating mode:no change（分开读写）  
port a options->enable type:use ena pin（添加使能端口A信号）  
port a options->port optional output register:不使能  
其他不设置  
生成IP核  
**设计verilog源文件**   
创建ram_rw.v底层文件  
创建ip_ram.v顶层文件  
修改例化程序  
**设计testbeach程序**  
修改例化程序  
**运行仿真** 
查看信号输出情况  
***  
## 代码实现  
`ramrw.v`
```
module ramrw(
    input sys_clk,//系统时钟信号
    input sys_rst_n,//复位信号
    output ram_en,//使能信号
    output ram_wea,//读写选择 1写数据 0读数据
    output reg [4:0] ram_addr,//读写地址
    output reg [7:0] ram_wr_data,//写数据
    output reg [7:0] ram_rd_data//读数据
    );   
reg [5:0] rw_cnt;//读写控制计数器
//控制RAM使能信号
assign ram_en = sys_rst_n;
//rw_cnt计数范围在0-31时写数据 32-63时读数据
assign ram_wea = (rw_cnt <= 6'd31 && ram_en == 1'b1 ) ? 1'b1 : 1'b0;
//读写控制计数器
always @(posedge sys_clk or negedge sys_rst_n) begin
    if(sys_rst_n == 1'b0)
        rw_cnt <= 1'b0;
    else if(rw_cnt == 6'd31)
        rw_cnt <= 1'b0;
    else
        rw_cnt <= rw_cnt +1'b1;
end
//写数据 [7:0] ram_wr_data
always @(posedge sys_clk or negedge sys_rst_n) begin
    if(sys_rst_n == 1'b0)
        ram_wr_data <= 1'b0;
    else if(rw_cnt <= 6'd31)
        ram_wr_data <= ram_wr_data +1'b1;//写数据累加
    else 
        ram_wr_data <= 1'b0;
end
//读写地址 [4:0] ram_addr
always @(posedge sys_clk or negedge sys_rst_n) begin
    if(sys_rst_n == 1'b0)
       ram_addr <= 1'b0;
    else if(ram_addr == 5'd31)
       ram_addr <= 1'b0;
    else
       ram_addr <= ram_addr + 1'b1;//地址递增
end 
endmodule
```
`ipram`
```
module ipram(
    input sys_clk,//系统时钟
    input sys_rst_n//系统复位 低有效
    );    
wire ram_en;//使能
wire ram_wea;//读写使能 高写入 低读出
wire [4:0] ram_addr;//读写地址
wire [7:0] ram_wr_data;//写数据
wire [7:0] ram_rd_data;//读数据

//读写模块 例化程序
ramrw ramrw(
    .sys_clk(sys_clk),//系统时钟信号
    .sys_rst_n(sys_rst_n),//复位信号
    .ram_en(ram_en),//使能信号
    .ram_wea(ram_wea),//读写选择 1写数据 0读数据
    .ram_addr(ram_addr),//读写地址
    .ram_wr_data(ram_wr_data),//写数据
    .ram_rd_data(ram_rd_data)//读数据
 );  
//ram ip核例化程序
blk_mem_gen_0 blk_men_gen_0 (
  .clka(sys_clk),    // input wire clka
  .ena(ram_en),      // input wire ena
  .wea(ram_wea),      // input wire [0 : 0] wea
  .addra(ram_addr),  // input wire [4 : 0] addra
  .dina(ram_wr_data),    // input wire [7 : 0] dina
  .douta(ram_rd_data)  // output wire [7 : 0] douta
);
endmodule
```
`tbipram`
```
`timescale 1ns / 1ps
module tbipram();
reg sys_clk;
reg sys_rst_n;
always #10 sys_clk = ~sys_clk;

initial begin
    sys_clk =1'b0;
    sys_rst_n = 1'b0;
    #200
    sys_rst_n = 1'b1;
end

ipram u_ipram(
    .sys_clk(sys_clk),//系统时钟
    .sys_rst_n(sys_rst_n)//系统复位 低有效
    );  
endmodule
```
***  
## 实验验证  
```
在simulation里可以观察到RAM读写数据波形
```
***
## 存在问题与解决方法  
```
没有很好理解RAM功能:多加强学习
```
***  



