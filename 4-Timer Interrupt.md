# Timer Interrupt

## Introduction

There are three kinds of timer in STM32: **advanced timers**, **general purpose timers** and **basic timers**. **General purpose timers** are the most common timers in STM32, which supports PWM generation, input capture, time-base generation(update interrupt) and output compare. **Basic timers** don’t have IOs channels for input capture/PWM generation so that they are only used in time-base generation purposes. **Advanced timers** have more feature than general purpose timers like complementary PWM generation. 



![1571487591548](Timer%20Interrupt.assets/1571487591548.png)

Most of the STM32 timers have a 16-bit auto-reload counter and a 16-bit programmable prescaler used to divide the counter clock frequency by any factor between 1 and 65535. Unlike most other MCUs in which timers usually count incrementally, STM32 timers can count up, down or center-aligned(TIM6 and TIM7 in STM32RCT6 only support up-counting mode). As the above figure shows, the counter increases/decreases its value by one(depends on the counter mode) at each clock tick. Once the counter overflow(reach zero or auto-reload(ARR) value), it generates an update event(interrupt), set the counter as zero or the ARR value and continue to the next period.

![1571487991789](Timer%20Interrupt.assets/1571487991789.png)

![1571488000572](Timer%20Interrupt.assets/1571488000572.png)

![1571488007773](Timer%20Interrupt.assets/1571488007773.png)



## Configuration in STM32CubeIDE

All the timers in STM32 support timer interrupt(time-base generation). Let’s take **TIM3** as an example.

### Clock Source

![1571489528864](Timer%20Interrupt.assets/1571489528864.png)

Select the clock source as **Internal Clock**. And there comes a question: what is the frequency of the **Internal Clock**? To answer this question, we need to back to the system architecture of this MCU:

![1571490039757](Timer%20Interrupt.assets/1571490039757.png)

It is clear that **TIM3** is connected on the APB1 bus. Check the clock configuration of our project and find the **Internal Clock** of **TIM3** is **APB1 timer clocks**, whose frequency is 72MHz.

![1571490464847](Timer%20Interrupt.assets/1571490464847.png)

### Counter Setting

From the above content, we can easily find a relationship about update interrupt period and internal clock:

*T = (arr+1)(psc+1)/f_clk*

where:

- *T* is the update interrupt period(us)
- *f_clk* is the frequency of the internal clock(MHz​)
- *arr* is the auto-reload value
- *psc* is the prescaler

If we want to set the update interrupt period *T* as 1s, the value of *arr* and *psc* can be 9999 and 7199.

![1571492734737](Timer%20Interrupt.assets/1571492734737.png)

### Enable the Interrupt

![1571492855163](Timer%20Interrupt.assets/1571492855163.png)

### API

Call the following function to start/stop timer in interrupt mode(for example in the main routine):

```c
HAL_StatusTypeDef HAL_TIM_Base_Start_IT(TIM_HandleTypeDef *htim)
HAL_StatusTypeDef HAL_TIM_Base_Stop_IT(TIM_HandleTypeDef *htim)
```

```c
int main(void)
{
  //...
  /* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_TIM3_Init();
  MX_USART1_UART_Init();
  /* USER CODE BEGIN 2 */
  HAL_TIM_Base_Start_IT(&htim3);
  /* USER CODE END 2 */
  //...
}
```

All the interrupts of TIM3 are handled by this function in **stm32f1xx_it.c**:

```c
/**
  * @brief This function handles TIM3 global interrupt.
  */
void TIM3_IRQHandler(void)
{
  /* USER CODE BEGIN TIM3_IRQn 0 */

  /* USER CODE END TIM3_IRQn 0 */
  HAL_TIM_IRQHandler(&htim3);
  /* USER CODE BEGIN TIM3_IRQn 1 */

  /* USER CODE END TIM3_IRQn 1 */
}
```

which calls the public TIM interrupt handler `HAL_TIM_IRQHandler`, which will call the corresponding callback function according to the interrupt type and clear the corresponding interrupt pending bits:

```c
void HAL_TIM_IRQHandler(TIM_HandleTypeDef *htim)
{
  //...
  /* TIM Update event */
  if (__HAL_TIM_GET_FLAG(htim, TIM_FLAG_UPDATE) != RESET)
  {
    if (__HAL_TIM_GET_IT_SOURCE(htim, TIM_IT_UPDATE) != RESET)
    {
      __HAL_TIM_CLEAR_IT(htim, TIM_IT_UPDATE);
#if (USE_HAL_TIM_REGISTER_CALLBACKS == 1)
      htim->PeriodElapsedCallback(htim);
#else
      HAL_TIM_PeriodElapsedCallback(htim);
#endif /* USE_HAL_TIM_REGISTER_CALLBACKS */
    }
  }
  //..
}
```

```c
/**
  * @brief  Period elapsed callback in non-blocking mode
  * @param  htim TIM handle
  * @retval None
  */
__weak void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
  /* Prevent unused argument(s) compilation warning */
  UNUSED(htim);

  /* NOTE : This function should not be modified, when the callback is needed,
            the HAL_TIM_PeriodElapsedCallback could be implemented in the user file
   */
}
```

Re-implement the ``HAL_TIM_PeriodElapsedCallback``.

## Task
Use timer interrupt to blink the LED and send data by UART