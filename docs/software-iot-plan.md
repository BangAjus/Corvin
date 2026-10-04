# CORVIN — Software, IoT link & manual-control plan

**Scope:** how the software works end to end, how it talks to the robot (IoT), and whether/how we can add a *game-controller style* manual drive (forward / back / left / right) as a fallback when automated movement fails.
**Status:** plan + working UI demonstrator (`index.html`). Hardware parts are proposals, not yet built.
**Baseline:** Robot Design Document Rev C (§3.4 three tiers, §8 control & software, R-17/R-18 safety).

## Ringkasan (ID)

- **Bisa.** Kontrol manual maju/mundur/kiri/kanan itu pola standar di robotika (*teleoperation*). Yang bikin aman bukan tombolnya, tapi aturan di sisi robot: **watchdog** (berhenti kalau perintah berhenti datang), **prioritas** (manual mengalahkan auto), dan **E-stop hardware**.
- **Jalur kontrol harus lokal-dulu.** Halaman di Vercel itu `https`; browser memblokir `ws://` ke robot di LAN (mixed content). Solusi: UI yang sama juga disajikan dari robot sendiri (`http://robot.local`) atau pakai `wss://`.
- **Cloud (MQTT) untuk telemetri/dashboard**, bukan untuk setir real-time (latensi tidak pasti). Ini juga konsisten dengan klaim "edge inference, tanpa cloud".
- UI baru sudah mendemokan alurnya: auto gagal → **STALLED** → operator ambil alih manual → scan/ganti komponen → *Resume Auto* atau *Return to Dock*.

---

## 1. How the software works

```
 ┌──────────────────────── OPERATOR (browser) ────────────────────────┐
 │ Detection Simulation tab        Dashboard tab                       │
 │  map · camera · manual · log     weekly error rate · jobs · faults   │
 └───────────────┬───────────────────────────────▲─────────────────────┘
        control  │ (LAN, ws)                     │ telemetry/events (MQTT over wss)
                 ▼                               │
 ┌──────────────────────── TIER 3 · EDGE (Pi / laptop, ROS 2) ─────────┐
 │ dcim_client ── work orders in / closeout out                         │
 │ rack_perception ── ArUco (position) + barcode (identity) + LED check │
 │ task_manager ── state chart, Dijkstra route, mode arbitration        │
 │ teleop_gateway ── ws/MQTT → /cmd_vel_teleop  (NEW, this plan)        │
 │ twist_mux ── priority: manual > auto, per-input timeout (NEW)        │
 │ plc_bridge ── sole owner of RS-485 / Modbus RTU                      │
 └───────────────┬─────────────────────────────────────────────────────┘
                 │ Interface A: Modbus RTU (frozen register map, §B)
 ┌───────────────▼───────── TIER 2 · Delta DVP-14SS2 ───────────────────┐
 │ homing · soft limits · watchdog (heartbeat > 500 ms → stop, fault 0x21)│
 └───────────────┬─────────────────────────────────────────────────────┘
                 │ Interface B: STEP/DIR + discrete I/O
 ┌───────────────▼───────── TIER 1 · physical ─────────────────────────┐
 │ Z / Y drivers · cam servo · base motors · limit switches · HARDWIRED E-STOP │
 └──────────────────────────────────────────────────────────────────────┘
```

Rules carried over from Rev C (§3.4, §8): Tier 3 sends **intents**, never raw steps; the PLC re-validates everything; the E-stop breaks drive power and consults nobody.

### What the web app does today (simulation)

| Piece | Behaviour |
|---|---|
| **Library map** | 2 stacks × 2 faces × 4 slots = 16 servers, 3 aisles, dock at the south cross-aisle. Robot moves on aisle cells only. |
| **Routing** | Dijkstra over the aisle grid (cross-aisle cells cost 1.4, aisle cells 1.0). Robot always goes to the nearest open work order. |
| **Positioning vs identity** | **ArUco** marker → slot *position*. **Barcode** → server *identity*. Both are logged and shown in the camera POV. |
| **Faults** | A server can have **several faulty components across categories** (Compute, Memory, Power, Network, Storage, Cooling). Each component is swapped separately. |
| **Modes** | `AUTO` (Scenario B), `HUMAN ASSIST` (Scenario C, hidden faults, human walks and scans), `MANUAL`, `STALLED`, `E-STOP`. |
| **Dashboard** | Mock weekly data (deterministic) blended with live events from the simulation (today's bars, active faults). |
| **Server log** | Lives in the Detection Simulation tab; the real simulation log, kept in the browser between sessions. |

---

## 2. How it connects to the IoT

Three separate links, because they have different needs:

| Link | Purpose | Transport | Why |
|---|---|---|---|
| **Control** (manual drive, mode, e-stop) | Real-time, must stop on loss | WebSocket on the **local network** (`ws://robot.local:8765`, or `wss://` with a local cert) | Low, predictable latency; works with no internet (data-center networks are restricted). |
| **Telemetry & events** | Dashboard, history, alerts | **MQTT over WebSocket (wss)** to a broker; robot connects *outbound* | Outbound-only works behind NAT/firewalls. MQTT gives retained state, QoS and Last Will for free. |
| **Video** | Operator POV | Local MJPEG/WebRTC; cloud gets low-FPS snapshots only | Video over the cloud is heavy and laggy; do not steer on it. |

### The Vercel gotcha (important)

The deployed page is `https://…vercel.app`. Browsers treat a plain `ws://` connection from an HTTPS page as **mixed content and block it** (Firefox is strictest; Chromium has some leniency for `localhost` only). So the Vercel build **cannot** drive a robot at `ws://192.168.x.x`. Options, in order of preference:

1. **Serve the same `index.html` from the robot** (nginx/Caddy on the Pi → `http://robot.local`). Page and socket are both plain HTTP/WS, no mixed content. Vercel stays as the demo/analytics site.
2. Give the robot a local TLS cert (Caddy internal CA / mkcert) and use `wss://robot.local`.
3. Route control through the cloud broker (`wss://`) — works, but only acceptable at low speed because latency is not bounded.

### MQTT topic contract (proposal)

```
corvin/{robot}/status                 retained, LWT "offline"          {online, fw, mode}
corvin/{robot}/telemetry/state        QoS0 · 5 Hz                      {mode, cell|pose, battery, aruco_lock, rtt_ms}
corvin/{robot}/telemetry/event        QoS1                             {type, server, component, result, t}
corvin/{robot}/cmd/mode               QoS1                             {mode: "auto"|"manual", lease}
corvin/{robot}/cmd/estop              QoS1                             {engage: true|false}
corvin/{robot}/cmd/drive              QoS0 · 10–20 Hz (LAN path only)  {seq, lease, vx, vy, wz}
```

Drive message, one JSON per tick **while a key is held** (not "go" / "stop" events):

```json
{ "seq": 4821, "lease": "op-7f3a", "vx": 0.10, "vy": 0.0, "wz": 0.0 }
```

`vx` forward/back (m/s), `wz` rotate (rad/s), `vy` strafe (only if the base is mecanum). Limits are enforced **on the robot**, not in the UI.

---

## 3. Manual control — is it possible?

**Yes, and it is the normal way to build the fallback.** Pieces that already exist and are widely used:

- Browser input: keyboard, on-screen D-pad, and gamepads via the **Gamepad API** (polled each animation frame; supported by essentially all modern browsers). The demonstrator already supports WASD/arrows, the D-pad and a gamepad.
- Robot side: ROS 2 **`twist_mux`** takes several velocity sources, each with a **priority** (0–255, higher wins) and a **timeout** after which that input is dropped to zero. Typical setup: teleop priority above navigation, short timeout on teleop.
- Safety pattern: a **dead-man / timeout** — the robot only moves while commands keep arriving, and stops by itself when they stop.

### Design

```
 key / pad / D-pad ──► UI (10–20 Hz while held) ──► teleop_gateway ──► twist_mux ──► base driver
                                                         │                  ▲
                              lease + seq check ─────────┘      auto nav ───┘ (lower priority)
```

| Rule | Value / behaviour | Why |
|---|---|---|
| **Command timeout** | 200–300 ms on the robot (twist_mux input timeout + MCU watchdog) | Releasing the key, Wi-Fi drop or a crashed tab all end in a stop. UI-side "stop" messages are not trusted. |
| **Link watchdog** | Keep Rev C **R-17**: stale heartbeat > 500 ms → controlled stop, fault `0x21`, explicit operator CLEAR_FAULT + re-home | Already specified; manual drive must not weaken it. |
| **Priority** | `E-STOP (hardware)` > `manual lease` > `auto` | Taking over must be instant; handing back must be explicit. |
| **No silent resume** | After a takeover, auto stays off until the operator presses **Resume Auto** | Prevents the robot lurching off while someone is standing next to it. |
| **Single operator lease** | One `lease` token at a time; drive messages with another lease are ignored | Two tabs/two people cannot fight over the robot. |
| **Speed cap** | Manual ≤ ~0.15 m/s near racks; slower at dock; ramped acceleration | Small robot, fragile connectors. Values to be tuned on the bench. |
| **Hardwired E-stop** | Unchanged (R-18, ≤ 100 ms, works with PLC unpowered) | Software e-stop in the UI is a convenience, never the safety function. |

### Mapping to Rev C

- **Base (forward/back/left/right):** `/cmd_vel_teleop` → `twist_mux` → base driver. This is the Phase 5 `dock_controller` side (`/cmd_vel`, bump switches, Nav2).
- **Arm jog (Z up/down, Y in/out):** must go through `plc_bridge` (only owner of the serial port) as an **intent**, e.g. `CMD_JOG{axis, dir, step_um}` clamped by PLC soft limits. That command does not exist in register map v1, so it needs a **`PROTOCOL_VERSION` bump to v2** (§8.3: version mismatch is a fault, by design).

### Mode state machine

```
        ┌───────────── Resume Auto (explicit) ─────────────┐
        ▼                                                    │
     [ AUTO ] ──nav fault / link loss / operator──► [ STALLED ] ──► [ MANUAL ] ──┘
        │                                                  ▲            │
        └───────────────── E-STOP (any state) ─────────────┴────────────┘
                              Reset requires operator + re-home
```

The simulation implements exactly this (toggle *Manual*, or let an automation failure trigger it; *Resume Auto* / *Return to Dock*; *E-STOP* / *Reset*).

---

## 4. Build plan

| Phase | Deliverable | Acceptance test |
|---|---|---|
| **P0** (done) | Redesigned web UI: library map, multi-component faults, ArUco + barcode, manual drive, dashboard | Walk through: auto run, stall → manual → resume; dashboard updates from live events |
| **P1** · bench teleop (2–3 days) | ESP32/Pi + 2 motors; page served from the robot; WebSocket drive; 250 ms watchdog | Yank Wi-Fi mid-drive ×10 → stop ≤ 300 ms every time. Close the tab → stop. |
| **P2** · ROS 2 bridge | `teleop_gateway` + `twist_mux` (manual 10 / auto 5, short timeouts); stall hand-over | Force a nav fault → manual takes over within one command; auto stays off until Resume |
| **P3** · telemetry to cloud | MQTT broker (wss), robot publishes state/events; Dashboard reads real data | Kill the internet: robot keeps working; reconnect: events are back-filled (QoS1) |
| **P4** · video | Local MJPEG/WebRTC in the camera panel; ArUco/barcode overlays | Glass-to-glass latency measured on the venue Wi-Fi; decide if manual drive needs the camera |
| **P5** · arm jog | `CMD_JOG` + protocol v2; D-pad Z/Y | Jog into soft limit → rejected with `0x11`, nothing moves |

### Measure before promising

Do not quote latency numbers until measured on the real network under load: round-trip time of the control socket, watchdog trip time, and manual→auto hand-over time. Put them in the test log (rosbag2 already records every trial in Rev C §8.1).

## 5. Risks / open questions

1. **Mixed content** — decide now: robot-hosted UI (recommended) vs local TLS.
2. **Base type** — differential drive has no true "left/right" (it rotates); strafing needs mecanum. The UI sends `vx/wz` (+`vy`); the base decides.
3. **Fault visibility in manual mode** — in the sim, scanning reveals every fault on a server. On hardware, hidden component faults need the actual inspection step; define what the operator is expected to do.
4. **Auth** — a public Vercel URL must never be able to move a robot. Control path needs a lease/token and should only be reachable on the robot's network.
5. **Dashboard truth** — today's charts are mock + live sim events. Switching to real data is P3.

## Sources

- [twist_mux (ROS wiki mirror)](https://mirror.umd.edu/roswiki/twist_mux.html) and [Controlling a robot with multiple inputs using twist_mux](https://robofoundry.medium.com/controlling-a-robot-with-multiple-inputs-using-twist-mux-4535b8ed9559) — per-input priority and timeout.
- [Using the Gamepad API in web games (Smashing Magazine)](https://www.smashingmagazine.com/2015/11/gamepad-api-in-web-games/) and [Jumping the hurdles with the Gamepad API (web.dev)](https://web.dev/articles/doodles-gamepad) — polling with `requestAnimationFrame`.
- [Handling mixed content issues when serving WS over HTTP](https://www.resumelens.org/blog/websockets/handling-mixed-content-issues-when-serving-ws-over-http) and [Mozilla bug 1370861](https://bugzilla.mozilla.org/show_bug.cgi?id=1370861) — `ws://` blocked from HTTPS pages; localhost behaviour differs by browser.
- [MQTT over WebSocket (EMQX docs)](https://docs.emqx.com/en/emqx/latest/connect-emqx/mqtt-over-websocket.html) and [WebSocket vs MQTT](https://websocket.org/comparisons/mqtt/) — retained messages, Last Will, QoS over WebSocket.
- [NVIDIA Isaac robot remote control](https://docs.nvidia.com/isaac/doc/extensions/robot_remote_control/doc/index.html) — dead-man switch mode applies in both autonomous and manual mode.
