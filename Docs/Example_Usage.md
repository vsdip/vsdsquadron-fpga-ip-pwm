# Example Usage : Breathing LED
Here, we'll integrate the IP within a RISC-V SoC to give a "breathing effect" to an LED by controlling its brightness through PWM signal.

### C Driver Macros
Make a header file `io.h` for defining the addresses to interact with the IP.

```c
#include <stdint.h>

#define IO_BASE      0x400000 // IO Base Address

// UART Offsets 
#define UART_DATA    0x10  // Data Register for UART communication
#define UART_CNTL    0x14  // Control Register for UART communication

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
	cd firmware
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
	You will be able to check the device utilisation on your terminal once your run the `make build` command.
	<img width="575" height="363" alt="device_utilisation" src="https://github.com/user-attachments/assets/9c9e5900-0602-4890-b7a4-d20bc34ba2d3" />
3. Make the physical connections and observe the output. Don't leave the RESET port to float, make sure to connect it appropriately otherwise board will assume it to be permanently pressed.
4. Observe the output received through UART on console.
   ```bash
   make terminal
   ```

**Expected Output:**
You will observe your LED undergoing a repetitive cycle of increasing and decreasing brightness, and text getting printed on the terminal as the output changes its state i.e., either polarity changes or PWM is disabled as shown in the video provided in [README.md](../README.md).

___
## Software Programming Model
Let's understand how the software program controls this IP so that you can write a custom program to utlize this IP.

### How Software Controls the IP
The PWM IP operates as a hardware accelerator. Instead of the CPU manually toggling a pin, which consumes significant processing power, the software delegates this task to the IP.
* **Memory-Mapped Interface:** The CPU interacts with the PWM block by writing to specific memory addresses. It treats the IP settings just like standard variables in memory.
* **Configuration Flow:** The software first defines the waveform shape by writing integer values for the Frequency (Period) and Pulse Width (Duty Cycle).
* **Autonomous Operation:** Once configured, the software sends a simple "Enable" command. The IP then takes over, running its internal counters and driving the physical output pin independently. This frees the CPU to perform other tasks while the signal is generated.
* **Dynamic Control:** The software can update the Duty Cycle during run-time (e.g., to dim an LED) without needing to stop or reset the IP.

### Typical Initialization Sequence
To generate a clean signal without glitches, follow this sequence:
1. **Define Pointers:** Define pointers for base address and individual register address provided in [Register Map Documentation](Register_Map.md). For example, `#define IO_BASE  0x400000`.
2. **Disable IP:** Write `0x0` to `PWM_CTRL` to ensure the block is stopped during setup.
3. **Set Period:** Write the desired total cycle count to `PWM_PERIOD` (e.g., `1000` for a 1kHz signal on a 1MHz clock).
4. **Set Duty Cycle:** Write the active time count to `PWM_DUTY` (e.g., `500` for 50% duty cycle).
5. **Enable IP:** Write `0x1` (Enable) or `0x3` (Enable + Inverted Polarity) to `PWM_CTRL` to start output generation.
6. **Update Duty Cycle:** Update value of `PWM_DUTY` during runtime to observe PWM.

### Polling vs. Status Checking
* **Status Checking:** Software can read the `PWM_STATUS` register to verify if the IP is currently enabled (Bit 0) or to read the live counter value (Bits [15:0]) for debugging.
* **No Polling Required:** Once enabled, the PWM IP runs autonomously. The software does not need to poll any register to keep the signal generating; it only needs to write new values if it wants to change the frequency or duty cycle dynamically.

