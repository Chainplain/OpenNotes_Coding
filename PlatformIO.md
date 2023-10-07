# Note something worth noting
_Started Oct 7th, written by Chainplain_
## Application Notes
Let us first explain the following code.
``` C
#include <main.h>

/** 外设使用情况
 * HSE_VALUE = 8000000; // HSE_VALUE is set in stm32f4xx_hal_conf.h, default is 25MHz, using -D HSE_VALUE=8000000UL at platform.ini predefined
 * System_Clock 168MHz, APB1=42MHz, APB2=84MHz
 * ADC      PA0(VI),PA1(ADC1),PA2(ADC2),PA3(ADC3),PA6(ADC4),PA7(ADC5),PB0(ADC6),PB1(ADC7)
 * DAC      PA4(DAC1),PA5(DAC2),
 * PWM      PC6(PWM1),PC7(PWM2),PC8(PWM3),PC9(PWM4) -> TIM3/8-CH1-4;PA8(PWM5),PA9(PWM6),PA10(PWM7) -> TIM1-CH1-3
 * LED      PA11(LED1),PA12(LED2),PC10(LED3),PB3(LED4),PB4(LED5),PB5(LED6),PB6(LED7),PB7(LED8),PB8(LED9),PB9(LED10)
 * SW       PA13(SWDIO),PA14(SWCLK)
 * SPI      PB12(SPI2-NSS),PB13(SPI2-SCK),PB14(SPI2-MISO),PB15(SPI2-MOSI)
 * EN       PC4(REST),PC5(GPIO0)
 * GPIO     PC0(D0),PC1(D1),PC2(D2),PC3(D3),PH1(PH1),PC13(PC13),PC14(PC14),PC15(PC15)
 * USART    USART5[PC12(TX5), PD2(RX5)] --> for user-communication
 *          USART4[PC10(TX4), PC11(RX4)],  --> IMU
 *          USART3([PB10]=TX, [PB11]=RX) -> ESP8266
 * DMA      DMA1_Stream1(USART3_RX with DMA_Channel4)
 *          DMA1_Stream3(USART3_TX with DMA_Channel4)
 **/

osThreadId led_thread_id;
osThreadId uart_thread_id;
osThreadId setup_thread_id;

int main(int argc, char **argv)
{
  // 选择是否是OTA的版本，如果是需要设置：
  // 1 在platform.ini中，修改：board_build.ldscript = STM32L432KCUX_FLASH_OTA.ld
  // 2 在platform.ini中，增加：-D OTA_VERSION
  // 3 下载程序不能直接下载，需要通过无线下载，或者使用 stm32cubeprogrammer
#ifdef OTA_VERSION // use board_build.ldscript = STM32L432KCUX_FLASH_OTA.ld in platform.ini
  SCB->VTOR = 0x08008000;
#endif

  HAL_Init();

  // HSE=8MHz, CPU=168MHz
  // 要修改HSE_VALUE的值（在stm32f4xx_hal_conf.h）文件里
  SystemClockHSE_PLL_Config(); // 外部晶振
  //SystemClockHSI_PLL_Config();

  // iwatchdog_init();

  GPIO_PINs_Init();

  ADC_init();

  tim_init();

  osThreadDef(ledControl, vTaskLedBreathing, osPriority::osPriorityIdle, 0, 256);
  led_thread_id = osThreadCreate(osThread(ledControl), NULL);

  osThreadDef(uartControl, vTaskUart, osPriority::osPriorityNormal, 0, 1024);
  uart_thread_id = osThreadCreate(osThread(uartControl), NULL);

  osThreadDef(setup, vTaskControl, osPriority::osPriorityLow, 0, 1024);
  setup_thread_id = osThreadCreate(osThread(setup), NULL);

  osKernelStart();
}
```

First, we can see the function 
``` C
osThreadDef(ledControl, vTaskLedBreathing, osPriority::osPriorityIdle, 0, 256)
```

Here is a breakdown of the parameters:
- `thread_name`: This is a user-defined name for the thread, which can be used to refer to the thread in other parts of the code.
- `thread_function`: This is the name of the function that will be executed when the thread is created. It should be of type void and take no arguments.
- `thread_priority`: This specifies the priority of the thread. FreeRTOS uses numerical priorities, where a lower number represents a higher priority. For example, priority 0 is the highest priority, while priority 15 is the lowest.
- `thread_instances`: This parameter specifies the number of instances of the thread that can run concurrently. In most cases, you would only need a single instance of a thread, so this value is typically set to 1.
- 
Then let us see the vTaskUart:
``` C
void vTaskUart(void *pvParameters)
{
	uart_init();

	while (true)
	{
		for (int i = 0; i < USE_UART_NUM; i++)
		{
			if (uart_handle[i] != NULL)
				uart_handle[i]->func_rx_check(uart_handle[i]);
		}
		vTaskDelayMs(RX_ROLLING_INTERVEL_MS);
	}

	// 如果任务不是永久性的需要调用 vTaskDelete 删除任务
	//vTaskDelete(NULL);
}
```

We want to first send something out, we have to ascertain which UART we are use.
``` C
void uart_init(void)
{
	uart_handle[0] = rs232_computer_init();
}
```
Follow this clue, we further find 
``` C
UART232_Handle *rs232_computer_init(void)
{
    userComm = &uartComm;
    rs232_handle = &uart232_handle;
    uart_computer_init(115200);
    return rs232_handle;
}
```
Futher, the uart is defined as 
``` C
uart232_handle.huart = &UartHandle;
```

And the uart handle is defined as 
``` C
typedef struct __UART232_Handle
{
	UART_HandleTypeDef *huart;
	
	// 与DMA有关的变量
	uint32_t unReadIndex; // 代表一个有效桢的第一个数据的位置
	uint32_t unCheckIndex; // 代表一个有效桢的最后一个数据的位置
	uint16_t toBeReadLength;
	
	uint32_t txBufferSize;
	uint32_t rxBufferSize;	
	uint8_t *txCmdBuffer;
	uint8_t *txDataBuffer;
	uint8_t *txBuffer;
	uint8_t *rxBuffer;
	
	void (* func_data_analysis) (struct __UART232_Handle * hrs232);
	void (* func_rx_check) (struct __UART232_Handle * hrs232);
	int (* func_crc_check) (struct __UART232_Handle * hrs232);
	UART_FRAME rxFrame;
	UART_FRAME txFrame;
}UART232_Handle;
```

Further, we can find ` UartHandle.Instance = USART_COMPUTER;`,
and we also have `#define USART_COMPUTER UART5`.

## Shematic
From stm32f405rg.pdf we find:
![image](https://github.com/Chainplain/OpenNotes_Coding/assets/13344614/7afe674a-312d-4b1e-a581-bdc7e58de5aa)
![image](https://github.com/Chainplain/OpenNotes_Coding/assets/13344614/5c537dbf-e4c9-4e20-885f-8c02962eee3b)

and the Circuit schematic diagram
![image](https://github.com/Chainplain/OpenNotes_Coding/assets/13344614/da58ab91-3703-4b5a-93a0-bd9e28a3c31f)
![image](https://github.com/Chainplain/OpenNotes_Coding/assets/13344614/c9f687dd-4bb3-457e-8cd5-61d0c30b7f9c)
