# System design

Flower Lab is a tabletop cabinet for one plant, about 40 cm wide and 70 cm tall. It keeps a light schedule, waters when the soil is dry, and doses nutrient concentrates only when an NPK reading is short.

## Roles

| Part | Role |
| --- | --- |
| Cabinet | Senses, decides, and acts. Keeps working if the network is down. |
| Orange Pi Zero 3 | The only computer in the cabinet. Runs the loop on Armbian. Does not power pumps or the LED bar. |
| Main server | Receives POSTs. Collects readings and doses. Aggregates history. Does not switch a pump. |
| Camera, later | CSI camera on the same board. A small YOLO model tracks leaf color, spots, and bloom. Posts events. Does not dose. |

The wet side and the dry side are separate compartments. Bottles, pumps, and tubing stay on the wet side. The Pi, the ADC, the PWM board, and the 12 V MOSFET board stay on the dry side.

## What the cabinet does

1. **Light.** Hours and brightness are a schedule. A PCA9685 channel dims the 12 V bar. If the bay is too warm, that channel backs off.
2. **Water.** A capacitive probe asks for a drink. A peristaltic pump runs for a few seconds, then the cabinet waits and reads again. It stops if the soil is wet, the tank float is open, or the day's cap is hit.
3. **Nutrients.** An RS485 probe reads nitrogen, phosphorus, and potassium a few times a day. If nitrogen is short, pump A pulses. If phosphorus or potassium is short, pump B pulses. If the reading is in range, neither pulses. A and B join the water line through check valves. They have no path of their own to the pot, so a nutrient pulse happens only while water is moving.
4. **Report.** Every reading and every dose is posted. The server stores them and keeps the aggregate.

## Boundaries

- Setpoints live on the cabinet. The server is a collector, not the controller.
- The control period is seconds for water and hours for nutrients. Soil and the NPK probe are slow. A tight nutrient loop will overdose.
- Linux timing is enough. Pump pulses are seconds long. Nothing is bit-banged.
- The header UART is left free. NPK uses USB–RS485, not the header UART.

## Failure

| Failure | Cabinet behavior |
| --- | --- |
| POST fails | Act anyway. Queue the payload and retry. |
| Float open | Water pump stays off. A and B stay off, because they only ride the water line. |
| Moisture read fails | Do not water. Post a fault. |
| NPK read fails | Do not dose A or B. Watering may continue. |
| Bay too hot | Light duty falls, including to off. |
| Pi reboots | Pumps must come up off. The 12 V rail is switched, not left enabled. |

## Later: vision

A camera connects to the Zero 3 on CSI. The USB port is already the RS485 adapter. YOLO runs on the board, a small model, a frame every few minutes. The H618 has no NPU, so this is not live video. Tracked classes are leaf color (yellowing, browning), spots, and bloom stage. Events are posted with the other readings. Vision never enables a pump.
