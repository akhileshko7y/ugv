# Cloud-Teleoperated UGV Arena

Drive a physical UGV rover over the internet from a browser at home.
Hard requirement: **< 120ms round-trip latency** (stick input → visible frame).

## Architecture

```
[ GROUND STATION ]              [ CLOUD (AWS ap-south-1) ]            [ UGV ARENA ]

Chromium browser                Nginx Ingress (TCP 443 wss)           Rover SBC (Pi 5 / Jetson)
 - HTML5 Gamepad API      ───▶   └─ Matchmaker (Go, room alloc)  ───▶   - LiveKit client daemon
 - WebRTC engine                 └─ LiveKit SFU pod                     - GStreamer HW H.264
     │                                    │                                   │ UART / CRSF
     └────────────────────────────────────┴───────────────────────────────────┤
       Direct UDP: video over SRTP, controls over SCTP data channel      Motor MCU
       (media bypasses the load balancer)                                 - PWM generator
                                                                          - LiDAR guardrail loop
Input: RadioMaster Pocket (ELRS) or Xbox controller
```

Key design points:
- Video encoding is offloaded to onboard **hardware** encoder blocks (never CPU).
- The control data channel is **unreliable + unordered** (`maxRetransmits: 0`,
  `ordered: false`) — a stale input packet is worse than a dropped one.
- Signaling goes through the LB; **media does not**. The browser binds UDP
  directly to the SFU host node.
- Collision guardrails live on the **motor MCU**, not the SBC — they must hold
  when the network link dies.

## Project State

**`docs/TODO.md` is the source of truth** for what's done and what's next.
Read it before starting work, and update it as items land.

The project is currently at Phase 1 (hardware procurement / dev OS setup).
No code exists yet — directories get created as each phase starts.
