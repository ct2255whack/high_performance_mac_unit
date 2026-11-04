# A7-Lite-35T 16位MAC实体演示

适用：微相科技A7-Lite-35T，Vivado 2026.1，Part xc7a35tfgg484-2。
A/B=16位；C/Y=33位；Y=(A×B+C) mod 2^33。

## 1 演示内容与LED含义

FPGA自行产生测试数据，运行两阶段Wallace-style MAC并与离线生成的参考值比较。
64组不同向量包含8组定向边界和56组固定种子随机数；重复16遍，共1024笔事务。每8拍插入1个气泡。
每轮结束约1秒后自动重测。任意一轮错误会锁存failed，直到重新下载或断电。

LED1 D6：约1Hz心跳，说明时钟正在驱动逻辑。
LED2 D5：约1Hz慢闪表示已完成该轮且未检测到错误；约8Hz快闪表示发现结果、valid或超时错误。
运行中的LED2为4Hz，但每轮运行只有几十微秒，肉眼通常看不到。下载后等待2秒再判断。
闪烁频率不受LED有效电平相反的影响，只会改变亮灭相位。

这是64组不同向量重复16遍的实体功能自检，不等于穷举证明，也不是Fmax测试。

## 2 引脚

clk_50m：J19，板载50MHz晶振，20ns，LVCMOS33。
led_heartbeat：M18，LED1 D6，LVCMOS33。
led_status：N18，LED2 D5，LVCMOS33。

位置来自微相官方手册。XDC按标准3.3V接法编写，使用前与随板原理图/XDC及板卡版本核对。
本演示无需杜邦线或外部复位按键，使用配置后的内部计数器复位，不猜测K3引脚。
JTAG是配置接口，不在XDC中另行分配TCK/TMS/TDI/TDO。

## 3 建立独立上板工程

解压到英文目录，例如 D:/FPGA/a7_lite_16bit_board_demo/。
方法一：双击open_demo.bat，使用你的安装路径，如：
D:\vivado\2026.1\Vivado\bin\vivado.bat

方法二：在Vivado底部Tcl Console输入：
source D:/FPGA/a7_lite_16bit_board_demo/create_board_project.tcl

当前打开旧工程时，按Vivado提示关闭旧工程后再建新工程；原文件保留。
脚本新建build/a7_mac_demo/a7_mac_demo.xpr。遇到已有目录会停止，后续直接打开该xpr。
Sources应显示：
Design Sources顶层：a7_mac_demo_top
Simulation Sources顶层：tb_a7_mac_demo
Constraints：a7_lite_35t.xdc

四个RTL文件为a7_mac_demo_top、demo_vectors、mac_wallace_pipe、csa_reduce。
本包完整独立，不混用旧版同名RTL，不把Testbench设为硬件顶层。

## 4 先跑顶层仿真

Run Simulation → Run Behavioral Simulation → Run All。
测试缩短两轮等待时间，但每轮仍送入1024笔。正常实例检查每笔结果，故障实例翻转一个期望值验证错误检测。
预期最终输出：
PASS board demo: two rounds, 2048 results; injected fault detected

这是预期输出，当前尚无本次工程的Vivado执行日志。遇到$fatal或编译失败先保存第一条报错。
故障注入只在Testbench的bad实例中启用，正常硬件顶层默认INJECT_FAULT=0。
通过后关闭仿真，回到工程。

## 5 综合 实现 生成bitstream

1. Run Synthesis。
2. Run Implementation。
3. 查看Report Timing Summary，检查50MHz约束、setup和hold，并排查严重警告。
4. Generate Bitstream。

常见输出：
build/a7_mac_demo/a7_mac_demo.runs/impl_1/a7_mac_demo_top.bit

仅两个无同步接收端的LED端点设false path，内部MAC寄存器路径仍按20ns检查。
不要把内部MAC路径设false path，也不要降低DRC严重性跳过引脚和电气约束报错。
若UCIO-1/NSTD-1报错，核对顶层是否为a7_mac_demo_top，且三个端口使用本包XDC。
若出现专用时钟布线报错，先核对时钟脚，不先加入CLOCK_DEDICATED_ROUTE绕开约束。

## 6 JTAG临时下载

1. Open Hardware Manager → Open Target → Auto Connect。
2. 选xc7a35t_0或类似名称。这里一般只显示器件，不显示完整封装速度后缀。
3. 右键 → Program Device。
4. Bitstream file选择上一步生成的a7_mac_demo_top.bit。
5. Debug probes file留空。
6. 点击Program，成功后观察两颗用户LED至少2秒。

此流程下载到FPGA配置SRAM，断电后重新Program。首次演示暂不写SPI Flash，不用Add Configuration Memory Device。

## 7 自检向量与判据

重点向量：
A=5 B=7 C=3 → Y=38
A=65535 B=65535 C=0 → Y=4294836225
A=65535 B=65535 C=8589934591 → Y=4294836224
A=1 B=1 C=8589934591 → Y=0
A=61385 B=53504 C=1354367992 → Y=4638711032

有效输出时Y应等于exp_y2。输入输出按valid对齐，不直接拿同一时刻A/B/C和Y相乘相加比较。
一轮结束后tx_count=1024、rx_count=1024、done=1、failed=0表示该轮通过；下一轮计数复位。
本演示报告含ROM、计数器、检查器等，不直接用于比较裸MAC资源。

## 8 两分钟实物展示

展示板卡及JTAG连接 → Program成功与两颗LED慢闪
→ 展示一轮发送/接收1024、done=1、failed=0 → 说明资源与Fmax另外按统一条件的实现报告比较。

保存实物视频、下载成功截图、资源与时序报告及源码版本。

## 9 验证状态

已核对64组ROM常量与独立整数参考；独立Python周期模型的正常及故障注入两种情况符合判据。
模型每轮从RUN开始到读取最后输出为1172拍，包含气泡及检查器读取，不是MAC本体延迟。
HDL编译、xsim、综合实现和实体结果尚待在Vivado 2026.1执行。
本包未含预生成bitstream。预期PASS行与模型结果不充当HDL仿真通过证明。

## 10 资料

微相官方A7-Lite手册：
https://fpga-docs.microphase.cn/zh-cn/latest/DEV_BOARD/A7-LITE/A7-Lite_Reference_Manual.html
AMD Vivado Programming and Debugging UG908，以2026.1已安装帮助为准。
AMD下载流程示例：
https://www.amd.com/content/dam/amd/en/documents/university/vivado-teaching/hdl-design/2015x/VHDL/docs-pdf/Vivado_Tutorial.pdf

