# vsdsquadron-fpga-ip-pwm

## IP Overview

### Purpose
The PWM (Pulse Width Modulation) IP is a single-channel, memory-mapped peripheral designed for VSDSquadron SoC. It serves as a timing generator that produces a square wave with variable frequency and duty cycle, suitable for controlling motors, LEDs, or generating audio tones.

### Why use this IP?
You should use this IP when you need precise, hardware-timed signals without burdening the CPU. Unlike software-based loops, this IP runs independently once configured, allowing the processor to perform other tasks while the signal is generated.

### Typical Use Cases
* **LED Dimming:** Varying the brightness of an LED by changing the duty cycle (e.g., "Breathing" LED effect).
* **Motor Control:** Controlling the speed of a DC motor.
* **Audio Generation:** Generating simple square-wave tones (beeps) by adjusting the period (frequency).
___

## Block Diagram
```text
       +-----------------------------------------------------------+
       |                       VSDSquadron SoC                     |
       |                                                           |
       |   +-------------------+             +-----------------+   |
       |   |                   |             |                 |   |
       |   |      RISC-V       |             |    SYSTEM CLK   |   |
       |   |       CORE        |             |      & RESET    |   |
       |   |                   |             |                 |   |
       |   +---------+---------+             +--------+--------+   |
       |             |                                |            |
       |             |            System Bus          |            |
       |   <---------+--------------------------------+-------->   |
       |             |      (Addr, Data, WE, SEL)     |            |
       |             v                                v            |
       |   +---------------------------------------------------+   |
       |   |                     PWM IP BLOCK                  |   |
       |   |  +---------------------------------------------+  |   |
       |   |  |        Bus Interface & Register Map         |  |   |
       |   |  | (Decodes Addr: 0x...20, 0x...24, 0x...28)   |  |   |
       |   |  +---------+--------------+--------------+-----+  |   |
       |   |            |              |              |        |   |
       |   |            v              v              v        |   |
       |   |  +---------------+  +------------+  +---------+   |   |
       |   |  |    CTRL REG   |  | PERIOD REG |  | DUTY REG|   |   |
       |   |  | (En+Polarity) |  | (Freq Set) |  | (Width) |   |   |
       |   |  +---------+-----+  +-----+------+  +----+----+   |   |
       |   |            |              |              |        |   |
       |   |            |              v              v        |   |
       |   |            |        +--------------------------+  |   |
       |   |            +------->|   Counter & Comparator   |  |   |
       |   |                     |           Logic          |  |   |
       |   |                     +-------------+------------+  |   |
       |   |                                   |               |   |
       |   +-----------------------------------+---------------+   |
       |                                       |                   |
       +---------------------------------------+-------------------+
                                               |
                                         +-----v-----+
                                         |  PWM_OUT  |------> (LED / Motor)
                                         +-----------+
```
___

## Register Map

**Base Address:** User Configurable (Default in this repo: `0x400000`)

| Offset | Name       | Type | Reset Value   | Description                          |
| :----: | :--------: | :--: | :-----------: | :------------------------------------|
| `0x20` | PWM_CTRL   | R/W  | `0x00`        | Control Register (Enable & Polarity) |
| `0x24` | PWM_PERIOD | R/W  | `0x01`        | Period Register (Total Ticks)        |
| `0x28` | PWM_DUTY   | R/W  | `0x00`        | Duty Cycle Register (Active Ticks)   |
| `0x2C` | PWM_STATUS | R    | `0x00`        | Status Register (Counter & Running)  |

View the detailed Register Map documentation [here](Docs/Register_Map.md).
___

## Feature Summary

### Key Features
* **Configurable Period:** 32-bit register defines the total signal frequency.
* **Configurable Duty Cycle:** 32-bit register defines the "Active" pulse width.
* **Polarity Control:** Supports both Active High (standard) and Active Low (inverted) output logic.
* **Interface:** 32-bit Memory-Mapped (APB-style) bus interface.

### Clock Assumptions
* **Domain:** Fully synchronous to the system bus clock (`clk`).
* **Frequency:** The output frequency is a direct division of the system clock.
* *Formula:* `Output_Freq = System_Clock_Freq / PERIOD_Reg_Value`

### Limitations
* **Single Channel:** This IP controls only one physical output pin.
* **Counter Reset on Update:** If the `PERIOD` is updated to a value smaller than the current internal counter, the counter resets to 0 immediately to prevent undefined behavior.
* **No Prescaler:** The counter always runs at the system clock rate, so very low frequencies may require large period values.
___

## Getting Started

To test this IP, you may clone this repository and follow below steps:
1. Connect an LED to your VSDSquadron FPGA at pin `28` and a push button at pin `10` which will act as a RESET button.
2. Make a `.hex` file from the provided software application `firmware/pwm_test.c` by using the commands -
	```bash
	cd firmware
	make pwm_test.bram.hex
	```
3. Build the bitstream for programming the FPGA.
	```bash
	cd ../rtl
	make build
	```
4. Flash the program into your VSDSquadron FPGA.
	```bash
	make flash
	```
	You will observe the brightness of LED being increased and decreased in a cyclic manner.
5. To observe messages on your terminal, connect a USB-to-UART module with `RXD` at pin `18` and type the below command -
	```bash
	make terminal
	```

You can see what output to expect in the given video -
<video src="[user-images.githubusercontent.com](https://github.com/user-attachments/assets/b0f1e7a2-ad49-4e74-a0b0-3cb12110af3a)" controls width="500">![Demo video](https://github.com/user-attachments/assets/b0f1e7a2-ad49-4e74-a0b0-3cb12110af3a)</video>

https://github.com/user-attachments/assets/b0f1e7a2-ad49-4e74-a0b0-3cb12110af3a

To integrate this IP in your SoC or write a custom software program to utilize the IP, you can refer to [Integration Guide](Docs/Integration_Guide.md) and [Example Usage](Example_Usage.md) respectively.

