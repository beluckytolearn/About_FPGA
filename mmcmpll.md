# MMCM/PLL  
***
## 设计思路与原理  
```
PLL phase locked loop 锁相环 反馈控制电路 
利用外部输入的参考信号控制环内震荡信号的频率和相位 常用于闭环跟踪电路
PLL对时钟网络进行系统的时钟管理和偏移控制，具有时钟倍频、分频、相位偏移和可编程占空比的功能
锁相环在工作的过程中，当输出信号的频率和输入信号的频率相等时，输出电压和输入电压保持固定的相位差值，即相位锁住
Xilinx7系列器件中的时钟资源包括时钟管理单元CMT
每个CMT由一个MMCM和一个PLL组成
xc7z020芯片内部与4个CMT 为设备提供强大的系统时钟管理和高速IO通信功能
如CMT总体框图 
MMCM/PLL的参考时钟输入可以使IBUFG（CC）及具有时钟功能的IO输入、区域时钟BUFR、全局时钟BUFG、GT收发器输出时钟、行时钟BUFH以及本地布线
MMCM/PLL的输出可以驱动全局时钟BUFG和行时钟BUFH等
（BUFG能驱动整个器件内部的PL侧通用逻辑的所有时序单元的时钟端口）
MMCM的功能是PLL的超集，其具有比PLL更强大的相移功能
MMCM主要用于驱动器件逻辑（CLB、DSP、RAM）的时钟
PLL主要用于为内存接口生成所需要的时钟信号，也具备与器件逻辑的连接 需要额外的时钟资源
PLL由以下几个部分组成：前置分频计数器（D计数器）、相位-频率检测器（PFD phase frequency detector）、电荷泵（charge pump）、环路滤波器（LOOP Filter）、压控震荡器（VCO voltage controlled oscillator）、反馈乘法器计数器（M计数器）、后置计数器（O1-O6计数器）
在工作时，PFD检测其参考频率FREF和反馈信号feedback之间的相位差和频率差，控制电荷泵和环路滤波器将相位差转换为控制电压；VCO根据不同的控制电压产生不同的震荡频率，从而影响feedback信号的相位和频率。在FREF和feedback信号具有相同的相位和频率之后，就认为PLL处于锁相状态
在反馈路径中插入M计数器会使VCO的震荡频率是FREF信号的M倍 FVCO=FIN*M/D
FREF信号等于输入时钟FIN除以预缩放计数器D FREF=FIN/D
PLL的输出频率为FOUT=（FIN*M）/（N*O）
Xilinx提供用于实现时钟功能的IP核clocking wizard
该IP核能够根据用户的时钟需求自动配置器件内部的CMT及时钟资源，实现用户的时钟需求
```  
***    
## 设计过程  
**新建工程**  
板子型号*xc7z020clg400-2*  
**设计输入输出**  
sys_clk *系统时钟输入* 16位  
sys_rst_n *系统复位输入*16位  
clk_100m 100mhz时钟频率  
clk_100m 100mhz时钟频率相位偏移180度  
clk_50m 50mhz时钟频率  
clk_25m 25mhz时钟频率  
**设计IP核**  
IP Catalog窗口  clocking wizard   
Customize IP窗口  
Clocking Optitons  
设置相关参数 设置开发板晶振频率  
Output clock  
设置输出时钟频率及相位偏移  
其他查看设置是否正确 分频是否合理  
Generate综合生成  
**设计verilog源文件**   
修改例化程序  
**设计testbeach程序**  
修改例化程序  
**运行仿真**  
查看信号输出情况  

***    
## 代码实现  
`IP_CLK_WIZ.v`  
```
//源文件
module IP_CLK_WIZ(
    input sys_clk,//系统时钟
    input sys_rst_n,//系统复位低电平有效
    output clk_100m,//100mhz时钟频率
    output clk_100m_180deg,//100mhz时钟频率 相位偏移180度
    output clk_50m,//50mhz时钟频率
    output clk_25m//25mhz时钟频率
    );
//MMCM/PLL IP核例化程序
wire locked;
clk_wiz_0 clk_wiz_0
(
    // Clock out ports
    .clk_out1(clk_100m),     // output clk_out1
    .clk_out2(clk_100m_180deg),     // output clk_out2
    .clk_out3(clk_50m),     // output clk_out3
    .clk_out4(clk_25m),     // output clk_out4
    // Status and control signals
    .reset(~sys_rst_n), // input reset
    .locked(locked),       // output locked
   // Clock in ports
    .clk_in1(sys_clk)
);  
endmodule
```  
`ip_clk_wiztest.v `  
```
`timescale 1ns / 1ps
module ip_clk_wiztest();
//输入
reg sys_clk;//系统时钟
reg sys_rst_n;//系统复位低电平有效
//输出
wire clk_100m;//100mhz时钟频率
wire clk_100m_180deg;//100mhz时钟频率 相位偏移180度
wire clk_50m;//50mhz时钟频率
wire clk_25m;//25mhz时钟频率
//生产时钟
always #10 sys_clk = ~sys_clk;//延迟10个单位时间 改变系统时钟电平
//信号初始化
initial begin//初始赋值 200个单位时间后开启复位
    sys_clk = 1'b0;
    sys_rst_n = 1'b0;
    #200
    sys_rst_n = 1'b1;
end
//例化设计
IP_CLK_WIZ u_IP_CLK_WIZ(
    .sys_clk (sys_clk),
    .sys_rst_n (sys_rst_n),
    .clk_100m (clk_100m),
    .clk_100m_180deg (clk_100m_180deg),
    .clk_50m (clk_50m),
    .clk_25m (clk_25m)
);
endmodule
```  
***  
## 实验验证  
```
仿真界面出现输出时钟信号 
添加locked信号进行观察 locked信号置1后输出稳定
时钟频率的IP核设置情况与仿真情况一致
```
***  
## 存在问题与解决方法
```  
例化程序中英文符号导致错误：仔细检查  
例化程序系统复位是否需要取反：根据逻辑进行判断 也可先进行仿真后修改
```  
***  
