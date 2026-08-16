# STM32-HAL-LoRa-based-MPU6050-DHT11-and-MQ-135-data-transmitter

# STM32F411 Multi-Sensor IoT Telemetry Node over LoRa (SX1278)

A multi-sensor IoT telemetry node built around the **STM32F411VETx** (ARM Cortex-M4) microcontroller. This project collects real-time environmental data (temperature, humidity, air quality) and motion dynamics (roll, pitch, yaw) and transmits telemetry wirelessly over long distances using an **SX1278 LoRa** transceiver.

---

## Technical Overview

* **Microcontroller**: STM32F411VETx (ARM Cortex-M4 with FPU) running on STM32CubeIDE and HAL drivers.


* **System Clock & Core**: Configured using an external High-Speed Crystal (HSE) driven via internal PLL.


* **Motion Tracking**: MPU6050 6-axis IMU interfaced over I2C1 calculating filtered orientation angles.


* **Environmental Sensing**: DHT11 relative humidity and temperature sensor sampled via single-wire dynamic open-drain GPIO.


* **Air Quality Monitoring**: MQ-135 gas sensor digitized using 12-bit ADC Channel 0.


* **Long-Range Wireless**: SX1278 433 MHz LoRa module interfaced via SPI1.


* **Debug Interface**: Redirected `printf` output over USART2 for real-time console logging.



---

## Hardware Pinout & Peripherals

| Peripheral / Module | Pin / Signal | STM32 Pin | Function / Configuration |
| --- | --- | --- | --- |
| **MPU6050 IMU**<br> | SCL | `PB6` | I2C1 Clock (400 kHz Fast Mode)

 |
|  | SDA | `PB7` | I2C1 Data

 |
| **SX1278 LoRa**<br> | SCK | `PA5` | SPI1 Clock

 |
|  | MISO | `PA6` | SPI1 Master In Slave Out

 |
|  | MOSI | `PA7` | SPI1 Master Out Slave In

 |
|  | NSS (CS) | `PA4` | SPI Chip Select Output

 |
|  | RESET | `PB0` | GPIO Reset Control Output

 |
|  | DIO0 | `PC5` | GPIO Interrupt Input (RxDone / TxDone)

 |
| **MQ-135 Sensor**<br> | Analog Out | `PA0` | ADC1 Channel 0 (12-bit conversion)

 |
| **DHT11 Sensor**<br> | Data | Configured GPIO | Dynamic Output Open-Drain / Input

 |
| **USART2 Debug**<br> | TX / RX | USART2 Pins | Retargeted `printf` streaming

 |

---

## Firmware Architecture & Driver Details

### 1. Motion Tracking & Angle Calculations (`MPU6050` & `CalculateAngle`)

* **I2C Initialization**: MPU6050 device presence verified by reading the `WHO_AM_I` register (`0x68`).


* **Sampling Rate**: Set to 200 Hz (`SMPRT_DIV = 39`) with DLPF configured.


* **Complementary Filter**: Accelerometer angles (Roll and Pitch) are fused with Integrated Gyroscope angular velocities using a complementary filter weighting ($\alpha = 0.96$) to reject high-frequency gyro drift and low-frequency accelerometer vibrations:



$$\text{Filter Roll} = 0.96 \times (\text{Gyro}_y \cdot dt + \text{Roll}_{\text{prev}}) + 0.04 \times \text{Acc Roll}$$



### 2. Environmental & Gas Sensing (`DHT` & `ADC1`)

* **DHT11 Driver**: Implements single-wire protocol by dynamically toggling the GPIO pin mode between `GPIO_MODE_OUTPUT_OD` and `GPIO_MODE_INPUT` to perform precision start signals and read responses.


* **MQ-135 Sensing**: ADC1 is configured in single conversion software-triggered mode with 12-bit resolution (`ADC_RESOLUTION_12B`) on `PA0` (`ADC_CHANNEL_0`).



### 3. LoRa Telemetry Stack (`SX1278`)

* **SPI Communication**: Uses SPI1 in Master Mode with software/hardware slave management.


* **LoRa Settings**:


* **Carrier Frequency**: 433 MHz


* **Spreading Factor**: SF7


* **Bandwidth**: 125 kHz


* **Coding Rate**: 4/5


* **TX Power**: +20 dBm


* **Over-Current Protection (OCP)**: 120 mA


* **Preamble Length**: 10 Symbols





---

## Main Program Execution Flow

1. **System Core Init**: Configures clock system, HAL delay, GPIO, SPI1, I2C1, USART2, and ADC1 peripherals.


2. **Sensor Setup**: Wakes up MPU6050, configures register scale parameters, and verifies connection.


3. **LoRa Initialization**: Resets SX1278 transceiver, configures modem parameters, and enters standby/receive mode.


4. **Telemetry Loop**:


* Polls MQ-135 raw air quality value via ADC1.


* Checks for MPU6050 data readiness, converts raw 6-axis data, and computes filtered angles.


* Reads raw bytes from DHT11 sensor and validates checksum.


* Formats telemetry packet payload incorporating Node Address ID (`0x3B`), temperature, humidity, filtered roll/pitch/yaw angles, and gas metric.


* Transmits frame via LoRa over SPI and delays for next transmission interval.





---

## Project Structure

```text
├── Drivers/
│   └── STM32F4xx_HAL_Driver/
├── Core/
│   ├── Inc/
│   │   ├── adc.h
│   │   ├── gpio.h
│   │   ├── i2c.h
│   │   ├── spi.h
│   │   ├── DHT.h
│   │   ├── MPU6050.h
│   │   ├── CalculateAngle.h
│   │   └── LoRa.h
│   └── Src/
│       ├── main.c              # Application entry point & main processing loop
│       ├── adc.c               # ADC1 configuration & pin init
│       ├── gpio.c              # System GPIO init
│       ├── i2c.c               # I2C1 configuration
│       ├── spi.c               # SPI1 configuration
│       ├── DHT.c               # DHT11 custom bit-banging driver
│       ├── MPU6050.c           # MPU6050 register read/write & raw data acquisition
│       ├── CalculateAngle.c    # IMU angle fusion algorithms
│       ├── LoRa.c              # SX1278 packet transmission & register configuration
│       └── stm32f4xx_hal_msp.c # Low-level HAL MSP initialization
└── KiCad/                      # Custom PCB schematic and footprint design project files

```

---

## How to Build and Flash

1. **Prerequisites**:
* STM32CubeIDE (v1.10.0 or later recommended).
* ST-Link V2/V3 programmer/debugger.


2. **Importing Project**:
* Open STM32CubeIDE $\rightarrow$ `File` $\rightarrow$ `Import` $\rightarrow$ `Existing Projects into Workspace`.
* Select the repository root folder.


3. **Compilation**:
* Select `Build Configuration` $\rightarrow$ `Release` or `Debug`.
* Click `Build` (`Ctrl + B`).


4. **Flashing**:
* Connect ST-Link to target board (`SWDIO`, `SWCLK`, `GND`, `3V3`).
* Run/Debug project (`F11`) to flash memory.
