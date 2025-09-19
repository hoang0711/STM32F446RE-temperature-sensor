# Interface Temperature & Humidity Sensor with STM32 MCU

This project will demonstrate how to interface the DHT11 sensor (temperature & humidity) with the STM32F446RE microcontroller 
and display the values on a 16x2 LCD via PCF8574 I2C expander.

The program is written in C using STM32CubeIDE and the HAL library.

![IMG_8345](https://github.com/user-attachments/assets/4c041c4d-16d2-4be4-a2c0-1161dedd878f)
![IMG_8348](https://github.com/user-attachments/assets/dc550ae6-5f30-49bf-a617-dfb072603755)
![IMG_8347](https://github.com/user-attachments/assets/50cbbd8f-d819-4531-a456-66e7d41963e5)
![IMG_8349](https://github.com/user-attachments/assets/4be902dc-9073-4730-be62-692413c4e023)

## Hardware
- STM32F446RE (Nucleo-64) https://www.st.com/resource/en/datasheet/stm32f446mc.pdf 
- DHT11 Temperature & Humidity sensor
- 16x2 LCD display with I2C extender (PCF8574)
- Jumper wires & solderless breadboard

## Overview
- Reads temperature(C) and humidity(%) from the DHT11 sensor
- Displays values on a 16x2 LCD via I2C extender
- Implements a Microsecond delay (delay_us) using TIM6 since HAL_Delay() only supports millisecond delay

## Connections
| MCU Pin | Peripheral | Description |
|---|---|---|
| PA1 | DHT11 Data | Digital data line |
| PB8 | I2C_SCL | LCD I2C clock |
| PB9 | I2C_SDA | LCD I2C  data |
| 3.3V/5V | VCC | Power for DHT11 + LCD |
| GND | GND | Common ground |

Some setup may required a pull-uo resistor (4.7kΩ - 10kΩ) between VCC and the data line of DHT11.

## Microsecond Delay Implementation
The DHT11 sensor communication protocol requires delays between 20 - 80 microseconds, which is not possible with the built-in HAL delay. Therefore, TIM6 is used to implement a custom function delay_us().

```c
void delay_us(uint16_t time)
{
    __HAL_TIM_SET_COUNTER(&htim6, 0);
    while ((__HAL_TIM_GET_COUNTER(&htim6)) < time);
}

