# Hardware control

The Orange Pi Zero 3 decides. A separate 12 V supply feeds the LED bar and the three pumps through a MOSFET board. 12 V never touches a Pi pin. Ground is common.

GPIO numbers are not assigned here. The Zero 3 header is not a Raspberry Pi header. Pins are chosen when the wiring map is confirmed. Buses below are the contract.

## Map

| Function | Connection | Notes |
| --- | --- | --- |
| LED dim | PCA9685 channel 0 → MOSFET → 12 V bar | Duty 0–1. Off is duty 0 |
| Water pump | PCA9685 channel 1 → MOSFET | 12 V peristaltic, 80–150 ml/min |
| Concentrate A | PCA9685 channel 2 → MOSFET | 12 V peristaltic, 10–30 ml/min |
| Concentrate B | PCA9685 channel 3 → MOSFET | 12 V peristaltic, 10–30 ml/min |
| PWM and ADC | I2C | SDA, SCL, 3.3 V, GND. PCA9685 and ADS1115 on the same bus |
| Moisture | Capacitive probe → ADS1115 analog | Not a resistive probe |
| NPK | RS485 Modbus → USB–RS485 → USB | Header UART stays free |
| Tank float | Dry contact → GPIO | Open means empty |
| Bay temperature | Sensor → GPIO | Used to back off the light |
| Pi power | 5 V supply | Not the 12 V rail |
| Camera, later | CSI | USB is the RS485 adapter |

On boot, all four MOSFET channels are off before the loop starts. A crashed process must not leave a pump enabled. The MOSFET inputs need a default-off resistor.

## Fluid path

Water, A, and B each have a pump. A and B pass a check valve and tee into the water line. The single line ends at the drip ring. Concentrate cannot siphon back, and it cannot reach the pot except in that line.

A nutrient pulse runs only while the water pump is on, so the line carries it up. A or B never run alone.

## Light loop

Each minute:

1. If local time is outside `light_on`–`light_off`, set channel 0 to 0.
2. If bay temperature is at or above `temp_limit_c`, set channel 0 to 0.
3. Otherwise set channel 0 to `light_duty`.
4. Post a light reading. Post a dose only when duty changes.

## Water loop

Each cycle, after `water_wait_s`:

1. Read the ADS1115. If the read fails, post a fault and do not pulse.
2. Read the float. If the tank is empty, hold water, A, and B. Post the float reading.
3. If raw moisture is above `moisture_high`, do nothing.
4. If raw moisture is below `moisture_low` and today's water-on time is under `water_cap_s`, pulse channel 1 for `water_pulse_s`.
5. Post the dose with reason `moisture_low`, then wait, then read again.

One pulse per cycle. No second pulse until the wait has elapsed.

## Nutrient loop

Every `npk_period_h`:

1. Read N, P, and K over Modbus. If the read fails, post a fault and do not dose. The register map is probe-specific and is not fixed in this document.
2. Post the NPK reading.
3. If a concentrate pulse happened within `nutrient_cooldown_h`, stop.
4. If N is below `n_low`, arm pump A for the next water pulse (`n_short`).
5. If P is below `p_low` or K is below `k_low`, arm pump B (`p_short` or `k_short`).
6. If nothing is short, arm nothing.

The armed pulse is `a_pulse_s` or `b_pulse_s`, and it starts only after channel 1 is already on. It ends before channel 1 ends, so water flushes the line. Both may run in the same drink if both are short. Neither runs if the float is open or the water pulse was refused.

## Vision loop, later

A frame every few minutes. A small YOLO model on the CPU. Emit one reading per detection above a confidence floor, with labels `leaf_green`, `leaf_yellow`, `leaf_brown`, `spots`, `bloom`, `bud`. Post them. Do not read this loop from the water or nutrient code.

## Process

Python on Armbian. One process owns I2C, GPIO, and the USB serial port. The sender is a queue in that process, or a second process reading the same queue. The sender never writes a PWM register.

Libraries should talk to `/dev/i2c` and `/dev/ttyUSB` (or the CSI device, later). Do not depend on a Raspberry Pi GPIO library. The Zero 3 is not that board.
