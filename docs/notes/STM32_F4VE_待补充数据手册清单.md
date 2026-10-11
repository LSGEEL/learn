# STM32_F4VE 核心板：驱动开发待补充数据手册清单

- 依据：[STM32_F4VE_原理图功能引脚分析.md](STM32_F4VE_原理图功能引脚分析.md)（原理图来源：`资料/STM32_F4VE--核心板原理图.PDF`）
- 已有资料（`资料` 目录）：STM32F407VE 数据手册、RM0090 参考手册、PM0214 编程手册、STM32CubeF4 软件包
- 整理日期：2026-10-11

## 优先级说明

| 优先级 | 含义 |
|---|---|
| P0 | 器件型号尚未确定，会直接阻塞驱动编写，需要先确认型号再找手册 |
| P1 | 型号已知，写驱动时必须查阅手册 |
| P2 | 手册基本已有，或只在排查问题时才需要 |

## 待补充数据手册清单

| 优先级 | 外设/模块 | 芯片/器件型号 | 为什么需要补数据手册 | 立创或淘宝链接 |
|---|---|---|---|---|
| P0 | TFT LCD 模组（J1，FSMC 16 位并口） | **待确认**。原理图只画了接口，没有标控制器型号（这类屏常见 ILI9341 等） | 初始化命令序列、显示方向寄存器、写显存命令、FSMC 读写时序都取决于控制器型号，没有手册就写不了驱动。还要用模组规格书核对它的 32 针排针定义是否和 J1 一致 | 立创没有这类模组。淘宝搜索：[STM32F407VET6 开发板 TFT 液晶屏](https://s.taobao.com/search?q=STM32F407VET6%20%E5%BC%80%E5%8F%91%E6%9D%BF%20TFT%E6%B6%B2%E6%99%B6%E5%B1%8F)（建议向卖家要模组规格书和控制器手册） |
| P0 | 触摸控制器（在 LCD 模组上） | **待确认**。电阻屏常见 XPT2046，与 TSC2046、ADS7846 兼容 | 需要查：命令字格式（X/Y 通道选择、12 位或 8 位模式）、SPI 时序和最高时钟、PENIRQ 的有效电平。这些决定 T_CS（PB12）和 T_PEN（PC5）怎么配置 | 立创 [XPT2046（C19076）](https://item.szlcsc.com/19779.html)，页面显示缺货，但可下载手册 |
| P1 | NRF24L01 无线模块（JP2） | nRF24L01+。模块上的实际芯片要看丝印，市面上也有兼容芯片 | 需要查：寄存器表、SPI 命令、CE/CSN 时序、IRQ 有效电平和清除方法、SPI 最高 10 MHz。它和 W25Q16 共用 SPI1，要按它的时钟上限选分频 | 芯片：立创 [NRF24L01P-R（C8791）](https://item.szlcsc.com/9305.html)。模块：淘宝搜索 [NRF24L01 无线模块](https://s.taobao.com/search?q=NRF24L01%20%E6%97%A0%E7%BA%BF%E6%A8%A1%E5%9D%97) |
| P1 | 外部 SPI Flash（U3） | W25Q16。BOM 只写到 W25Q16，具体后缀要看芯片丝印，目前在售的是 W25Q16JV | 需要查：指令集（读 ID、读数据、页编程、扇区和块擦除）、状态寄存器的 BUSY 位、页和扇区大小、擦写耗时、最高 SPI 时钟 | 立创 [W25Q16JVSSIQ（C82317）](https://item.szlcsc.com/83474.html) |
| P2 | 主控 MCU（U2） | STM32F407VET6 | `资料` 目录里已有数据手册、RM0090、PM0214，**只缺勘误表 ES0182**。ES0182 列出了 FSMC、SDIO、USB 等外设的已知问题和规避方法 | 立创 [STM32F407VET6（C28730）](https://item.szlcsc.com/29482.html)、[立创数据手册页](https://item.szlcsc.com/datasheet/STM32F407VET6/29482.html)。勘误表在 ST 官网：[ES0182](https://www.st.com/resource/en/errata_sheet/es0182-stm32f405407xx-and-stm32f415417xx-device-errata-stmicroelectronics.pdf) |
| P2 | TF 卡（U5，SDIO） | 这里需要的是 SD 协议规范，不是芯片手册：SD Physical Layer Simplified Specification | 直接用 HAL 的 SD 驱动一般用不到。自己写 SDIO 底层，或排查卡初始化失败时，需要查命令和卡状态的定义 | 不在立创或淘宝，从 SD 协会官网下载：[sdcard.org](https://www.sdcard.org/downloads/pls/) |

## 不需要补手册的器件

| 器件 | 原因 |
|---|---|
| U1 AMS1117-3.3 稳压芯片 | 没有使能脚或状态输出，不需要软件控制 |
| Q1 BAT54C 双二极管 | VBAT 自动切换，不需要软件控制 |
| Q2 背光 PNP 三极管 | 只用 GPIO 或 PWM 控制开关，电平逻辑在原理图里已能确定 |
| Y1 32.768 kHz、Y2 8 MHz 晶振 | 频率已在原理图中标明，时钟配置不需要额外资料 |
| J4 Mini USB 座 | 无软件交互；USB 部分用 RM0090 加 HAL 库即可 |
| LED D1~D3、按键 K_UP/K0/K1/RST | 只用 GPIO，电平逻辑在原理图里已能确定 |

## 关于链接

- 立创商品页是 2026-10-11 通过网页搜索找到的，打开确认过页面存在；价格和库存会变动，以页面实际显示为准。
- 淘宝商品页无法直接核实，所以淘宝两项给的是搜索链接，不是具体商品页。
- LCD 控制器和触摸芯片的型号，最好先看实物丝印或问卖家确认，再按确定的型号下载手册。
