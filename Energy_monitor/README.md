# Energy Data Acquisition & Monitoring System (STM32-M4)

## Project Overview

This project is a high-precision **Energy Data Acquisition and Monitoring System** built on the **STM32F446RE (ARM Cortex-M4)** microcontroller. It implements a custom Digital Signal Processing (DSP) pipeline to measure partial electrical parameters such as **RMS Voltage, RMS Current, Active Power, Energy (kWh), Power Factor, and Frequency** in real-time.

The system utilizes a bare-metal architecture with no HAL/Standard Peripheral Libraries, ensuring maximum control over hardware resources and adhering to **MISRA-C** coding standards for reliability.

## Key Features

-   **Real-Time Data Acquisition**: High-speed sampling using ADC coupled with DMA.
-   **Custom DSP Algorithm**: RMS and Power calculations computed efficiently on the fpu-enabled Cortex-M4.
-   **Zero-Overhead Triggering**: Hardware timer (TIM2) triggers ADC conversions automatically without CPU intervention.
-   **Double-Buffering**: Continuous processing using DMA Half-Transfer and Complete-Transfer interrupts to ensure no data loss.
-   **User Interface**: 
    -   **OLED Display (SSD1306)** for live metrics.
    -   **UART Logging** for remote monitoring and debugging.

---

## Hardware Architecture

| Component | Description |
| :--- | :--- |
| **MCU** | STM32F446RE (ARM Cortex-M4F @ 180MHz) |
| **Sensors** | AC Voltage Sensor (ZMPT101B), AC Current Sensor (ACS712) |
| **Display** | 0.96" OLED (SSD1306) via I2C |
| **Debug** | UART Serial Console |

![System Schematic](schematic.png)


---

## Software Drivers

This project implements custom bare-metal drivers located in `src/` and `inc/`.

### 1. ADC & DMA Driver (`adc_dma_driver.h/.c`)
-   **Role**: Handles high-speed analog-to-digital conversion.
-   **Implementation**: Configures **ADC1** in Circular Scan Mode. **DMA2 Stream 0** is engaged to transfer conversion results directly to a memory buffer (`adc_buffer`).
-   **Trigger source**: External trigger from **TIM2 TRGO** ensure precise sampling timing (jitter-free).

### 2. Timer Driver (`timer_driver.h/.c`)
-   **Role**: Provides the timebase for data acquisition.
-   **Implementation**: Configures **TIM2** to generate a Trigger Output (TRGO) event at exactly **8000 Hz**. This defines the sampling rate ($F_s$) of the system.

### 3. I2C Driver (`i2c_driver.h/.c`)
-   **Role**: Communication link for the OLED display.
-   **Implementation**: Bare-metal manipulation of the **I2C1** peripheral. Supports Master Transmit mode with standard speed (100kHz) or fast mode. Handles start/stop generation and address transmission manually.

### 4. UART Driver (`uart_driver.h/.c`)
-   **Role**: Data logging and debug interface.
-   **Implementation**: Configures **USART2** (connected to ST-Link Virtual COM port) for TX/RX. Used to stream measurement data to a PC terminal.

### 5. SSD1306 Driver (`ssd1306.h/.c`)
-   **Role**: Graphics controller for the OLED.
-   **Implementation**: Application-layer driver that builds on top of the I2C driver. Manages a frame buffer in RAM and handles text rendering commands.

---

## Core Application Logic: `energy_meter.c`

The heart of the application resides in `energy_meter.c`. This module handles the DSP algorithm and state management.

### Data Flow Pipeline

1.  **Sampling**: `TIM2` triggers the `ADC` 8000 times per second. `DMA` moves samples into `adc_buffer[]`.
2.  **Interrupt Handling**: The buffer is split into two halves (Double Buffering).
    -   When the first half is full (Half-Transfer Interrupt), the CPU processes the first half.
    -   When the second half is full (Transfer-Complete Interrupt), the CPU processes the second half.
    -   This allows simultaneous sampling and processing.

### DSP Algorithm Details

The `Accumulate_Data` function iterates through raw ADC values for Voltage ($V$) and Current ($I$):

1.  **Offset Removal**:
    -   Raw ADC values (0-4095) are centered by subtracting calibrated **DC Offsets** (`V_OFFSET`, `I_OFFSET`).

2.  **Instantaneous Calculation**:
    -   **$V^2$**, **$I^2$**: Squared values accumulated for RMS calculation.
    -   **$P_{inst}$**: Instantaneous Power ($V \times I$) accumulated for Active Power calculation.

3.  **Frequency Detection (Zero-Crossing)**:
    -   Tracks the sign of the voltage signal.
    -   Counts transitions from negative to positive (or vice-versa) to determine signal frequency.
    -   Includes a hysteresis threshold (`ZERO_CROSS_THRES`) to reject noise around the zero point.

4.  **1-Second Aggregation**:
    -   After 8000 samples (1 second), the final metrics are computed:
        -   **RMS Voltage ($V_{rms}$)**: $\sqrt{\frac{\sum V^2}{N}} \times CalibrationFactor$
        -   **RMS Current ($I_{rms}$)**: $\sqrt{\frac{\sum I^2}{N}} \times CalibrationFactor$
        -   **Active Power ($P$)**: $\frac{\sum (V \times I)}{N}$
        -   **Apparent Power ($S$)**: $V_{rms} \times I_{rms}$
        -   **Power Factor ($PF$)**: $\frac{P}{S} \times 100\%$
        -   **Energy ($E$)**: Accumulated $P$ over time ($\text{kWh}$).

5.  **Output**:
    -   The results are formatted and sent to the **OLED** and **UART** every second.

![Energy Aggregation](energy_aggregation.png)


## Build & Run

1.  Import project into STM32CubeIDE or preferred toolchain.
2.  Ensure source path includes `inc/` and `src/`.
3.  Build and Flash to STM32F446RE.
4.  Connect sensors to assigned Analog pins (PA0/PA1 typical).
5.  View output on OLED or Serial Terminal (115200 baud).

![OLED Initialization](oled_init.png)

