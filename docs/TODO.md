# Cloud-Teleoperated UGV Arena — Things To Do

Working task list. Phases are sequential: bench prototype → scaled cluster.
Mark items `[x]` as they land. Add notes inline under an item when reality
diverges from the plan — this file is the source of truth for project state.

**Target:** < 120ms round-trip latency (control input → visible frame).

---

## Phase 1 — Local Control & Hardware Baseline

- [ ] **Procure components** — RadioMaster Pocket (ELRS version), flat-top
      unprotected 18650 cells, right-angle USB-C data cable. Sources: Robu.in,
      TujoRC.
- [ ] **Configure dev OS** — Ubuntu 22.04 LTS or Pop!_OS partition on the
      workstation. udev rules for the transmitter, add user to the input group.
- [ ] **Calibrate stick metrics** — install Steam + Liftoff, verify Hall effect
      inputs map cleanly across gamepad axes with no drift or lag.

## Phase 2 — Vehicle Hardware & Automated Guardrails

- [ ] **Assemble the chassis** — 4WD UGV platform, compute mounted inside a
      360° polycarbonate roll cage.
- [ ] **Build the control bridge** — serial UART link between the SBC
      (Pi 5 / Jetson) and the motor microcontroller.
- [ ] **Write sensor guardrails** (C++ on the MCU) — poll front LiDAR
      continuously. Below 30cm: scale down max throttle. At 10cm: absolute
      motor freeze. Guardrail runs on the MCU, not the SBC — it must survive
      loss of the link.
- [ ] **Test native HW encoding** — GStreamer pipeline piping raw CMOS frames
      into the SBC's hardware encoder blocks. Goal: sub-15ms compression, no
      CPU bottleneck.

## Phase 3 — WebRTC Real-Time Signaling Pipeline

- [ ] **Launch cloud network** — VM in AWS Mumbai (ap-south-1). Firewall:
      TCP 443, UDP 50000–60000.
- [ ] **Configure the SFU** — LiveKit (or Mediasoup). Data channel configured
      unreliable + unordered (`maxRetransmits: 0`, `ordered: false`).
- [ ] **Write the web client** — HTML5 Gamepad API, normalize stick tilts to
      -1.0…+1.0 floats, push over the WebRTC data channel at 60Hz.
- [ ] **Establish direct UDP handshake** — verify ICE: token authorized over
      WebSocket through the LB, then browser binds UDP directly to the host
      node handling the session (media bypasses the LB).

## Phase 4 — Fleet Ops, Management & Analytics

- [ ] **Deploy to Kubernetes** — containerize media servers, deploy as
      StatefulSets for stable identities (`rover-00` … `rover-09`).
- [ ] **Build the matchmaking backend** (Go) — WebSocket registry: queue
      incoming sessions, track rover health (available / playing / charging),
      mint room tokens.
- [ ] **Deploy monitoring** — Prometheus scraping RTT, cell signal strength,
      CPU temp, battery voltage.
- [ ] **Create the fleet dashboard** — Grafana grid of all active rovers.
      Auto-kill / e-brake safe-stop when ping > 250ms or pack voltage < 3.5V.

---

## Open Questions

- [ ] Pi 5 or Jetson for the onboard computer?
- [ ] MCU choice: Arduino / STM32 / RP2040?
- [ ] Uplink from the rover — 4G/5G modem or arena WiFi?

## Next Up

Pick one to start:

1. udev rules + permissions to map the RadioMaster Pocket for the simulator.
2. Minimal Python WebRTC data channel translating joystick floats over the network.
3. C++ MCU logic reading LiDAR and overriding steering before a crash.
