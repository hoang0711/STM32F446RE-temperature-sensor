# Interface Temperature & Humidity Sensor with STM32 MCU

This project will demonstrate how to interface the DHT11 sensor (temperature & humidity) with the STM32F446RE microcontroller 
and display the values on a 16x2 LCD via PCF8574 I2C expander. I picked DHT11 sensor due to its simplicity and ease of use. 
It only requires one GPIO pin for communication and no complex ADC conversion is required. Its 3.3V - 5V compatibility makes 
it suitable for most microcontrollers.

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

```

## Clock Configuration in STM32CubeIDE
Set the external crystal to 8MHz and the HCLK to 50 MHz.

![clock configuration](https://github.com/user-attachments/assets/68460e9b-4f0f-4f51-9f0a-28462c90333a)


## Timer Settings
TIM6 is used to generate delays in microseconds. Since TIM6 is connected to the APB1 bus and the ABP1 timer clock is at 50 MHz, enter 50-1 in the prescaler section to bring the TIM6 clock to 1 MHz. 
The PA1 pin will be used as an output for DHT11.

![TIM6](https://github.com/user-attachments/assets/def5c611-a0e1-483a-9f9b-1f089bd067ec)
![DHT11 pin](https://github.com/user-attachments/assets/3356ec2a-c428-4543-b350-3a92eb98c5ab)


## I2C Settings
The 16x2 LCD uses an I2C PCF8574 extender to connect to the MCU to display the temperature and humidity. Activate the I2C1, select 'standard' speed mode and set the clock speed to 100000 Hz. 
Configure PB8 pin as SCL and PB9 pin as SDA.

![I2C parameter](https://github.com/user-attachments/assets/2e193797-ee84-4064-908e-349b2d7aa6e4)
![I2C pin](https://github.com/user-attachments/assets/8a1c69ec-bec4-4666-acf2-12d9412f697e)


# Code for DHT11 Communication with HAL library
## Initialization

```c
void MCU_StartSignal (void)
{
	HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, 1);
	HAL_Delay(300);

	Set_GPIO_Output(DHT11_PORT, DHT11_PIN);				// set PortA Pin1 as output
	HAL_GPIO_WritePin(DHT11_PORT, DHT11_PIN, 0);		// set bit 0 to pull pin LOW
	HAL_Delay(18);										// delay for 18 milliseconds
	HAL_GPIO_WritePin(DHT11_PORT, DHT11_PIN, 1);		// set bit 1 to pull pin HIGH
	delay_us(20);										// delay for 20 microseconds
	Set_GPIO_Input(DHT11_PORT, DHT11_PIN);				// finally set PortA Pin1 as input
}
```
- Set pin PA1 as output.
- Pull the pin low and delay for 18 milliseconds.
- Pull the pin high for wait for 20 microseconds.
- Set the pin as input.


## Response

```c
uint8_t DHT11_CheckSignal (void)
{
	uint8_t Response_Signal = 0;
	delay_us(40);										// wait for 40 microseconds
	if (!(HAL_GPIO_ReadPin(DHT11_PORT, DHT11_PIN))) {
		delay_us(80);									// delay for 80 microseconds
		if (HAL_GPIO_ReadPin(DHT11_PORT, DHT11_PIN)) {	// check if the pin is HIGH
			Response_Signal = 1;						// if so, the signal is 1 (present)
		} else {
			Response_Signal = -1;						// else, the signal is -1 (not present)
		}
	}
	while (HAL_GPIO_ReadPin(DHT11_PORT, DHT11_PIN));	// wait for pin to go LOW

	return Response_Signal;
}
```
- Delay for 40 microseconds.
- Check if DHT11_Pin (pin PA1) is low, then wait for another 80 microseconds. The pin should be high after that.
- Check if the pin is high, the signal is present and response is 1.


## Read Signal

```c
uint8_t DHT11_ReadSignal (void)
{
	uint8_t data_bit, count;

	for (count = 0; count < 8; count++) {
		while (!(HAL_GPIO_ReadPin(DHT11_PORT, DHT11_PIN))); // wait for the pin to go HIGH
		delay_us(40);										// delay for 40 microseconds

		if (!(HAL_GPIO_ReadPin(DHT11_PORT, DHT11_PIN))) {	// if the pin is low
			data_bit &= ~(1 << (7 - count));				// set the bit 0
		} else {
			data_bit |= (1 << (7 - count));					// else, set the bit 1
		}

		while (HAL_GPIO_ReadPin(DHT11_PORT, DHT11_PIN));	// wait for the pin to go LOW
	}
	return data_bit;
}
```
- Wait for the pin to go high.
- Delay for 40 microseconds (voltage-length of data "0" is 26-28 us).
- After 40 us, if the pin is low, set it to 0. Otherwise, set it to 1.


## The main function

```c
int main(void)
{
  /* Reset of all peripherals, Initializes the Flash interface and the Systick. */
  HAL_Init();
  /* Configure the system clock */
  SystemClock_Config();

  /* Initialize all configured peripherals */
  MX_GPIO_Init();
  MX_I2C1_Init();
  MX_TIM6_Init();
  
  HAL_TIM_Base_Start(&htim6);

  lcd_init();
  lcd_send_string("WELCOME TO STM32");
  HAL_Delay(3000);
  lcd_clear();

  while (1)
  {

	  // Data format in correct order: 8bit integral RH data + 8bit decimal RH data + 8bit integral T data
	  // + 8bit decimal T data + 8bit check sum.
	  MCU_StartSignal();
	  sensorData = DHT11_CheckSignal();
	  intRh_data = DHT11_ReadSignal();
	  decRh_data = DHT11_ReadSignal();
	  intTemp_data = DHT11_ReadSignal();
	  decTemp_data = DHT11_ReadSignal();
	  CHECKSUM = DHT11_ReadSignal();

	  TEMP = intTemp_data;
	  RH = intRh_data;

	  DHT11_Temperature = (float) TEMP;
	  DHT11_Humidity = (float) RH;

	  displayLCD_Temp(DHT11_Temperature);
	  displayLCD_Rh(DHT11_Humidity);

	  HAL_Delay(4000);

  }
}
```
- First, send the MCU Start Signal and then check for Response Signal.
- If the Reponse Signal is present, read the 40 bit data in order and store it in the integral Rh data, decimal Rh data, integral T data and decimal T data. 
- Combine the integral T and decimal T data together and the same for Rh data.
- Only the 1st byte of Temperature and Rh data are read.
- Display the Temp. and Rh results on the LCD.


## Displaying Temperature and Humidity on the LCD
These are functions for displaying Temp and Rh floats
```c
/* Function for displaying temperature on LCD */
void displayLCD_Temp (float temp)
{
	char str[20] = {0};
	lcd_put_cur(0, 0);

	sprintf(str, "TEMP: %.2f ", temp);
	lcd_send_string(str);
	lcd_send_data('C');
}

/* Function for displaying humidity on LCD */
void displayLCD_Rh (float rh)
{
	char str[20] = {0};
	lcd_put_cur(1, 0);

	sprintf(str, "HUMID: %.2f ", rh);
	lcd_send_string(str);
	lcd_send_data('%');
}
```


