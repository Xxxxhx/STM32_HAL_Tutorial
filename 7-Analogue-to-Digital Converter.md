# Analogue-to-Digital Converter

Analogue-to-Digital Converter is a system that converts an analog signal into a digital signal. STM32 series MCU has 1 to 3 ADCs, while STM32F103RCT6 has three. All these ADC are independent. The 12-bit ADC has up to 18 multiplexed channels allowing it to measure signals from 16 external and two internal sources. A/D conversion of the various channels can be performed in single, continuous, scan or discontinuous mode. The result of the ADC is stored in a left-aligned or right-aligned 16-bit data register. Following is the relationship between the ADC channel and GPIO.

| Channel |            ADC1            | ADC2 | ADC3 |
| :-----: | :------------------------: | :--: | :--: |
|    0    |            PA0             | PA0  | PA0  |
|    1    |            PA1             | PA1  | PA1  |
|    2    |            PA2             | PA2  | PA2  |
|    3    |            PA3             | PA3  | PA3  |
|    4    |            PA4             | PA4  |      |
|    5    |            PA5             | PA5  |      |
|    6    |            PA6             | PA6  |      |
|    7    |            PA7             | PA7  |      |
|    8    |            PB0             | PB0  |      |
|    9    |            PB1             | PB1  |      |
|   10    |            PC0             | PC0  | PC0  |
|   11    |            PC1             | PC1  | PC1  |
|   12    |            PC2             | PC2  | PC2  |
|   13    |            PC3             | PC3  | PC3  |
|   14    |            PC4             | PC4  |      |
|   15    |            PC5             | PC5  |      |
|   16    | Temperature Sensor Channel |      |      |
|   17    |      Vrefint Channel       |      |      |



## Configuration on STM32CubeIDE

Go to the **Analog** categories, click **ADC1** and select **IN1**, which means we enable the channel 1 of ADC1 and we are able to measure the voltage of **PA1**

![1572877316401](Analogue-to-Digital%20Converter.assets/1572877316401.png)

Remember to set the data alignment to **Right alignment** and keep the ADC clock normal

![1572874025787](Analogue-to-Digital%20Converter.assets/1572874025787.png)

![1572874263139](Analogue-to-Digital%20Converter.assets/1572874263139.png)

## Single Channel, Single Conversion Mode

In single conversion mode, ADC only does one conversion, so we need to restart conversion if we want to update the measurements 

```c
// in main.c file
int main(void)
{
  // ...
  /* USER CODE BEGIN 1 */
  uint16_t raw;
  char msg[20];
  // ...
  while (1)
  {
    HAL_ADC_Start(&hadc1);
    // Wait for regular group conversion to be completed
    HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY);
    // Get ADC value
    raw = HAL_ADC_GetValue(&hadc1); // the voltage should be raw * (3.3/4096)(12 bits)
    // Convert to string and print
    sprintf(msg, "%hu\r\n", raw);
    HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
 }
}
```

## Single Channel, Continuous Conversion Mode

In continuous conversion mode, ADC starts another conversion as soon as it finishes one, so we don’t need to restart the conversion.

Back to configuration part, enable the continuous conversion mode.

![1572877222130](Analogue-to-Digital%20Converter.assets/1572877222130.png)

```c
// in main.c file
int main(void)
{
  // ...
  /* USER CODE BEGIN 1 */
  uint16_t raw;
  char msg[20];
  // ...
  HAL_ADC_Start(&hadc1);
  while (1)
  {

    // Wait for regular group conversion to be completed
    HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY);
    // Get ADC value
    raw = HAL_ADC_GetValue(&hadc1); // the voltage should be raw * (3.3/4096)(12 bits)
    // Convert to string and print
    sprintf(msg, "%hu\r\n", raw);
    HAL_UART_Transmit(&huart2, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
 }
}
```

## Multiple Channel, Single Conversion Mode

If we wanna to get the measurements of multiple channel and don‘t want to use DMA(Direct Memory Access) and interrupt, only single conversion mode can be used.

![1572879507340](Analogue-to-Digital%20Converter.assets/1572879507340.png)

- Enable the other two channels and discontinuous conversion mode

```c
  while (1)
  {
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  	for (i = 0; i < 3; i++)
  	{
  		HAL_ADC_Start(&hadc1);
  		HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY);
  		adcBuf[i]=HAL_ADC_GetValue(&hadc1);
  		sprintf(msg, "ch:%d  %d\r\n", i, adcBuf[i]);
  		HAL_UART_Transmit(&huart1, (uint8_t*)msg, strlen(msg), HAL_MAX_DELAY);
  	}
    HAL_Delay(500);
  }
```

## Practice

Try to use the ADC to get the measurement of internal temperature sensor

![1572881059595](Analogue-to-Digital%20Converter.assets/1572881059595.png)

The relationship between the voltage and temperature:

T = {(1.43 - V) / 4.3} + 25