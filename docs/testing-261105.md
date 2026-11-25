
Power

5V输入 ORing works

+35V 34.84
+33V 33.31
-33V -33.31
-35V -34.83

3V3     3.28
3.3VA  3.298
3V3A 3.298
VBIAS 1.648


25MHz clock OK
full chip erase OK

migration

从ttf-ng项目迁移；

1. 在cube/mx目录下创建ttf-compact文件夹，把ttf-ng.ioc文件复制到文件夹内，重命名位ttf-compact.ioc，用cubemx打开；
2. 粗略检查
	1. Project Manager内的Toolchain/IDE是STM32CubeIDE
	2. Clock Configuration中25MHz时钟激活，选择HSE，Enable CSS/PLLCLK，SYSCLK是144MHz；
3. Generate Code
4. 打开STM32CubeIDE，选择cube目录的workspace；
5. File->STM32 Project Create/Import，对话框中选择Import STM32 Project分类下的STM32CubeMX/STM32CubeIDE Project，点击Next
6. 在Import Source中选择cube\mx\ttf-compact目录，点击finish；

## 第一次build

错误1：`profile_v2_t` undeclared
原因：在CubeMX的FREERTOS里配置了profileTask, uxTask, uartTask, requestQueue，messageQueue，其中requestQueue类型是profile_v2_t；相关文件均不在项目内；
解决：
复制h文件，edge_detect.h, embedded_cli.h, profile.h, hv2801.h (excluded from build)；但未覆盖main.h；
复制c文件，command.c，edge_detect.c，embedded_cli.c，profile.c，uart.c，hv2801.c (excluded from build)；但未覆盖main.c；

需diff main.h和main.c；done，code migrated. build OK

## 核对硬件修改

- 去掉uart6_tx (PC6, Pin 37)
- 增加SW999 (PA8, Pin 41)
- 增加MUX DIN, CLR, LEN, CLK，全部设置为output, high, medium

## 软件修复去掉uart6_tx的问题

### 分析
1. 所有代码都在uart.c文件中，在main.h有个UART_Send函数的声明，程序的其它部分在使用这个声明；
2. 涉及freertos资源包括StartUartTask，messageQueueHandle；

### 修改
1. delete uartTask；
2. delete messageQueue；
3. 注释掉uart.c内所有代码，保留UART_Send为空函数；

UART_Send函数在command.c有调用，隶属ff命令；在profile.c中有调用，隶属profile任务主循环，即收到请求后向b板发送开关状态；

至此，高压开关尚未工作但a板部分可以开始下载测试了。

移植开关代码

从ttf-ng-b项目中复制hv2801.h，hv2801.c文件，
在hv2801.h中添加#include "profile.h"，allpads_v2_t数据结构在该文件中定义，该数据结构在ttf-ng和ttf-ng-b中一致。
修改CLR, CLK, DIN, LEN定义，加上MUX_前缀，和新项目GPIO名字一致。

至此编译可以通过。

## Gemini Test

Here is a lean, practical set of sanity test cases tailored for a 2-channel oscilloscope to verify your 16-channel differential analog switch board before shipping.
  
### Test 1: Basic Routing & Phase Integrity (A vs. B)

- **Objective:** Verify that selecting Signal A or Signal B routes the correct differential phase to the output without undue attenuation or distortion.
- **Setup:**
    - Drive differential inputs A and B with a clean, low-frequency sine wave (e.g., 1 kHz, 1 Vpp).
    - Connect **Scope Ch 1** to Input A+ (reference).
    - Connect **Scope Ch 2** to Channel 1 Output (+ terminal).
- **Execution:**
	1. Program Channel 1 to **Route A**. Scope Ch 2 should show an exact match in frequency and amplitude to Ch 1.
    2. Program Channel 1 to **Route B**. Scope Ch 2 should show a **180-degree phase shift** relative to Ch 1 (confirming the differential inversion).

Routing测试，CH1检测A，CH2检测输出通道；

### Test 2: High-Z / Float State Verification

- **Objective:** Ensure that the "None" (floating) state truly disconnects the output and doesn't leave a low-impedance residual path.
- **Setup:**
    - Keep inputs A and B driven at normal operating voltage.
    - Connect **Scope Ch 1** to Input A.
    - Connect **Scope Ch 2** to Channel 1 Output.
- **Execution:**
    1. Program Channel 1 to **None (Float)**.
    2. Observe Scope Ch 2: The signal should drop to baseline noise or high-impedance pickup. If you terminate the scope channel with 50$\Omega$ or 1M$\Omega$, verify there is no significant signal bleed-through from A or B.

Floating测试，CH1检测A，CH2检测输出通道；

### Test 3: Sequential 16-Channel Scan (Control & Continuity Check)

- **Objective:** Quickly verify that all 16 output channels are physically connected, solder joints are intact, and the digital control logic/addressing works across the entire board.
- **Setup:**
    - Route a steady test signal (e.g., 10 kHz sine wave) to Input A.
    - Leave **Scope Ch 1** on Input A (as a trigger reference). 
    - Connect **Scope Ch 2** to a probe or a quick test fixture pin.
- **Execution:**
    1. Run a test firmware routine that automatically steps through **Channels 1 through 16 sequentially** (e.g., 500 ms per channel, routing A).
    2. Sweep your probe across the output headers (or use a simple automated jig). You should see the waveform cleanly appear on each channel sequentially, confirming zero dead channels, stuck bits, or broken traces.

Scan测试，CH1检测A，CH2扫描输出通道；

### Test 4: Basic Off-Isolation / Crosstalk Spot Check

- **Objective:** Ensure that an active channel driving a signal doesn't severely bleed into adjacent floating channels.
- **Setup:**
    - Drive Input A with a higher amplitude signal (e.g., 5 Vpp, 10 kHz).
    - Route Signal A to **Channel 1**.
    - Program **Channel 2** to **None (Float)**.
    - Connect **Scope Ch 2** to Channel 2 Output.
- **Execution:**
    1. Measure the peak-to-peak voltage on Channel 2. It should be heavily attenuated compared to Channel 1 (depending on your switch IC specs, typically >40–60 dB down at 10 kHz), confirming acceptable off-isolation.

隔离度测试，CH1检测A，CH2检测floating通道噪音。




```
Array.from({ length: 4 }, ()=>Array.from({ length: 4 }, ()=>Math.floor(Math.random() * 3)).join('')).join()

define 0 a 2012,2000,1121,1211 3000 50 200000
1202,2222,1212,0010
1110,0102,1110,1021
1021,1122,0100,0122

OLDI0
```
