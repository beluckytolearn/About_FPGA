# 呼吸灯
***
## 设计思路与原理
```
呼吸灯是采用PWM的方式，在固定频率下，通过调整占空比的方式来控制LED的亮度变化
LED的亮度由通过LED的电流量决定
PWM pulse width modulation 脉冲宽度调制
设计在计数器周期下 占空比由0%到100%变化 可递增可递减
```
***
## 设计过程

**新建工程**
板子型号*xc7z020clg400-2*
**设计输入**
sys_clk *系统时钟输入* 16位
sys_rst_n *系统复位输入*16位
led *LED灯输出*
**分析综合**
open elaborated design分析
run synthesis综合
**约束输入**
添加约束文件
时序约束 管脚约束
**设计实现**
run implementation设计
**生产和下载比特流文件**
generate bitstream生成bit流文件
open hardware manager自动连接开发板
program device下载
***
## 代码实现
`breathled2.v`
```
`timescale 1ns / 1ps
//
module breathled2(
    input sys_clk,
    input sys_rst_n,
    output led
    );
    
reg [15:0] priod_cnt;//周期计数器 1khz 1ms 计数50000
reg [15:0] duty_cycle;//占空比数值
reg        inc_dec_flag;//0递增1递减
//通过占空比和计数值大小控制led输出
assign led = (priod_cnt >= duty_cycle)? 1'b1:1'b0;
//周期计数 每次加1 至50000 产生led驱动信号
always @(posedge sys_clk or negedge sys_rst_n)begin
    if(!sys_rst_n)
        priod_cnt <= 1'd0;
    else if(priod_cnt == 16'd50000)
        priod_cnt <= 1'd0;
    else
        priod_cnt <= priod_cnt + 1'b1;        
end
//每次递增或递减 控制占空比 每次计数完改变占空比完成控制呼吸灯变化  
always @(posedge sys_clk or negedge sys_rst_n)begin
    if(!sys_rst_n)begin
        duty_cycle <= 16'd0;
        inc_dec_flag <= 16'd0;
    end
    else begin//判断计数值是否到达50000 是否递增 占空比是否递增到50000
        if(priod_cnt == 16'd50000)begin//计数满1ms
            if(inc_dec_flag == 1'b0)begin//占空比递增
                if(duty_cycle == 16'd50000)//占空比最大
                    inc_dec_flag <= 1'b1;//占空比递减
                else
                    duty_cycle <= duty_cycle + 16'd50;
            end
            else begin
                if(duty_cycle == 16'd0)//占空比递减
                    inc_dec_flag <= 1'b0;//占空比递增
                else 
                    duty_cycle <= duty_cycle - 16'd50;
            end
        end
    end
end
endmodule
```
`breathled2.xdc`
```
set_property -dict {PACKAGE_PIN U18 IOSTANDARD LVCMOS33} [get_ports sys_clk]
set_property -dict {PACKAGE_PIN J15 IOSTANDARD LVCMOS33} [get_ports sys_rst_n]
set_property -dict {PACKAGE_PIN J16 IOSTANDARD LVCMOS33} [get_ports led]
```
***
## 实验验证
```
连接开发板 上电下载bit流文件 正常控制LED灯显示呼吸灯效果
```
***
## 存在问题与解决方案
```
JTAG接口连接不稳定 USB接口松动问题：更换接口或者数据线重新连接
约束文件注意约束电平和管脚匹配：查表
```
***
