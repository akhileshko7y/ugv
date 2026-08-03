# Cloud-Teleoperated UGV Arena

Drive a real robot, over the internet, from your browser — with the responsiveness of a video game.

> **Status: early build.** Phase 1 of 4 (hardware baseline). No working system yet.
> See [`docs/TODO.md`](docs/TODO.md) for the roadmap.

## The Problem

Remote-controlling a physical vehicle over the public internet is fundamentally a
latency problem, and most stacks get it wrong.

The moment you route a robot through a normal cloud application, you inherit every
assumption that stack was built on: TCP retransmits, load balancers that terminate
and re-forward your traffic, software video encoders that add 50ms before a frame
even leaves the vehicle, and buffering layers that would rather deliver a late
packet than no packet. Each is reasonable in isolation. Stacked together they turn
into half a second of lag — and half a second is the difference between driving a
rover and watching a recording of one you already crashed.

The second problem is that latency isn't a safety mechanism. Any system that lets a
person drive a real machine they cannot see with their own eyes will eventually
have to answer: what happens when the network drops mid-corner? If the answer
depends on a packet arriving, the answer is a wall.

## The Idea

Build a fleet of physical rovers in an arena that anyone can queue up for and drive
from a browser tab, with sub-120ms round-trip latency — input to visible frame.

Four decisions carry most of the weight:

**Encode in silicon, not software.** Frames go straight from the camera sensor into
the onboard hardware encoder blocks (Pi 5 / Jetson), never touching the CPU for
compression. Budget: under 15ms.

**Treat control inputs as disposable.** Joystick state is sent over an unreliable,
unordered WebRTC data channel at 60Hz. A retransmitted stick position is *stale* —
worse than useless, because the next fresh one is 16ms behind it. We would rather
drop it.

**Let media skip the front door.** Signaling and auth go through the load balancer
over WebSocket. Once the session token is issued, the browser binds UDP directly to
the SFU node holding that session. Video and control packets never touch the LB.

**Put the reflexes in the vehicle.** Collision avoidance runs on the motor
microcontroller, not the onboard Linux computer and definitely not the cloud.
Front LiDAR under 30cm scales down max throttle; under 10cm freezes the motors.
This loop keeps running when the network is gone, which is exactly when it matters.

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

Cloud region is Mumbai (ap-south-1) to keep packets from crossing borders on their
way to a rover sitting in the same country as the driver.

## Stack

| Layer | Choice |
|---|---|
| Transport | WebRTC — SRTP for video, SCTP data channel for control |
| Media server | LiveKit SFU on Kubernetes (StatefulSets, stable rover identities) |
| Matchmaker | Go — session queue, rover health, room token minting |
| Client | Browser + HTML5 Gamepad API |
| Onboard | Raspberry Pi 5 or Jetson, GStreamer with hardware H.264 |
| Motor control | Arduino / STM32 / RP2040 over serial UART |
| Observability | Prometheus + Grafana — RTT, signal strength, CPU temp, pack voltage |

## Roadmap

1. **Local control & hardware baseline** — transmitter, dev OS, stick calibration
2. **Vehicle hardware & guardrails** — chassis, UART bridge, LiDAR safety loop, HW encoding
3. **WebRTC signaling pipeline** — cloud SFU, web client, direct UDP handshake
4. **Fleet operations** — Kubernetes, matchmaking, monitoring, auto-kill dashboard

Full breakdown in [`docs/TODO.md`](docs/TODO.md).
