# PWM IP Register Map

**Base Address:** User Configurable (Default in this repo: `0x400000`)

| Offset | Name       | Type | Reset Value   | Description                          |
| :----: | :--------: | :--: | :-----------: | :------------------------------------|
| `0x20` | PWM_CTRL   | R/W  | `0x00`        | Control Register (Enable & Polarity) |
| `0x24` | PWM_PERIOD | R/W  | `0x01`        | Period Register (Total Ticks)        |
| `0x28` | PWM_DUTY   | R/W  | `0x00`        | Duty Cycle Register (Active Ticks)   |
| `0x2C` | PWM_STATUS | R    | `0x00`        | Status Register (Counter & Running)  |
___
## Register Details

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
## Behavior for edge cases
1. `DUTY = 0` : Output will always be inactive i.e., LOW (for Polarity = 0) & HIGH (for Polarity = 1)
2. `DUTY >= PERIOD` : Output will always be active i.e., HIGH (for Polarity = 0) & LOW (for Polarity = 1)
3. `CTRL = 0` : Output will be LOW as PWM is disabled and default polarity is 0.
