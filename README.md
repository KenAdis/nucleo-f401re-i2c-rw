# nucleo-f401re-i2c-rw

Reads the ambient temperature from an **MCP9808** I2C temperature sensor with an **STM32 Nucleo-F401RE** board and prints it over the ST-LINK virtual COM port every 500 ms.

```
Temp: 23.50 C
Temp: 23.56 C
Temp: 23.50 C
```

## Hardware

| Part | Notes |
|---|---|
| NUCLEO-F401RE | STM32F401RET6, 84 MHz Cortex-M4F |
| MCP9808 breakout | ±0.25 °C typical accuracy, 0.0625 °C resolution |

### Wiring

| MCP9808 | Nucleo pin | Arduino header |
|---|---|---|
| VDD | 3V3 | 3V3 |
| GND | GND | GND |
| SCL | PB8 (I2C1_SCL) | D15 |
| SDA | PB9 (I2C1_SDA) | D14 |
| A0, A1, A2 | GND | — |

- With A0–A2 tied to GND, the sensor's 7-bit I2C address is `0x18`.
- I2C needs pull-up resistors on SCL and SDA. Most MCP9808 breakout boards include them. If yours does not, add 4.7 kΩ resistors from each line to 3V3.

## Peripheral configuration

| Peripheral | Setting |
|---|---|
| System clock | 84 MHz (HSI → PLL) |
| I2C1 | Standard mode, 100 kHz, PB8/PB9 |
| USART2 | 115200 baud, 8N1, PA2/PA3 (ST-LINK VCP) |

The configuration lives in [`nucleo-f401re-i2c-rw.ioc`](nucleo-f401re-i2c-rw.ioc) and can be opened in STM32CubeMX.

## How it works

The main loop in [`Core/Src/main.c`](Core/Src/main.c) does the following:

1. Writes the register pointer `0x05` (ambient temperature, T<sub>A</sub>) to the sensor.
2. Reads back 2 bytes.
3. Converts the raw value to °C.
4. Prints the result over USART2 and waits 500 ms.

If either I2C transfer fails, it prints `Error Tx` or `Error Rx` instead.

### T<sub>A</sub> register format

| Bits | Meaning |
|---|---|
| 15–13 | Alert flags (T<sub>CRIT</sub>, T<sub>UPPER</sub>, T<sub>LOWER</sub>) |
| 12 | Sign bit |
| 11–0 | Temperature, 1 LSB = 1/16 °C |

The conversion masks off the flags, divides the 12-bit magnitude by 16 and subtracts 256 if the sign bit is set. That treats the value as a 13-bit two's-complement number:

```c
temp_raw &= 0x1FFF;                       // drop alert flags
temp_c = (temp_raw & 0x0FFF) / 16.0f;     // 0.0625 °C per LSB
if (temp_raw & 0x1000) temp_c -= 256.0f;  // sign bit
```

> **Note:** The negative-temperature formula in the MCP9808 datasheet's example code (`256 - (...)`) is widely reported to give the wrong sign. The code above uses `value - 256`.

## Building

The project uses the CMake-based STM32Cube toolchain (STM32CubeIDE 2.x / STM32Cube for VS Code) with `arm-none-eabi-gcc`.

```sh
cmake --preset Debug
cmake --build --preset Debug
```

The output is `build/Debug/nucleo-f401re-i2c-rw.elf`.

## Flashing

Flash it from the IDE's Run/Debug launch, or with STM32CubeProgrammer:

```sh
STM32_Programmer_CLI -c port=SWD -w build/Debug/nucleo-f401re-i2c-rw.elf -rst
```

## Viewing the output

Open the ST-LINK virtual COM port at **115200 baud**:

```sh
# Linux
screen /dev/ttyACM0 115200
# or
picocom -b 115200 /dev/ttyACM0
```

On Windows, use the `STMicroelectronics STLink Virtual COM Port` (COMx) shown in Device Manager with PuTTY or another terminal program.

## References

- [MCP9808 datasheet (Microchip)](https://www.microchip.com/en-us/product/MCP9808)
- [NUCLEO-F401RE user manual (UM1724)](https://www.st.com/resource/en/user_manual/um1724-stm32-nucleo64-boards-mb1136-stmicroelectronics.pdf)
- [STM32F4 HAL documentation (UM1725)](https://www.st.com/resource/en/user_manual/um1725-description-of-stm32f4-hal-and-lowlayer-drivers-stmicroelectronics.pdf)
