# Integration Guide

To integrate this IP in your SoC, follow the below given steps:

### 1. RTL Instantiation
Add `pwm_ip.v` to your project source list, and instantiate the module in your top-level SoC wrapper (e.g., `riscv.v`) as follows:

```verilog
pwm_ip u_pwm (
    .clk(clk),
    .resetn(resetn),
    .i_sel(pwm_select),       // Active HIGH Select Signal
    .i_we(mem_write_enable),  // Write Enable from CPU
    .i_addr(mem_addr[3:0]),   // Address Offset
    .i_wdata(mem_wdata),      // Write Data from CPU
    .o_rdata(pwm_rdata),      // Read Data to CPU
    .pwm_out(pwm_pin)         // Connect to IO Pad
);
```

### 2. Address Decoding
Generate the `pwm_select` signal based on your SoC's memory map inside your `SOC` module.
* **Example Logic:**
	```verilog
	wire isIO = mem_addr[22]; // Base Address: 0x400000
	wire pwm_select = isIO & (mem_addr[7:4] == 4'h2); // Base Offset: 0x20
	```

### 3. VSDSquadron FPGA Board Mapping
Map the `pwm_out` port to a physical pin in your `.pcf` file.
* **Example Mapping:**
	```text
	set_io pwm_pin 28  # Example Pin
	```

Now follow the steps provided in [Example Usage Documentation](Example_Usage.md) to run a software program on your SoC equipped with this PWM IP.
