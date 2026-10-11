**原理图来源**：`资料/STM32_F4VE--核心板原理图.PDF`（Altium 导出矢量 PDF，共 3 页：第 1 页为原理图，第 2 页为 PCB 视图，第 3 页为 BOM）。本文依据第 1 页原理图，器件值参考第 3 页 BOM；原理图中的网络名、位号、器件值均可读，无 `UNREADABLE` 项。

# 1、电源架构分析：输入电源路径

| 电源节点/网络 | 来源 | 去向/负载 | 电压/条件 | 控制/保护器件 | 软件相关影响 |
|---|---|---|---|---|---|
| 5V（USB 供电） | J4 Mini USB 的 VDD（J4-1） | 经 R25 接入 5V 网络 | USB 总线 5V | R25 串联（标注 500mA，0805）；无 TVS、无反接保护 | VBUS 不接 MCU 任何引脚，USB 设备驱动需关闭 VBUS 检测 |
| 5V（排针输入） | J2-1、J2-2、J2-3、J2-4（原理图注“预留 5V 输入接口，连接器间距 2.54mm”） | 5V 网络 | 5V | 无保护器件 | 无 |
| 5V（串口座） | J6-1（ISP_COM） | 5V 网络 | 5V | 无保护器件 | USB 转串口模块可直接给板子供电 |
| 5V | 以上三路直接并联，无隔离二极管 | U1 AMS1117-3.3 输入（U1-3）；C5 106、C3 104 | 5V | — | 无 |
| 3V3 | U1 AMS1117-3.3 输出（U1-2）；C2 106、C4 104 | MCU VDD_1~5；W25Q16；TF 卡；NRF 接口 JP2-2；TFT J1-31 与背光 Q2；JTAG P1-1/P1-2；电源灯 D1；各上拉电阻；J2-5~8、J3-1~4 | 3.3V | 线性稳压，无使能脚、无 Power Good 网络 | 没有软件可控的电源开关；单一 3.3V 供电，可按电压调节器 Scale 1、Flash 5 等待周期运行 168 MHz |
| VDDA | 3V3 经 R34 0R | MCU VDD（Pin 19）、VDDA（Pin 22）；C23 104、C24 106 | 3.3V | R34 0R | ADC、DAC 的模拟电源等于 3V3 |
| VREF+ | VDDA 经 R11 0R | MCU VREF+（Pin 21）；引出 J2-22 | 3.3V | R11 0R | ADC 满量程约为 3.3V（以 3V3 实测值为准，可用内部 VREFINT 校正） |
| VREF- | 经 R35 0R 接 GND | MCU VREF-（Pin 20）；引出 J2-21 | 0V | R35 0R | — |
| VBAT | Q1 BAT54C 双二极管：3V3 接 Q1-1，电池 B1 接 Q1-2，公共端 Q1-3 输出 | MCU VBAT（Pin 6）；C7 104 | 两者中较高者减去二极管压降 | Q1 | 装了 B1 时，3V3 掉电后 RTC、LSE、备份寄存器和备份 SRAM 仍保持；RTC 初始化前应先检查备份域是否已配置过 |
| VCAP_1 / VCAP_2 | MCU 内部稳压器 | C21 106、C22 106 接 GND | — | — | 无需软件处理 |
| 电源指示 | 3V3 经 R5 510R 接 D1 到 GND | — | 上电常亮 | — | 不受软件控制 |

# 2、主控芯片分析

| 项目 | 原理图内容 | 软件相关说明 |
|---|---|---|
| 型号与封装/位号 | U2：STM32F407VET6，LQFP100 | 原理图“CPU简介”注释写的是“Cortex-M3”，这是注释错误；F407 实际为 Cortex-M4F（带 FPU），工程应按 M4F 配置并开启 FPU。注释中“RAM 196K”对应 192 KB SRAM（含 64 KB CCM）+ 4 KB 备份 SRAM |
| 可确认的核心资源 | Flash 512 KB，最高 168 MHz；本板实际用到的外设：FSMC（16 位并口 LCD）、SDIO（4 位 TF 卡）、SPI1（W25Q16 与 NRF24L01 共用）、USART1（ISP 串口）、USB OTG FS（设备）、RTC + LSE，以及 GPIO（LED、按键、触摸接口） | 除上述外设外，其余 IO 均只引到排针 J2、J3 |
| 电源引脚与去耦 | VDD_1~5（Pin 50/75/100/28/11）接 3V3，去耦 C12~C17 104 ×6、C18 106；VSS_2~5（Pin 74/99/27/10）接 GND；VDD（Pin 19）与 VDDA（Pin 22）同接 VDDA 网络，去耦 C23 104、C24 106；VCAP_1（Pin 49）接 C21 106，VCAP_2（Pin 73）接 C22 106 | — |
| 时钟电路 | HSE：Y2 8 MHz 晶振，接 OSC_IN（Pin 12）/ OSC_OUT（Pin 13），负载电容 C10、C11 22pF，R10 1MΩ 并联；LSE：Y1 32.768 kHz，接 PC14（Pin 8）/ PC15（Pin 9），负载电容 C8、C9 12pF | `HSE_VALUE=8000000`；168 MHz 配置为 PLLM=8、PLLN=336、PLLP=2；USB 和 SDIO 需要 48 MHz 的 PLL48CLK，必须设置 PLLQ=7（336/7=48）；RTC 时钟源可选 LSE |
| 复位电路 | NRST（Pin 14）网络为 RST：R6 10K 上拉到 3V3，C6 104 到 GND，RST 按键到 GND；同一网络还接 JTAG P1-15（nSRST）和 TFT 接口 J1-2（LCD 复位） | LCD 只能跟随 MCU 一起复位，软件无法单独复位 LCD；上电后应先延时，等 LCD 控制器完成复位再发初始化命令 |
| 启动配置 | BOOT0（Pin 94）：R7 10K 下拉到 GND，经 R23 510R 引到 J3-5，跳帽插 J3-3（3V3）为 1、插 J3-7（GND）为 0。BOOT1/PB2（Pin 37）：R8 10K 下拉到 GND，经 R33 510R 引到 J3-6，跳帽插 J3-4（3V3）为 1、插 J3-8（GND）为 0。启动方式：BOOT0=0 从主 Flash 启动；BOOT0=1、BOOT1=0 进入系统存储器（ISP 下载）；BOOT0=1、BOOT1=1 从内部 SRAM 启动 | 原理图注明默认已通过 R7、R8 接地，跳帽默认插在闪存模式 |
| 下载调试接口 | P1 为 20 针 ARM JTAG：1 Vref、2 Vsupply 接 3V3（C1 104）；3 nTRST = PB4（R4 10K 上拉）；5 TDI = PA15（R3 10K 上拉）；7 TMS/SWDIO = PA13（R2 10K 上拉）；9 TCK/SWCLK = PA14（R9 10K 下拉）；11 RTCK 未连接；13 TDO = PB3（R1 10K 上拉）；15 nSRST = RST；17、19 NC；偶数脚接 GND。J6（ISP_COM）：1 = 5V，2 = GND，3 = PA10/RXD1，4 = PA9/TXD1，用于 USART1 系统 Bootloader 下载 | PB3、PB4 同时是 JTAG 的 TDO、nTRST 和 SPI1 的 SCK、MISO。只要使用 SPI1（W25Q16、NRF），就只能用 SWD（PA13/PA14）调试，4 线 JTAG 和 SWO（PB3）不可用。用 SWD 时 PA15 可作普通 GPIO |

# 3、IO 分配表

“引出 J2-xx / J3-xx”表示该引脚同时引到扩展排针的对应针脚。

| MCU引脚 | 网络名 | 外设功能 | 输入/输出 | 默认状态 | 驱动方式 | 软件配置建议 |
|---|---|---|---|---|---|---|
| PA0 / Pin 23 | PA0/WK_UP | K_UP 按键（另一端经 R16 10K 接 3V3）；WKUP 唤醒脚；引出 J2-23 | 输入 | 松开时浮空，按下为高 | 下拉输入 / 中断输入 | 输入 + 内部下拉；中断用 EXTI0 上升沿；需要待机唤醒时使能 PWR WKUP |
| PA1 / Pin 24 | PA1 | 引出 J2-24 | 未使用 | 浮空 | — | 仅排针引出，按扩展用途配置 |
| PA2 / Pin 25 | PA2 | 引出 J2-25 | 未使用 | 浮空 | — | 同上 |
| PA3 / Pin 26 | PA3 | 引出 J2-26 | 未使用 | 浮空 | — | 同上 |
| PA4 / Pin 29 | PA4 | 引出 J2-27 | 未使用 | 浮空 | — | 同上（可作 DAC_OUT1） |
| PA5 / Pin 30 | PA5 | 引出 J2-28 | 未使用 | 浮空 | — | 同上（可作 DAC_OUT2） |
| PA6 / Pin 31 | PA6 | LED D2（阳极接 3V3，阴极经 R13 510R 接本脚）；引出 J2-29 | 输出 | 浮空，LED 灭 | 推挽输出 | 推挽、无上下拉、低速，初始化输出高（灭）；输出低 = 亮。调光可用 TIM3_CH1（AF2），极性取反 |
| PA7 / Pin 32 | PA7 | LED D3（阴极经 R14 510R）；引出 J2-30 | 输出 | 浮空，LED 灭 | 推挽输出 | 同 PA6；调光可用 TIM3_CH2（AF2） |
| PA8 / Pin 67 | PA8 | 引出 J3-35 | 未使用 | 浮空 | — | 仅排针引出（可作 MCO1 输出） |
| PA9 / Pin 68 | PA9/TXD1 | USART1_TX，接 J6-4；引出 J3-34 | 输出 | 浮空 | 复用推挽 | AF7，高速，无上下拉 |
| PA10 / Pin 69 | PA10/RXD1 | USART1_RX，接 J6-3；引出 J3-33 | 输入 | 未接串口时浮空 | 复用功能 | AF7，开内部上拉，避免悬空时误收数据 |
| PA11 / Pin 70 | PA11/USB_DM | OTG_FS_DM，经 R18 22Ω 接 J4 D-；引出 J3-32 | 双向 | 浮空 | 复用功能 | AF10，最高速，无上下拉 |
| PA12 / Pin 71 | PA12/USB_DP | OTG_FS_DP，经 R19 22Ω 接 J4 D+；D+ 由 R21 1.5K 固定上拉到 3V3；引出 J3-31 | 双向 | 外部上拉为高 | 复用功能 | AF10，最高速，无上下拉 |
| PA13 / Pin 72 | PA13/TMS | JTMS/SWDIO，接 P1-7，R2 10K 上拉 | 调试 | 复位后为 SWDIO（AF0，内部上拉） | 复用功能 | 保持复位默认，不要重新配置 |
| PA14 / Pin 76 | PA14/TCK | JTCK/SWCLK，接 P1-9，R9 10K 下拉 | 调试 | 复位后为 SWCLK（AF0，内部下拉） | 复用功能 | 保持复位默认，不要重新配置 |
| PA15 / Pin 77 | PA15/TDI | JTDI，接 P1-5，R3 10K 上拉；引出 J3-30 | 调试 | 复位后为 JTDI（AF0，上拉），外部上拉为高 | 复用功能 | 用 JTAG 时保持默认；用 SWD 时可改作普通 GPIO（注意有外部上拉） |
| PB0 / Pin 35 | F_CS | W25Q16 片选 CS#（U3-1），R20 10K 上拉到 3V3；引出 J2-33 | 输出 | 外部上拉为高（未选中） | 推挽输出 | 推挽、高速，初始化输出高；输出低 = 选中 |
| PB1 / Pin 36 | LCD_BL | 背光控制：经 R30 1.5K 接 Q2（PNP）基极；引出 J2-34 | 输出 | 浮空，Q2 截止，背光灭（R31 未贴） | 推挽输出 / PWM 复用 | 推挽，初始化输出高（灭），LCD 初始化完成后拉低点亮；调光用 TIM3_CH4（AF2），极性取反 |
| PB2 / Pin 37 | PB2/BOOT1 | BOOT1 启动配置：R8 10K 下拉；经 R33 510R 接 J3-6 跳线 | 电源/时钟/复位/调试 | 外部下拉为低 | 无需 GPIO 配置 | 不作普通 IO 使用，保持复位默认输入 |
| PB3 / Pin 89 | PB3/TDO、SPI1_SCK | SPI1_SCK，接 W25Q16 CLK（经 R22 22Ω）和 JP2-5；同时接 JTAG TDO（P1-13，R1 10K 上拉）；引出 J3-18 | 输出 | 复位后为 JTDO（AF0），外部上拉为高 | 复用推挽 | AF5 SPI1_SCK，高速；配置后 JTAG 和 SWO 不可用，SWD 不受影响 |
| PB4 / Pin 90 | PB4/TRST、SPI1_MISO | SPI1_MISO，接 W25Q16 SO 和 JP2-7；同时接 JTAG nTRST（P1-3，R4 10K 上拉）；未引到 J2/J3 | 输入 | 复位后为 NJTRST（AF0，内部上拉），外部上拉为高 | 复用功能 | AF5 SPI1_MISO，可保留上拉 |
| PB5 / Pin 91 | PB5、SPI1_MOSI | SPI1_MOSI，接 W25Q16 SI 和 JP2-6；引出 J3-17 | 输出 | 浮空 | 复用推挽 | AF5 SPI1_MOSI，高速 |
| PB6 / Pin 92 | NRF_CE | NRF 接口 JP2-3；引出 J3-16 | 输出 | 浮空 | 推挽输出 | 初始化输出低（模块待机） |
| PB7 / Pin 93 | NRF_CS | NRF 接口 JP2-4（CSN）；引出 J3-15 | 输出 | 浮空 | 推挽输出 | 初始化输出高（不选中），必须在任何 SPI1 访问之前完成 |
| PB8 / Pin 95 | NRF_IRQ | NRF 接口 JP2-8；引出 J3-14 | 输入 | UNKNOWN（板上无上下拉，取决于模块） | 中断输入 | 输入 + 内部上拉；EXTI8（EXTI9_5_IRQn），触发边沿见第 5 部分 |
| PB9 / Pin 96 | PB9 | 引出 J3-13 | 未使用 | 浮空 | — | 仅排针引出 |
| PB10 / Pin 47 | PB10 | 引出 J2-44 | 未使用 | 浮空 | — | 仅排针引出 |
| PB11 / Pin 48 | PB11 | 引出 J2-45 | 未使用 | 浮空 | — | 仅排针引出 |
| PB12 / Pin 51 | T_CS | 触摸片选，接 J1-24；引出 J2-46 | 输出 | 浮空 | 推挽输出 | 推挽，初始化为无效电平（有效电平见第 5 部分） |
| PB13 / Pin 52 | T_SCK | 触摸 SPI 时钟，接 J1-23；引出 J2-47 | 输出 | 浮空 | 复用推挽或推挽输出 | 可用硬件 SPI2_SCK（AF5），或用 GPIO 模拟 |
| PB14 / Pin 53 | T_MISO | 触摸 SPI 数据输入，接 J1-26；引出 J2-48 | 输入 | 浮空 | 复用功能或浮空输入 | SPI2_MISO（AF5） |
| PB15 / Pin 54 | T_MOSI | 触摸 SPI 数据输出，接 J1-25；引出 J3-48 | 输出 | 浮空 | 复用推挽或推挽输出 | SPI2_MOSI（AF5） |
| PC0 / Pin 15 | PC0 | 引出 J2-17 | 未使用 | 浮空 | — | 仅排针引出（ADC123_IN10） |
| PC1 / Pin 16 | PC1 | 引出 J2-18 | 未使用 | 浮空 | — | 仅排针引出（ADC123_IN11） |
| PC2 / Pin 17 | PC2 | 引出 J2-19 | 未使用 | 浮空 | — | 仅排针引出（ADC123_IN12） |
| PC3 / Pin 18 | PC3 | 引出 J2-20 | 未使用 | 浮空 | — | 仅排针引出（ADC123_IN13） |
| PC4 / Pin 33 | PC4 | 引出 J2-31 | 未使用 | 浮空 | — | 仅排针引出（ADC12_IN14） |
| PC5 / Pin 34 | T_PEN、PC5 | 触摸中断 PENIRQ，经 R17 1.5K 串联接 J1-27；引出 J2-32 | 输入 | UNKNOWN（板上无上下拉） | 中断输入 | 输入 + 内部上拉；EXTI5（EXTI9_5_IRQn），触发边沿见第 5 部分 |
| PC6 / Pin 63 | PC6 | 引出 J3-39 | 未使用 | 浮空 | — | 仅排针引出 |
| PC7 / Pin 64 | PC7 | 引出 J3-38 | 未使用 | 浮空 | — | 仅排针引出 |
| PC8 / Pin 65 | SDIO_D0 | TF 卡 DAT0（U5-7），R28 10K 上拉；引出 J3-37 | 双向 | 外部上拉为高 | 复用推挽 | AF12，高速，上拉 |
| PC9 / Pin 66 | SDIO_D1 | TF 卡 DAT1（U5-8），R29 10K 上拉；引出 J3-36 | 双向 | 外部上拉为高 | 复用推挽 | AF12，高速，上拉 |
| PC10 / Pin 78 | SDIO_D2 | TF 卡 DAT2（U5-1），R12 10K 上拉；引出 J3-29 | 双向 | 外部上拉为高 | 复用推挽 | AF12，高速，上拉 |
| PC11 / Pin 79 | SDIO_D3 | TF 卡 CD/DAT3（U5-2），R24 10K 上拉；引出 J3-28 | 双向 | 外部上拉为高 | 复用推挽 | AF12，高速，上拉 |
| PC12 / Pin 80 | SDIO_SCK | TF 卡 CLK（U5-5），R27 10K 上拉；引出 J3-27 | 输出 | 外部上拉为高 | 复用推挽 | AF12，高速，无上下拉 |
| PC13 / Pin 7 | PC13 | 引出 J2-16 | 未使用 | 浮空 | — | 驱动能力受限（PC13~PC15 同一时刻只能有一个作输出，灌电流不超过 3 mA，速度不超过 2 MHz），不宜直接驱动负载 |
| PC14 / Pin 8 | PC14 | LSE 输入 OSC32_IN，接 Y1 32.768 kHz、C8 12pF | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | 在 RCC 中使能 LSE |
| PC15 / Pin 9 | PC15 | LSE 输出 OSC32_OUT，接 Y1、C9 12pF | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | 在 RCC 中使能 LSE |
| PD0 / Pin 81 | PD0、FSMC_D2 | LCD 数据线 D2，接 J1-16；引出 J3-26 | 双向 | 浮空 | 复用推挽 | AF12，最高速 |
| PD1 / Pin 82 | PD1、FSMC_D3 | LCD 数据线 D3，接 J1-15；引出 J3-25 | 双向 | 浮空 | 复用推挽 | AF12，最高速 |
| PD2 / Pin 83 | SDIO_CMD、PD2 | TF 卡 CMD（U5-3），R26 10K 上拉；引出 J3-24 | 双向 | 外部上拉为高 | 复用推挽 | AF12，高速，上拉 |
| PD3 / Pin 84 | PD3 | 引出 J3-23 | 未使用 | 浮空 | — | 仅排针引出 |
| PD4 / Pin 85 | PD4、FSMC_NOE | LCD 读使能，接 J1-19；引出 J3-22 | 输出 | 浮空 | 复用推挽 | AF12，最高速 |
| PD5 / Pin 86 | PD5、FSMC_NWE | LCD 写使能，接 J1-20；引出 J3-21 | 输出 | 浮空 | 复用推挽 | AF12，最高速 |
| PD6 / Pin 87 | PD6 | 引出 J3-20 | 未使用 | 浮空 | — | 仅排针引出（FSMC_NWAIT 未连接） |
| PD7 / Pin 88 | PD7、FSMC_NE1 | LCD 片选，接 J1-22；引出 J3-19 | 输出 | 浮空 | 复用推挽 | AF12，最高速 |
| PD8 / Pin 55 | PD8、FSMC_D13 | LCD 数据线 D13，接 J1-5；引出 J3-47 | 双向 | 浮空 | 复用推挽 | AF12，最高速 |
| PD9 / Pin 56 | PD9、FSMC_D14 | LCD 数据线 D14，接 J1-4；引出 J3-46 | 双向 | 浮空 | 复用推挽 | AF12，最高速 |
| PD10 / Pin 57 | PD10、FSMC_D15 | LCD 数据线 D15，接 J1-3；引出 J3-45 | 双向 | 浮空 | 复用推挽 | AF12，最高速 |
| PD11 / Pin 58 | PD11 | 引出 J3-44 | 未使用 | 浮空 | — | 仅排针引出 |
| PD12 / Pin 59 | PD12 | 引出 J3-43 | 未使用 | 浮空 | — | 仅排针引出 |
| PD13 / Pin 60 | PD13、FSMC_A18 | LCD RS（命令/数据选择），接 J1-21；引出 J3-42 | 输出 | 浮空 | 复用推挽 | AF12，最高速；A18 作为 RS 线 |
| PD14 / Pin 61 | PD14、FSMC_D0 | LCD 数据线 D0，接 J1-18；引出 J3-41 | 双向 | 浮空 | 复用推挽 | AF12，最高速 |
| PD15 / Pin 62 | PD15、FSMC_D1 | LCD 数据线 D1，接 J1-17；引出 J3-40 | 双向 | 浮空 | 复用推挽 | AF12，最高速 |
| PE0 / Pin 97 | PE0 | 引出 J3-12 | 未使用 | 浮空 | — | 仅排针引出 |
| PE1 / Pin 98 | PE1 | 引出 J3-11 | 未使用 | 浮空 | — | 仅排针引出 |
| PE2 / Pin 1 | PE2 | 引出 J2-11 | 未使用 | 浮空 | — | 仅排针引出 |
| PE3 / Pin 2 | KEY1、PE3 | K1 按键，另一端接 GND；引出 J2-12 | 输入 | 浮空（无外部上拉） | 上拉输入 / 中断输入 | 输入 + 内部上拉；中断用 EXTI3 下降沿；读到低 = 按下 |
| PE4 / Pin 3 | KEY0、PE4 | K0 按键，另一端接 GND；引出 J2-13 | 输入 | 浮空（无外部上拉） | 上拉输入 / 中断输入 | 输入 + 内部上拉；中断用 EXTI4 下降沿；读到低 = 按下 |
| PE5 / Pin 4 | PE5 | 引出 J2-14 | 未使用 | 浮空 | — | 仅排针引出 |
| PE6 / Pin 5 | PE6 | 引出 J2-15 | 未使用 | 浮空 | — | 仅排针引出 |
| PE7 / Pin 38 | PE7、FSMC_D4 | LCD 数据线 D4，接 J1-14；引出 J2-35 | 双向 | 浮空 | 复用推挽 | AF12，最高速 |
| PE8 / Pin 39 | PE8、FSMC_D5 | LCD 数据线 D5，接 J1-13；引出 J2-36 | 双向 | 浮空 | 复用推挽 | AF12，最高速 |
| PE9 / Pin 40 | PE9、FSMC_D6 | LCD 数据线 D6，接 J1-12；引出 J2-37 | 双向 | 浮空 | 复用推挽 | AF12，最高速 |
| PE10 / Pin 41 | PE10、FSMC_D7 | LCD 数据线 D7，接 J1-11；引出 J2-38 | 双向 | 浮空 | 复用推挽 | AF12，最高速 |
| PE11 / Pin 42 | PE11、FSMC_D8 | LCD 数据线 D8，接 J1-10；引出 J2-39 | 双向 | 浮空 | 复用推挽 | AF12，最高速 |
| PE12 / Pin 43 | PE12、FSMC_D9 | LCD 数据线 D9，接 J1-9；引出 J2-40 | 双向 | 浮空 | 复用推挽 | AF12，最高速 |
| PE13 / Pin 44 | PE13、FSMC_D10 | LCD 数据线 D10，接 J1-8；引出 J2-41 | 双向 | 浮空 | 复用推挽 | AF12，最高速 |
| PE14 / Pin 45 | PE14、FSMC_D11 | LCD 数据线 D11，接 J1-7；引出 J2-42 | 双向 | 浮空 | 复用推挽 | AF12，最高速 |
| PE15 / Pin 46 | PE15、FSMC_D12 | LCD 数据线 D12，接 J1-6；引出 J2-43 | 双向 | 浮空 | 复用推挽 | AF12，最高速 |
| PH0 / Pin 12 | OSC_IN | HSE 输入，接 Y2 8 MHz、C10 22pF、R10 1MΩ | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | `HSE_VALUE=8000000`，使能 HSE |
| PH1 / Pin 13 | OSC_OUT | HSE 输出，接 Y2、C11 22pF、R10 | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | 同上 |
| NRST / Pin 14 | RST | 复位：R6 10K 上拉、C6 104、RST 按键；接 P1-15、J1-2（LCD 复位） | 电源/时钟/复位/调试 | 高（不复位） | 无需 GPIO 配置 | — |
| BOOT0 / Pin 94 | BOOT0 | R7 10K 下拉；经 R23 510R 接 J3-5 跳线 | 电源/时钟/复位/调试 | 低（主 Flash 启动） | 无需 GPIO 配置 | — |
| VBAT / Pin 6 | VBAT | 经 Q1 BAT54C 取 3V3 或 B1 电池；C7 104 | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | 备份域供电 |
| VDD_1 / Pin 50 | 3V3 | 数字电源 | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | — |
| VDD_2 / Pin 75 | 3V3 | 数字电源 | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | — |
| VDD_3 / Pin 100 | 3V3 | 数字电源 | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | — |
| VDD_4 / Pin 28 | 3V3 | 数字电源 | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | — |
| VDD_5 / Pin 11 | 3V3 | 数字电源 | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | — |
| VDD / Pin 19 | VDDA | 与 VDDA 并接，经 R34 0R 接 3V3 | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | — |
| VDDA / Pin 22 | VDDA | 模拟电源，经 R34 0R 接 3V3；C23 104、C24 106 | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | — |
| VREF+ / Pin 21 | VREF+ | 经 R11 0R 接 VDDA；引出 J2-22 | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | ADC 参考电压约 3.3V |
| VREF- / Pin 20 | VREF- | 经 R35 0R 接 GND；引出 J2-21 | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | — |
| VSS_2 / Pin 74 | GND | 地 | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | — |
| VSS_3 / Pin 99 | GND | 地 | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | — |
| VSS_4 / Pin 27 | GND | 地 | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | — |
| VSS_5 / Pin 10 | GND | 地 | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | — |
| VCAP_1 / Pin 49 | 未命名 | 接 C21 106 到 GND | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | — |
| VCAP_2 / Pin 73 | 未命名 | 接 C22 106 到 GND | 电源/时钟/复位/调试 | — | 无需 GPIO 配置 | — |

# 4、驱动逻辑分析

| 模块/外设 | 相关网络/MCU引脚 | 电路原理 | 输入/输出电平状态 | 驱动方式 | 软件建议 |
|---|---|---|---|---|---|
| LED D2、D3 | PA6、PA7 | LED 阳极接 3V3，阴极分别经 R13、R14 510R 接 MCU 引脚 | 输出低 = 亮，输出高 = 灭 | 推挽输出 | 初始化时先输出高再切换为输出模式，避免上电闪烁；调光用 TIM3_CH1/CH2（AF2），PWM 极性取反 |
| 电源指示灯 D1 | 无 | 3V3 经 R5 510R 接 D1 到 GND | 上电常亮 | — | 无 |
| 按键 K_UP | PA0/WK_UP | 3V3 经 R16 10K 接按键，按键另一端接 PA0；无外部下拉 | 按下 = 高，松开 = 浮空 | 下拉输入 / 中断输入 | 必须开内部下拉；中断用 EXTI0 上升沿；PA0 同时是 WKUP 脚，可把系统从待机模式唤醒；需要软件消抖 |
| 按键 KEY0、KEY1 | PE4/KEY0、PE3/KEY1 | 按键一端接 MCU，另一端接 GND；无外部上拉 | 按下 = 低 | 上拉输入 / 中断输入 | 必须开内部上拉；中断用 EXTI4、EXTI3 下降沿；需要软件消抖 |
| 复位按键 | NRST（RST 网络） | R6 10K 上拉 + C6 104 组成 RC 复位，按键接 GND | 低电平复位 | 硬件 | 无需 GPIO；按下复位键时 LCD 也一起复位 |
| ISP 串口 USART1 | PA9/TXD1、PA10/RXD1，J6 | MCU 引脚直连 4 针座（5V、GND、RX、TX），板上无电平转换芯片 | 3.3V TTL | 复用功能 AF7 | 外接 3.3V TTL 串口模块（TX 接 J6-3，RX 接 J6-4）；BOOT0=1、BOOT1=0 时可用于系统 Bootloader 下载；RX 开内部上拉 |
| USB Slave | PA11/USB_DM、PA12/USB_DP，J4 | D-、D+ 分别串 R18、R19 22Ω 到 Mini USB；D+ 由 R21 1.5K 固定上拉到 3V3；VBUS 经 R25 接 5V，不接 MCU | 全速（12 Mbps）设备 | 复用功能 AF10（OTG_FS） | 设备模式；关闭 VBUS 检测（HAL：`vbus_sensing_enable = DISABLE`）；PLLQ=7 得到 48 MHz；D+ 外部上拉常接，软件无法靠断开上拉让主机重新枚举，需要插拔线缆 |
| W25Q16 SPI Flash | PB0/F_CS、PB3/SPI1_SCK、PB4/SPI1_MISO、PB5/SPI1_MOSI | U3：CS# 由 R20 10K 上拉；CLK 串 R22 22Ω；WP#、HOLD# 直接接 3V3（不使用写保护和保持功能）；VCC 去耦 C20 104 | CS 低 = 选中 | SPI1 复用 AF5 + CS 用 GPIO | SPI1 主模式，模式 0 或模式 3，8 位，高位先发，CS 由软件控制；与 NRF 共用 SPI1，访问 Flash 前确保 NRF_CS=1；PB3、PB4 上有 JTAG 的 10K 上拉，不影响 SPI |
| NRF24L01 接口 JP2 | PB6/NRF_CE、PB7/NRF_CS、PB8/NRF_IRQ、SPI1（PB3/PB4/PB5） | 2×4 排座：1 GND、2 3V3、3 CE、4 CSN、5 SCK、6 MOSI、7 MISO、8 IRQ；板上没有上下拉 | CE 高 = 收发使能；CSN 低 = 选中；IRQ 有效电平原理图无法证明（UNKNOWN） | GPIO + SPI1 | NRF_CS 初始化为高、NRF_CE 初始化为低；SPI 模式 0；nRF24L01 SPI 时钟上限 10 MHz，APB2 为 84 MHz 时分频 8 得 10.5 MHz 超限，应选分频 16（5.25 MHz），或访问 NRF 和 Flash 时分别切换分频；IRQ 用 EXTI8 |
| TF 卡 U5 | SDIO：PC8~PC11（D0~D3）、PC12（CK）、PD2（CMD） | D0~D3、CMD、CLK 均有 10K 上拉到 3V3；卡座 VCC 常接 3V3（C19 104），无供电开关；没有独立的卡检测脚（CD 与 DAT3 复用） | — | SDIO 复用 AF12 | 4 位总线；识别阶段时钟不超过 400 kHz，之后可提高到 24 MHz（SDIOCLK 48 MHz，CLKDIV=0）；需要 PLLQ=7；无卡检测 GPIO，通过初始化命令是否应答判断卡是否插入 |
| TFT LCD 并口 J1 | FSMC：D0~D15（PD14、PD15、PD0、PD1、PE7~PE15、PD8、PD9、PD10），NOE（PD4），NWE（PD5），NE1（PD7），A18（PD13，作 RS） | 16 位 8080 并口；J1-2 接系统复位 RST；J1-31 接 3V3；J1-1、J1-30、J1-32 接 GND；J1-29 NC | — | FSMC 复用 AF12 | FSMC Bank1 子区 1（NE1），16 位数据宽度，异步 SRAM 模式 A；RS 接 A18，因此命令地址为 0x6000_0000，数据地址为 0x6008_0000（16 位宽度时 A18 对应 HADDR[19]）；原理图注明 ZET6 版本用 NE4 和 A6，移植 ZET6 的 LCD 例程时要改成 NE1、A18 |
| LCD 背光 | PB1/LCD_BL | PB1 经 R30 1.5K 接 Q2（PNP）基极（Q2-1）；发射极（Q2-2）接 3V3；集电极（Q2-3）接 J1-28（背光电源），并经 R32 1.5K 接 GND；R31 未贴 | PB1 低 = 背光亮，PB1 高 = 背光灭 | 推挽输出 / PWM | 上电先输出高（灭），LCD 初始化完成后再拉低，避免花屏；调光用 TIM3_CH4（AF2），PWM 极性取反 |
| 触摸接口 | PB12/T_CS、PB13/T_SCK、PB14/T_MISO、PB15/T_MOSI、PC5/T_PEN | 接 J1-24、J1-23、J1-26、J1-25；PENIRQ（J1-27）经 R17 1.5K 串联接 PC5；板上没有上拉；触摸控制器在 LCD 模组上 | 片选和 PENIRQ 的有效电平原理图无法证明（UNKNOWN） | SPI2 复用 AF5 或 GPIO 模拟 | PB12~PB15 恰好是 SPI2 的 NSS/SCK/MISO/MOSI，可用硬件 SPI2；T_PEN 设为输入 + 内部上拉，用 EXTI5（EXTI9_5_IRQn） |
| RTC 与后备电池 | PC14、PC15、VBAT | Y1 32.768 kHz 接 LSE；VBAT 由 BAT54C 在 3V3 和 B1 之间自动选择 | — | — | RTC 时钟源选 LSE；装了 B1 时掉电后 RTC 和备份数据保持，上电时先检查备份寄存器标志，避免重复初始化 RTC |
| JTAG/SWD 调试 | PA13、PA14、PA15、PB3、PB4、RST | R1~R4 10K 上拉、R9 10K 下拉；P1 为 20 针 ARM JTAG | — | 复用功能 AF0 | 不要重新配置 PA13、PA14；使用 SPI1 时只能用 SWD 调试 |
| 启动跳线 | BOOT0、PB2/BOOT1 | R7、R8 10K 下拉；经 R23、R33 510R 接 J3 跳线 | 默认均为低 | 硬件 | PB2 有下拉且只接到跳线，不要作普通 IO 使用 |
| 扩展排针 J2、J3 | 见 IO 分配表 | 绝大部分 IO 都引到了排针，包括已被板载外设占用的 FSMC、SDIO、SPI1、USART1、USB、LED、按键、触摸引脚 | — | — | 外接模块时避开已占用引脚。不被任何板载外设占用的引脚：PA1~PA5、PA8、PA15（用 SWD 时）、PB9~PB11、PC0~PC4、PC6、PC7、PC13、PD3、PD6、PD11、PD12、PE0~PE2、PE5、PE6 |

# 5、用户待补充事项

| 已连接对象 | 相关 MCU 网络 | 当前已知 | 需要补充后才能写具体驱动代码的事项 | 参考示例 |
|---|---|---|---|---|
| TFT LCD 模组（J1） | FSMC_D0~D15、FSMC_NOE、FSMC_NWE、FSMC_NE1、FSMC_A18、LCD_BL（PB1）、RST | 16 位 8080 并口，RS 接 A18，背光 PNP 低有效，LCD 复位与系统复位共用 | LCD 控制器型号、分辨率、显示方向、颜色格式，以及控制器要求的读写时序 | 示例：ILI9341，240×320，竖屏，RGB565；FSMC 模式 A，地址建立 1 个 HCLK、数据建立 5 个 HCLK（HCLK 168 MHz）；上电延时 120 ms 后发送初始化序列 |
| 触摸控制器（J1） | T_CS（PB12）、T_SCK（PB13）、T_MISO（PB14）、T_MOSI（PB15）、T_PEN（PC5） | 引脚对应 SPI2（AF5）；PENIRQ 经 R17 串联，板上无上拉 | 触摸芯片型号、片选和 PENIRQ 的有效电平、采样方式（中断或轮询）、校准方式 | 示例：XPT2046，T_CS 低有效，PENIRQ 为低表示触摸按下；SPI2 模式 0，时钟不超过 2 MHz；PENIRQ 下降沿中断后，X、Y 各采样 5 次取中值；首次上电做 4 点校准，参数保存到 W25Q16 |
| NRF24L01 模块（JP2） | NRF_CE（PB6）、NRF_CS（PB7）、NRF_IRQ（PB8）、SPI1_SCK/MISO/MOSI | 与 W25Q16 共用 SPI1；IRQ 板上无上拉 | 模块型号及 IRQ 有效电平；射频信道、空中速率、发射功率；地址宽度和地址；载荷长度；是否开启自动应答和重发；本机是发送端还是接收端 | 示例：nRF24L01+ 模块，IRQ 低有效；信道 40，2 Mbps，0 dBm；5 字节地址 0x34,0x43,0x10,0x10,0x01；载荷 32 字节；开启自动应答，最多重发 15 次、间隔 500 µs；本机为接收端 |
| ISP 串口 USART1（J6） | PA9/TXD1、PA10/RXD1 | 直连 J6，3.3V TTL，无流控 | 应用程序中的波特率、数据格式和用途（调试打印还是通信协议） | 示例：115200 bps，8N1，无流控；作为调试串口，printf 重定向到 USART1；接收用 RXNE 中断配合环形缓冲区 |
| USB Slave（J4） | PA11/USB_DM、PA12/USB_DP | OTG_FS 设备模式，VBUS 不接 MCU，D+ 固定上拉 | USB 设备类型及描述符信息（VID/PID、供电方式等） | 示例：CDC 虚拟串口，VID 0x0483 / PID 0x5740，总线供电，全速，端点最大包长 64 字节 |
| 用户按键 | PA0/WK_UP、PE4/KEY0、PE3/KEY1 | K_UP 按下为高；KEY0、KEY1 按下为低 | 检测方式（中断或定时扫描）、消抖时间、是否需要识别长按、K_UP 是否用于待机唤醒 | 示例：每 10 ms 扫描一次，连续两次状态一致才确认（20 ms 消抖）；按住超过 1 s 判为长按；K_UP 同时作为待机唤醒脚（使能 PWR_CSR.EWUP） |
