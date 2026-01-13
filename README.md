# vsdsquadron-fpga-ip-pwm

## 1. IP Overview

### Purpose
The PWM (Pulse Width Modulation) IP is a single-channel, memory-mapped peripheral designed for VSDSquadron SoC. It serves as a timing generator that produces a square wave with variable frequency and duty cycle, suitable for controlling motors, LEDs, or generating audio tones.

### Why use this IP?
You should use this IP when you need precise, hardware-timed signals without burdening the CPU. Unlike software-based loops, this IP runs independently once configured, allowing the processor to perform other tasks while the signal is generated.

### Typical Use Cases
* **LED Dimming:** Varying the brightness of an LED by changing the duty cycle (e.g., "Breathing" LED effect).
* **Motor Control:** Controlling the speed of a DC motor.
* **Audio Generation:** Generating simple square-wave tones (beeps) by adjusting the period (frequency).
___

## 2. Block Diagram
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

## 3. Register Map

**Base Address:** User Configurable (Default in this repo: `0x400000`)

| Offset | Name       | Type | Default Value | Description                          |
| :----: | :--------: | :--: | :-----------: | :------------------------------------|
| `0x20` | PWM_CTRL   | R/W  | `0x00`        | Control Register (Enable & Polarity) |
| `0x24` | PWM_PERIOD | R/W  | `0x01`        | Period Register (Total Ticks)        |
| `0x28` | PWM_DUTY   | R/W  | `0x00`        | Duty Cycle Register (Active Ticks)   |
| `0x2C` | PWM_STATUS | R    | `0x00`        | Status Register (Counter & Running)  |

### 1. PWM_CTRL (Offset `0x20`)
Controls the main operation of the PWM block.
* **Bit 0 (EN):** Enable PWM. `1` = Run, `0` = Stop (Output forced to inactive state).
* **Bit 1 (POL):** Polarity.
   * `0`: Active High (Default).
   * `1`: Active Low (Inverts output logic).
* **Bits [31:2]:** Reserved.

### 2. PWM_PERIOD (Offset `0x24`)
Defines the frequency of the PWM signal.
* **Bits [31:0]:** Number of clock cycles for one full PWM period.
* *Note:* `Frequency = System_Clock / Register_Value`.

### 3. PWM_DUTY (Offset `0x28`)
Defines the pulse width.
* **Bits [31:0]:** Number of clock cycles the signal remains active.
* *Note:* Ensure `PWM_DUTY <= PWM_PERIOD`.

### 4. PWM_STATUS (Offset `0x2C`)
Read-only status for debugging.
* **Bit 0:** Run Status (Mirror of `PWM_CTRL` Enable bit).
* **Bits [31:16]:** Current internal 16-bit counter value (Lower bits of 32-bit counter).
___

## 4. Feature Summary

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

## 5. Integration Guide

### 1. RTL Instantiation
Add `pwm_ip.v` to your project source list. Instantiate the module in your top-level SoC wrapper (e.g., `riscv.v`) as follows:

```verilog
pwm_ip u_pwm (
    .clk(clk),
    .resetn(resetn),
    .i_sel(pwm_select),       // Active HIGH Select Signal
    .i_we(mem_write_enable),  // Write Enable from CPU
    .i_addr(mem_addr[3:0]),   // Address Offset
    .i_wdata(mem_wdata),      // Write Data from CPU
    .o_rdata(pwm_rdata),      // Read Data to CPU
    .pwm_out(pwm_out)         // Connect to IO Pad
);
```

### 2. Address Decoding
Generate the `pwm_select` signal based on your SoC's memory map.
* **Example Logic:**
	```verilog
	wire isIO = mem_addr[22]; // Base Address: 0x400000
	wire pwm_select = isIO & (mem_addr[7:4] == 4'h2); // Base Offset: 0x20
	```

### 3. FPGA Constraints Setup
Map the `pwm_out` port to a physical pin in your `.pcf` file.
* **Example Mapping:**
	```text
	set_io pwm_out 28  # Example Pin
	```

### Device Utilisation
<img width="575" height="363" alt="device_utilisation" src="https://github.com/user-attachments/assets/9c9e5900-0602-4890-b7a4-d20bc34ba2d3" />

___

## 6. Software Programming Model

### How Software Controls the IP
The PWM IP operates as a hardware accelerator. Instead of the CPU manually toggling a pin, which consumes significant processing power, the software delegates this task to the IP.
* **Memory-Mapped Interface:** The CPU interacts with the PWM block by writing to specific memory addresses. It treats the IP settings just like standard variables in memory.
* **Configuration Flow:** The software first defines the waveform shape by writing integer values for the Frequency (Period) and Pulse Width (Duty Cycle).
* **Autonomous Operation:** Once configured, the software sends a simple "Enable" command. The IP then takes over, running its internal counters and driving the physical output pin independently. This frees the CPU to perform other tasks while the signal is generated.
* **Dynamic Control:** The software can update the Duty Cycle on-the-fly (e.g., to dim an LED) without needing to stop or reset the IP.

### Typical Initialization Sequence
To generate a clean signal without glitches, follow this sequence:
1. **Define Base Address:** Ensure your pointer matches the hardware base address (e.g., `0x400000`).
2. **Disable IP:** Write `0x0` to `PWM_CTRL` to ensure the block is stopped during setup.
3. **Set Period:** Write the desired total cycle count to `PWM_PERIOD` (e.g., `1000` for a 1kHz signal on a 1MHz clock).
4. **Set Duty Cycle:** Write the active time count to `PWM_DUTY` (e.g., `500` for 50% duty cycle).
5. **Enable IP:** Write `0x1` (Enable) or `0x3` (Enable + Inverted Polarity) to `PWM_CTRL` to start output generation.

### Polling vs. Status Checking
* **Status Checking:** Software can read the `PWM_STATUS` register to verify if the IP is currently enabled (Bit 0) or to read the live counter value (Bits [15:0]) for debugging.
* **No Polling Required:** Once enabled, the PWM IP runs autonomously. The software does not need to poll any register to keep the signal generating; it only needs to write new values if it wants to change the frequency or duty cycle dynamically.
___

## 7. Example Usage : Breathing LED
Here, we'll integrate the IP within a RISC-V SoC to give a "breathing effect" to an LED by controlling its brightness through PWM signal.

### C Driver Macros
Include these definitions in your `io.h` or main file to interact with the IP.

```c
#include <stdint.h>

#define IO_BASE      0x400000 // IO Base Address

// PWM  Offsets
#define PWM_CTRL     0x20
#define PWM_PERIOD   0x24
#define PWM_DUTY     0x28
#define PWM_STATUS   0x2C

// Access Macros
#define IO_IN(offset)     (*(volatile uint32_t*)(IO_BASE + (offset)))
#define IO_OUT(offset,val) (*(volatile uint32_t*)(IO_BASE + (offset)) = (val))
```

### Program Source Code

```c
#include "io.h"
#include "uart.h" // Custom UART Header File

// Delay function
void delay(int cycles) {
    for (volatile int i = 0; i < cycles; i++);
}

void main() {
    uprint("\n\t\t\t\t--- PWM IP Test ---\n");    

    // Configure PWM
    int polarity = 0;                          // Polarity = 0 (Active HIGH)
    const char *mode = (polarity == 1) ? "Active LOW" : "Active HIGH";
    IO_OUT(PWM_PERIOD, 1000);                  // Set Period to 1000 ticks
    IO_OUT(PWM_CTRL, polarity * 2 + 1);        // Enable = 1 
    uprint("\nConfiguring PWM: Period=%d, Mode=%s\n", 1000, mode);

    uint32_t status = IO_IN(PWM_STATUS);
    uprint("PWM Enabled. Current Status: 0x%x\n", status);

    // Breathe LED effect
    int duty = 0;
    int direction = 1;
    int count = 0;

    jump:
    while (count < 3) { // Breathe 3 times 
        // Update Duty Cycle
        IO_OUT(PWM_DUTY, duty);
        delay(10000); // Wait a bit so changes are visible

        if (direction) {
            duty += 10;
            if (duty >= 1000) direction = 0;
        } else {
            duty -= 10;
            if (duty <= 0) { direction = 1; count++; }
        }
    }
    
    // Disable PWM
    IO_OUT(PWM_CTRL, 0);
    status = IO_IN(PWM_STATUS);
    uprint("\nPWM Disabled. Current Status: 0x%x\n", status);
    delay(500000); // Wait for few seconds to observe disability

    // Invert the polarity
    count = 0;
    polarity = (polarity + 1) % 2;
    mode = (polarity == 1) ? "Active LOW" : "Active HIGH";
    IO_OUT(PWM_CTRL, polarity * 2 + 1);
    status = IO_IN(PWM_STATUS);
    uprint("\nPolarity Inverted. Current Status: 0x%x. Mode=%s\n", status, mode);

    goto jump;    // Repeat the same process  
}
```

### RTL Simulation

1. Reduce the delay in software application `pwm_test.c` to speed up the simulation.
2. Convert `pwm_test.c` to a `.hex` file.
	```bash
	cd ./firmware
 	make pwm_test.bram.hex
 	```
3. Simulate the SoC.
   	```bash
	cd ../rtl
	iverilog -D BENCH -o pwm_test tb.v riscv.v sim_cells.v 
	vvp pwm_test
    ```

    **Expected Output:**
   	<img width="974" height="360" alt="pwm_output" src="https://github.com/user-attachments/assets/6e31e79f-ef17-448b-9a36-2f3dba00741b" />


4. Observe the waveform.
   ```bash
   gtkwave test.vcd
   ```

	<img width="1582" height="191" alt="pwm_waveform" src="https://github.com/user-attachments/assets/1d3be862-1740-4f86-a75b-0ec53dd11b66" />


### Hardware Validation

1. Turn back delay in software application `pwm_test.c` to original values and rewrite the `pwm_test.bram.hex` file. This delay facilitates visible changes over the hardware.
2. Perform the Synthesis & Flash through `Yosys (Synth) → Nextpnr (Place & Route) → Icepack (Bitstream)`. The commands for which are written in the `Makefile` in `rtl` directory.
   ```bash
   make build
   make flash
   ```
3. Make the physical connections and observe the output. Don't leave the RESET port to float, make sure to connect it appropriately otherwise board will assume it to be permanently pressed.
4. Observe the output received through UART on console.
   ```bash
   make terminal
   ```

	<video src="[user-images.githubusercontent.com](https://github.com/user-attachments/assets/b0f1e7a2-ad49-4e74-a0b0-3cb12110af3a)" controls width="500">![Demo video](https://github.com/user-attachments/assets/b0f1e7a2-ad49-4e74-a0b0-3cb12110af3a)</video>


https://github.com/user-attachments/assets/b0f1e7a2-ad49-4e74-a0b0-3cb12110af3a


