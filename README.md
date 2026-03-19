# STM32F407VET6 + LLCC68 LoRa 移植说明

本工程基于 STM32CubeMX 生成的 HAL 工程，在 STM32F407VET6 上移植 LLCC68 驱动，实现 LoRa 收发。

## 1. 硬件连接（依据当前源码）

### 1.1 LLCC68 与 STM32 连接表

| 功能 | STM32 引脚 | 代码定义位置 | 说明 |
|---|---|---|---|
| BUSY | PE2 | `Core/Inc/main.h` | 输入，查询射频芯片忙状态 |
| TX_EN | PE6 | `Core/Inc/main.h` | 输出，控制外部射频开关进入发射通道 |
| RX_EN | PC13 | `Core/Inc/main.h` | 输出，控制外部射频开关进入接收通道 |
| RESET | PB13 | `Core/Inc/main.h` | 输出，LLCC68 复位 |
| NSS/CS | PB14 | `Core/Inc/main.h` | 输出，SPI 片选（软件控制） |
| SCK | PA5 | `Core/Src/spi.c` | SPI1_SCK |
| MISO | PA6 | `Core/Src/spi.c` | SPI1_MISO |
| MOSI | PA7 | `Core/Src/spi.c` | SPI1_MOSI |

### 1.2 串口调试连接

| 功能 | STM32 引脚 | 参数 |
|---|---|---|
| USART1_TX | PA9 | 115200, 8N1 |
| USART1_RX | PA10 | 115200, 8N1 |

> 代码中 `printf` 已重定向到 USART1（见 `Core/Src/usart.c` 的 `fputc`）。

### 1.3 其它引脚

| 功能 | STM32 引脚 | 说明 |
|---|---|---|
| Key_3 | PD10 | 下拉输入，当前 LoRa 主流程未使用 |

## 2. 外设配置摘要

### 2.1 时钟

- HSE 打开，PLL 启用。
- `SYSCLK = 168MHz`（由 `PLLM=25, PLLN=336, PLLP=2` 推导）。
- APB1 分频 4，APB2 分频 2。

### 2.2 SPI1（LLCC68 通信）

- 主机模式，双线全双工，8bit。
- CPOL=0，CPHA=1Edge。
- 软件 NSS（片选由 GPIO PB14 手动拉低/拉高）。
- 预分频 `SPI_BAUDRATEPRESCALER_2`。

### 2.3 GPIO 默认状态（上电初始化）

- `TX_EN = 0`
- `RX_EN = 0`
- `RESET = 0`
- `CS = 1`

见 `Core/Src/gpio.c`。

## 3. LoRa 默认参数（见 `Core/Lora/Inf_Lora.h`）

- 频点：`480000000 Hz`
- 发射功率：`17 dBm`
- SF：`SF9`
- 带宽：`125 kHz`
- 编码率：`4/5`
- 同步字：`0x3444`
- CRC：开启
- 前导码长度：`12`

## 4. 软件流程

### 4.1 初始化顺序

1. `HAL_Init()`
2. `SystemClock_Config()`
3. `MX_GPIO_Init()`
4. `MX_SPI1_Init()`
5. `MX_USART1_UART_Init()`
6. `Inf_LoRa_Init()`

### 4.2 收发控制

- 进入接收：`Inf_Lora_EnterRx()`（置位 `RX_EN` 并配置 RX 中断掩码）
- 发送数据：`Inf_Lora_SendData(buf, len)`（内部切到 TX 后发包，再回到 RX）
- 轮询收包：`Inf_Lora_ReceiveData(buf, &len)`

> 当前工程未配置 LLCC68 DIO1 的 MCU 外部中断引脚，接收侧通过轮询 `llcc68_irq_handler()` 处理 IRQ。

## 5. 最小示例

在 `while(1)` 中可加入如下测试逻辑：

```c
#include "Inf_Lora.h"

uint8_t rx_buf[256];
uint8_t rx_len = 0;

Inf_LoRa_Init();

while (1)
{
	Inf_Lora_ReceiveData(rx_buf, &rx_len);
	if (rx_len > 0)
	{
		printf("RX(%d): %s\r\n", rx_len, rx_buf);
		rx_len = 0;
	}
}
```

## 6. 工程说明

- Keil 工程文件位于 `MDK-ARM/`。
- 应提交源码与工程配置文件，不提交编译中间文件与本机配置。
- 仓库已配置 `.gitignore` 用于过滤常见 Keil 输出文件。
