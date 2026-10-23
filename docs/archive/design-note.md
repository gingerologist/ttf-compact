---
title: TTF-ng V3 Hardware Design
author: ma (matianfu@gingerologist.com)
date: 2025-11-30
version: 1.0
---

[TOC]



# 背景

本文档是TTF-ng V3的完整硬件设计说明。

TTF-ng是之前的TTF项目的演进。`-ng`是一些软件项目的命名习惯，意为next generation。

启动TTF-ng的目的是为了替换光耦可控硅器件，该器件并非为高频信号开关设计，在频率响应和响应速度上都没有保障，而且因为实际使用的路数很多，占布板面积很大。拟更换为Microchip专门为超声高电压范围准备的器件，HV2801。

Microchip的HV2801和HV2901都是电压可达200V范围的2:1（AB-Y配置）高压模拟开关。在超声应用里，AB两路一般一路用于高压激励波的发射通路，另一路是回波进放大器的，是接收通道，Y接在换能器上；使用开关的目的是避免超声发射波直接进入接收器，这会损毁器件。大部分超声成像应用使用的超声波频率在1-10MHz范围，高于本项目所需的50-500KHz范围，这就保证了开关速度，线性度，损耗，寄生电容等要求均可满足本项目需要。

在本项目里，Y接在电极片上，AB可分别接100V Vpp和GND，也可以接推挽输出的两路。每个电极片以选择接A，接B，或都不接。控制上，HV2801的AY和BY是各自独立控制的。

> 注意：HV2901是带有bleeding resistor的版本，这个电阻是用于超声激励波发出后，回波接受之前，对地释放线路电容导致的残留电压。在我们的应用方式里不能选择这种有bleeding resistor的版本，只能使用HV2801，否则这个放电电阻会成为负载，而高压信号输出无论使用何种方式，均不可能在连续输出时承载这么低的输出负载电阻。这不同于超声应用里，激励通常只有几个周期，持续时间进1微秒左右的使用场景。

TTF-ng的v1版本在硬件上遇到了两个错误，一个是`-70V`电源的输入选错了电源的耐压值，另一个是HV2801的Exposed Pad，应该接VNN，即最低电压（-70V），而不是GND。这个芯片的GND主要是给控制电路使用的，SPI信号参考GND。

TTF-ng的v2版本修复了v1版本的错误，加入了电压跟随器以试图提供更高的高压信号驱动能力，加入输出电压的反馈电路。实际测试结果不理想，跟随器在高电压和高频率时无法失真较为严重。变压器的高压输出和之前测试结果类似，只能在200KHz附近的频率下能较好的工作，无法达到50-500KHz变频使用的目标，更换高压开关未能有显著改善。但是电压反馈电路可以正常工作，并使用了一个简单的算法实现RMS电压计算，从RMS电压换算成VPP，在正弦波基本无失真时准确性还是不错的，可以在后续版本沿用。

基于v2版本的测试结果和参考了Analog，TI最新的高速高压放大器产品的数据手册后，在v3版本决定了以下两点重大设计变更：

1. 改用差分信号；即AB两路不是信号和地，而是两路相位差180度的信号；这样对高压放大器的Slew Rate要求可以减半；可以找到可用的放大器；
2. 使用两级放大器实现正弦波信号发生器AD9834的两级放大；



重点设计挑战：

1. 放大器器件的选择；
2. 电源设计；



AD9834本身是两路位相相差180度的差分信号输出，幅度仅0.7V，考虑到工作频率要求，没有工作电压范围、增益带宽积和Slew Rate均满足要求的放大器可以一级放大完成目标，只能使用两级放大。第一级放大器把信号放大至30V左右的摆幅，选36V内的高速放大器，第二级放大到50V，选专门高压放大的高Slew Rate放大器。



电源方面的设计也有多项挑战。

1. HV2801有10V的headroom，比如信号摆幅`-25V`至`+25V`，HV2801需要供电范围应至少`-35V`至`+35V`，以保障信号经过模拟开关时的失真尽可能低。
2. 考虑到模拟放大器应该有高PSRR的LDO提供更好的电源质量，但需要的电压范围超过了大部分LDO器件的输入电压范围，尤其是负电压范围无器件可用；需要一种称为Virtual Ground的办法把LDO的参考地“垫高”，相当于拓展其输出电压范围。
3. 一级放大器需要至少`+/-15V`左右的电压，二级放大器需要至少`+/-26V`左右的电压，高压模拟开关需要`+/-36V`左右的电压；电压轨较多；



实际设计电源和放大器均需要软件仿真以确定最后设计。设计过程中充分调研过的参考设计和芯片选型，未能在设计中采用的，记录在文档的附录中。



# 系统设计

系统包含如下组成部分：

1. 输入电源，5V USB Type C
   1. 输入电源1，5V USB Type C，只供电，与输入电源2跳线二选一
   2. 输入电源2，5V USB Type C，和串口芯片同一路
   3. 5V到3.3V数字电源，mcu，ad9834
   4. 5V到3.3V模拟电源，mcu，ad9834模拟1，与模拟2二选一
   5. 5V到3.3V电压参考，mcu，ad9834模拟2
   6. +50V电源
   7. -50V电源
   8. +18V LDO，一级放大
   9. -18V LDO，一级放大
   10. +36V LDO，二级放大，虚地
   11. -36V LDO，二级放大，虚地
2. 信号发生器芯片，AD9834
   1. 输入时钟二选一
      1. MCU
      2. 专用数字时钟（25MHz）
3. 一级放大器 x2
4. 二级放大器 x2
5. 高压模拟开关
6. 输出电压检测
7. 接口
8. 人机
9. MCU



未在此版本纳入系统的设计考量

1. 隔离电源
2. 隔离ADC









- MCU
  - STM32F405RGT6
  - BOOT0 and BOOT1 with TP
  - NRST with TP
  - Debug port with 3V3, GND, NRST, SWDCLK, SWDIO
  - 25MHz and 32.768kHz xtal
  - i2c1, with 10k pull-up (there are 4.7k pull up on switch board)
  - timer channel x1 for PWM output
  - adc1 for peak voltage (attenuated)
  - spi1 for digipots
  - usart2 for usb-serial port

- 




# 电源设计

## +50V DC-DC

沿用了之前验证过的设计，使用TI TPS61391 Boost芯片。

使用TI Webench设计，5V输入，50V输出，最大40mA电流。设计文件：TPS61391RTER.pdf

从5V Boost到50V的升压芯片不常见；也可以考虑使用最大升压可达28-36V的Boost芯片加上电荷泵升压，但是反馈环需要仔细设计。如无供货问题还是尽量选择TPS61391或TPS61390，应对瞬态负载变化的能力会远好于电荷泵方案。TPS61391和TPS61390的主要差别是后者可以输出电压低至2.5V，前者至少20V，但是在本应用中没有差别。

另外TI有基于TPS6139x和电流镜的-90V电压输出的参考设计。如果布板面积、减少高压电容、减少物料种类是设计考虑，也可以用TPS6139x实现-50V输出。

## +18V和+36V LDO

一级放大和二级放大均需要质量较高的电源，尤其是一级放大。因此需要从50V到18V和到36V均使用high PSRR的LDO。器件选型





## -50V DCDC

沿用了之前验证过的设计，使用LMR38010SDDA芯片。该芯片是输入最高80V输出最高75V的Buck，使用inverting buck-boost配置，即芯片参考地作为负电压输出，电压输出接地。该方式会导致输入输出都要使用高达100V的电容。但该配置的瞬态反应能力还是显著优于多级电荷泵。

使用TI Webench设计



实现负电压的另一个好选择是使用SEPIC拓扑，TPS533440，LM5157/LM51571，LM5158/LM51581都曾经考虑过，但是在电压反馈上会麻烦一点，需要电流镜或运放把负电压转换到正电压范围。

除了上面说的需要高压电容之外，LMR38010还有两个目前可接受的缺点。一个是它使用COT（constant on time）模式设计，没有外置的反馈控制网络，因此会使用相对保守的控制方式，即瞬态反应不会特别快；二是使用5V电压输入时，设计参数成为从55V降压到50V，压差很小，需要特别大的电感。





## -18V和-36V LDO



放大器倍率

一级放大44倍，non-inverting，43K vs 1K。0.7V可放大至30.8V Vpp。

二级放大2倍，non-inverting，15K vs 15K。放大至61.6V Vpp。



ADC分压

+/-33V map到0-3.3V

```
3v3a			+/-33V
 |               |
 |               RH
 |				 |
 -----RB-------------- DIV
                 |
                 RL
                 |
                ---
```

如果RH top end电压为-33V，最小，此时DIV应该为0，RL无电流，即RB和RH分压+3.3到-33V得到0，即

```
RB/(RB + RH) = 3.3V / (3.3V + 33V)
=> (RB + RH) / RB = (3.3V + 33V) /3.3V
=> 1 + RH/RB = 11
=> RH/RB = 10
```

如果RH top end电压为33V，最大，此时DIV应该为3.3V，RB无电流，即RL和RH分压0-33V得到3.3V，即

```
RL/(RL + RH) = 3.3V / 33V
=> 1 + RH/RL = 10
=> RH/RL = 9
```

所以：

```
RH = 10 * RB = 9 * RL
=> RL/RB = 10/9 ~= (RL=22k)/(RB=20k)

RH = 10 * RB = 200k

或者

RL=22K|22K
RB=10K
RH=100K
```











# 硬件设计检查清单

## STM32

> 参考文献
>
> 1. AN4488 from ST
> 2. DS8626，datasheet

### Digital Power Supply
- [x] GND pin 18 & 63.
- [x] VDD pin 19, 32, 48, 64
  - [x] 2.2u + 0.1u for each
- [x] VBAT pin 1 (cap shared allowed)
- [x] VCAP, pin 31, 47
  - [x] 2.2u for each

### Analog Power Supply
- [x] VSSA pin 12
  - [x] well grounded
- [x] VDDA pin 13
  - [x] 1u + 0.1u

### Reset
- [x] external pull-up (10k)
- [x] 0.1u cap
- [x] reset button
- [x] debug interface

### Boot
- [x] boot 0 (原则上应该使用两个电阻串联接地，中点可接3V3)
  - [x] 10k pull down
  - [x] can be pulled up to 3V3
- [x]  boot 1 (原则上应该使用两个电阻串联接地，中点可接3V3)
  - [x]  10k pull down
  - [x]  can be pulled up to 3V3

### SWD (and JTAG)

- [x] SWDIO (PA13)
  - [x] keep trace short
- [x] SWCLK (PA14)
  - [x] keep trace short
- [x] is JTDI used? no
- [x] is JTDO used? yes, spi clock, but this pin is not pulled up (AN4488, p.30)
- [x] is JNTRST used? no
- [x] is NRST routed to pin header? yes

### Clocks

- [x] LSE used? no, removed
- [x] HSE used? yes. 25MHz XO with CMOS output
  - [x] XO powered with digital supply? yes
  - [x] XO power decoupled? yes
  - [x] XO output dampening resistor for all branches? yes
  - [x] XO well grounded?
  - [x] OSC_IN input
  - [x] OSC_OUT float

### Peripherals

- [ ] ADC1
  - [ ] 22p
- [ ] ADC2
  - [ ] 22p
- [ ] DAC
- [ ] USART1
- [ ] SW1-5, LEDC
- [ ] SPI1 + MUX1 LE_N, CLR
- [ ] SPI2



## 电源

### 3.3V数字电源

- [x] 输入cap，容量，摆件
- [x] 输出cap，容量，摆件
- [x] 输入输出共地距离

### 3.3V模拟电源

- [x] 输入cap，容量，摆件
- [x] 输出cap，容量，摆件（这个不是必须的，REF3033可以不用输出电容）
- [x] 输入输出共地距离
- [x] 输入磁珠（磁珠是必要的，因为LDO和磁珠滤波频率不同）

### +50V电源

- [x] 输入cap，容量，2u，min 1u
- [x] 输入cap，摆件（保证接地即可）
- [x] 输出cap，容量，4 x 100n，min 3 x 100n
- [x] 输出cap，摆件
- [x] 电感容量，4.7uH
- [x] 标注Vfb（1.2V）
- [x] 核算电压（50.1V）
- [x] 输入输出共地距离
- [x] 所有接地过孔和铺铜
- [x] 二极管耐压，100V
- [x] 输出电容耐压，100V

### -50V电源

- [x] 输入cap，容量，3.3u + 100n, = min
- [x] 输入cap，摆件，close to Vin
- [x] 输入cap，耐压，100V
- [x] 输出cap，容量，20u = min
- [x] 输出cap，摆件
- [x] 输出cap，耐压，100V
- [x] 电感容量，68uH
- [x] 标注Vfb，1V
- [x] 核算电压，50.0V
- [x] 输入输出共地（-50V），excellent
- [x] 所有接地过孔和铺铜
- [x] 电感下方挖铺铜

### +/-18V, +/-36V

- [x] 输入cap，容量，10u
- [x] 输入cap，摆件
- [x] 输入cap，耐压，50V
- [x] 输出cap，容量，10u
- [x] 输出cap，摆件
- [x] 输出cap，耐压，50V
- [x] 标注Vfb，TPS7A30
- [x] 标注Vfb，TPS7A49
- [x] 计算电压值，18V
- [x] 输入输出地
- [x] 接地过孔和铺铜
- [x] 电压测试点















