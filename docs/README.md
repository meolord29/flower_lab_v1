# Flower Lab documentation

The cabinet is one pot. An Orange Pi Zero 3 runs the loop and posts readings to the main server. The server collects and aggregates. It does not drive the pumps.

The product page is [concept.html](../concept.html). These notes are the longer form of the same design.

| Module | What it fixes |
| --- | --- |
| [System design](system-design.md) | What the cabinet is, what runs where, what happens when the network fails |
| [Data models](data-models.md) | The records the cabinet emits and the server stores |
| [API design](api-design.md) | The POST contract between the cabinet and the main server |
| [Hardware control](hardware-control.md) | Buses, pumps, probes, and the control loops |

A CSI camera and a small YOLO model are specified as later work. They track leaf color and bloom. They do not dose.
