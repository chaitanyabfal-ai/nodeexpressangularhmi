# iPTran schematic widget - revision history

Each revision is kept as a separate widget so every stage stays placeable
from the FUXA editor Lib Widgets panel (placed widgets freeze their SVG
content at placement time; views are never modified retroactively).

| File | Revision | Exported vars | Notes |
|---|---|---|---|
| iptran_schematic_v1.svg | v1 | 4 (_ps_ pressures) | static mock-up of station schematic |
| iptran_schematic_v2.svg | v2 | 15 | flows/level, pump states, comm dots, alarm-range coloring, click-to-command |
| iptran_schematic_v3.svg | v3 | 15 (same names) | adds animated pipeline flow, alarm banner, sparkline trends, live clock, stale-data indicator, comm dimming, tooltips |
| iptran_schematic.svg | latest | - | alias of the newest revision (v3) |

To release a new revision: copy the latest file to iptran_schematic_vN.svg,
edit it, then refresh the alias (cp iptran_schematic_vN.svg iptran_schematic.svg)
in BOTH server/_widgets/dynamicSVG/ and the Docker widgets/ mount.
