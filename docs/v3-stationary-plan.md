# CORVIN v3 — stationary rack-swap robot (mini version)

**Why v3:** budget and container limits rule out the v2 plan (a mobile robot driving the aisles of a server library). v3 keeps the same software stack and UI but simulates a **stationary robot** that swaps hot-swap components in **one rack**.
**Where:** `v3/index.html` (open through a web server, e.g. `python3 -m http.server 4173` → `http://localhost:4173/v3/`). v2 stays at `/index.html`; each version links to the other.
**3D:** Three.js r160, saved in `v3/vendor/` (MIT, see `THREE-LICENSE.txt`), so the page needs no CDN and works on closed data-center networks.

## What is simulated

| Piece | v3 behaviour |
|---|---|
| **Rack** | Rack R-01: 5 servers (S1–S5, U36…U4), each with 7 front hot-swap bays: CPU, GPU, RAM, PSU, NIC, Disk, Fan (same component list and suppliers as v2). |
| **Robot** | Cartesian gantry on a short floor rail in front of the rack. **X** = carriage along the rail (replaces v2's base drive), **Z** = lift on the mast, **Y** = telescoping reach. A rule taken from the PLC: the arm retracts before any X/Z move. |
| **Parts caddy** | Beside the rack: a spare-parts shelf (3 of each part) and an RMA bin for failed parts. With no spare left, the work order shows **No spare** until you restock. |
| **Positioning vs identity** | Same as v2: the **ArUco** marker on each server's label plate gives the slot, and the **barcode** confirms which server it is. Both are read before every automated swap. |
| **Swap cycle** | Scan label → move to bay → reach in → pull the failed part → drop it in the RMA bin → take a spare → insert it → LED check. |
| **Scenarios** | **Idle** (healthy rack) · **Auto-Swap** (BMC reports which part failed; the robot swaps each one). |
| **Failure / manual** | "Simulate automation failure" stalls the next swap (ArUco lock lost) → **STALLED** → manual control. W/S changes server, A/D changes bay, Space scans, E swaps. Then *Resume auto* or *Park arm*. E-STOP → Reset re-homes the arm; a part still in the gripper is put away first. |
| **Gripper camera** | A real rendered view from a camera on the arm, with ArUco / barcode / LED-status overlay. |
| **Wear** | Each bay has a health %, stored in the browser. Worn parts are more likely to be chosen when faults are injected; a swapped part goes back to 100%. |

## Dashboard (reduced)

- **Data cards:** 2 big (Active faults · live, Components swapped / 7 days) and 3 small (Swap success, Avg time broken, Spares in caddy).
- **Top failing components:** failures per component type with the previous week marked, and a supplier verdict. Click a row to filter the timing cards to that part.
- **Component condition · live:** a 5 × 7 grid of health % for every bay in the rack (Good / Worn / Replace soon / Failed).
- **When components break:** a day × 4-hour heatmap with a plain-language note on the peak window.
- **Recent breakdowns · time broken:** time from failure to restored (LED check passed), with "still broken" shown live.

Mock weekly data is deterministic. Everything you do in the simulation is added as LIVE records.

## Mapping to Rev C / the IoT plan

The control and telemetry design in `docs/software-iot-plan.md` still applies, with two changes:

- `/cmd_vel` (base drive) is replaced by **jog intents per axis** (`CMD_JOG{axis: X|Z|Y, dir, step_um}` through `plc_bridge`, clamped by PLC soft limits). The manual pad sends **slot targets** (server/bay), not velocities.
- MQTT events keep the same topics. `telemetry/event` gains `{server, bay, comp, broke_at, restored_at}` so the dashboard can compute "time broken" from real data.
