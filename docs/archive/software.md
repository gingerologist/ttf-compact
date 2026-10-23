# Software

see ttf-core.pdf and designnote.md

# test cases
## 1 Power, Pull-ups

- [x] 5V
- [x] 3.3V
- [x] -5V
- [x] -9V
- [x] NRST

## Connect to Programmer / Debugger

stlink v2 pinout

Notch Left

```
1 VTref [3v3]
                    4 GND [x]
                    6 GND
7 SWDIO             8 GND
9 SWCLK             10 GND
...
15 RESET
-
-
```

- [x] connect via stm32cube programmer

## Legacy

- `commands.c` custom cli commands
- `embedded_cli.c` third party libraries
- `main.c`
- `profile.c` for profile function, including persistent storage in flash.

There may be other things to remember.

## Configuration ioc from Schematics

- There is no pull-up/pull-down for sw1-5 pin, open or close to GND. All 5 pins need pull-up and detect ground.
- LED need to be pull-down on switch board. There is a digital transistor on core board. So output high to enable led and low vice versa.
- Freertos use CMSIS v1, for Multipads use v1. May be change to v2 in future.
- CRC also initialized.

## Modules

- led
- usart
- pwm output @ fixed frequency
- read adc
- read gpio (no need to be fast)
- write digipots
- write switch

## Framework (TODO)


## Tasks

### LED

- [x] blink

### USART

- [x] check how multipads works `HAL_UART_Transmit`
- [x] print hello `\r\n`

### TIM1 (PWM)

- [x] Configure PWM `exact 200KHz measured`
    - [x] https://01001000.xyz/2020-10-24-Tutorial-STM32CubeIDE-Timers-PWM-AM-Radio/
    - [x] change pll to /25 x256 /4 to make APB2 timer clock `64MHz`
    - [x] prescaler 0 `x1`
    - [x] counter period `319` (320 / 64 = 5us)
    - [x] pulse `159` (160 / 64 = 2.5us)
- [x] `HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1)`

### ADC

- [ ] 













