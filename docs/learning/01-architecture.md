# Architecture — Cloud-Teleoperated UGV Arena

*Code-level primer: the main abstraction and how a request flows through it.*

## Concepts

<!-- GEN:concepts topic=system-architecture-and-decisions level=code depth=deep src=931b9c4 START -->

> **Reading note.** No code exists yet — `CLAUDE.md:36-37` states directories get
> created as each phase starts. So the citations below point at the design
> documents (`README.md`, `CLAUDE.md`, `docs/TODO.md`), which are currently the
> source of truth. Regenerate this lesson against real source once Phase 2 lands.

### The one requirement everything derives from

Every structural choice in this system is downstream of a single number:
**< 120ms round-trip, stick input to visible frame** (`CLAUDE.md:3-4`,
`docs/TODO.md:7`). Not average — the number a driver *feels* on every input.

`README.md:10-19` names the failure mode this defends against: TCP retransmits,
load balancers that terminate and re-forward, software encoders that add 50ms,
and buffers that prefer a late packet to no packet. Each is reasonable alone;
stacked, they become half a second of lag. The architecture is best understood
as **a list of defaults that were deliberately refused.**

The second requirement is independent of the first: `README.md:21-24` argues
latency is not a safety mechanism. If the answer to "what happens when the
network drops mid-corner" depends on a packet arriving, the answer is a wall.

### The main abstraction: three trust domains

```
┌─ GROUND STATION ─────┐  ┌─ CLOUD (ap-south-1) ─┐  ┌─ ROVER ──────────────┐
│                      │  │                      │  │                      │
│  Browser             │  │  Nginx Ingress :443  │  │  SBC (Linux)         │
│  • Gamepad API       │  │  • WSS signaling     │  │  • LiveKit client    │
│  • WebRTC engine     │  │  Matchmaker (Go)     │  │  • GStreamer encode  │
│                      │  │  • rooms, tokens     │  │        │             │
│                      │  │  LiveKit SFU pod     │  │        │ UART        │
│                      │  │  • forwards packets  │  │  ┌─────▼─────┐       │
│                      │  │                      │  │  │ Motor MCU │       │
│                      │  │                      │  │  │ • PWM     │       │
│                      │  │                      │  │  │ • LiDAR   │       │
│                      │  │                      │  │  │ • failsafe│       │
└──────────┬───────────┘  └──────────┬───────────┘  └──┴─────┬─────┴───────┘
           │                         │                       │
           └── direct UDP: SRTP video + SCTP control ─────────┘
               (media bypasses the load balancer)
```

Adapted from `CLAUDE.md:8-20` and `README.md:53-65`.

The domains differ in **what they are permitted to be late for**:

| Domain | May be late? | May crash? | Why |
|---|---|---|---|
| Cloud | Yes | Yes | Rover has local reflexes |
| SBC (Linux) | Yes | Yes | Same — it holds no safety authority |
| **Motor MCU** | **No** | **No** | It is the only thing touching the motors |

That table *is* the architecture. Everything below is an elaboration of it.

### Principle 1 — Media skips the front door

Signaling goes through the load balancer; media does not (`CLAUDE.md:26-27`).
`README.md:43-45` gives the sequence: auth and signaling over WebSocket through
the LB, then once the session token is issued, the browser binds UDP **directly**
to the SFU node holding that session. `docs/TODO.md:43-45` restates this as a
verifiable Phase 3 task — confirm ICE negotiates a direct path.

The reason is that a load balancer terminates and re-forwards, which is a hop of
latency and a buffer, on the exact traffic that can least afford either. So the
control plane (rare, reliable, authenticated) and the data plane (constant,
lossy, latency-critical) take physically different paths.

This is why the firewall is two rules, not one (`docs/TODO.md:37-38`):

| Port | Plane | Carries |
|---|---|---|
| TCP 443 | control | WSS signaling, matchmaker registry, room tokens |
| UDP 50000–60000 | data | SRTP video, SCTP data channel |

### Principle 2 — Control inputs are disposable

The data channel is configured `maxRetransmits: 0, ordered: false`
(`CLAUDE.md:24-25`, `docs/TODO.md:39-40`). The web client normalizes stick tilts
to `-1.0…+1.0` floats and pushes at 60Hz (`docs/TODO.md:41-42`).

`README.md:37-41` gives the reasoning: a retransmitted stick position is *stale*
— worse than useless, because the next fresh one is only 16ms behind it. A
protocol that guarantees delivery is guaranteeing the delivery of a lie about the
present.

The important part is that this is not one setting. **It is a principle that must
be re-applied at every hop**, because each hop offers its own tempting buffer:

```
WebRTC data channel  →  maxRetransmits: 0, ordered: false
Rover receive loop   →  drop any packet whose sequence is older than the newest
Pi → MCU serial      →  drain the buffer fully; keep only the last frame
Any Go→C IPC         →  latest-value-wins; never a queue
```

Anywhere a queue forms, the driver is controlling the past.

### Principle 3 — Reflexes live in the vehicle

Collision guardrails run on the motor MCU, not the SBC (`CLAUDE.md:28-29`).
`docs/TODO.md:27-30` specifies the loop — poll front LiDAR continuously; below
30cm scale down max throttle; at 10cm freeze the motors — and states the reason
directly: *it must survive loss of the link.* `README.md:46-49` adds that this
loop keeps running when the network is gone, which is exactly when it matters.

Note the ordering this implies inside the MCU. Driver intent enters at the top
and is progressively overruled on the way down:

```
1. command from the SBC       ← what the driver wants
2. link-timeout failsafe      ← overrules if no frame arrived recently
3. LiDAR clamp                ← overrules regardless of link health
4. PWM out to ESC / servo     ← the only thing that touches the motors
```

Nothing lower can be re-enabled by anything higher. There is deliberately no
"override" flag the SBC can set, because the moment one exists, a corrupted
packet can set it.

**The failsafe is a timeout, not a message.** A severed link cannot deliver a
stop command; the only thing that works is a clock on the vehicle noticing that
nothing has arrived.

### Principle 4 — Encode in silicon

`README.md:33-35` requires frames go from the camera sensor into onboard hardware
encoder blocks, never touching the CPU for compression, with a budget of under
15ms. `CLAUDE.md:23` states it as an invariant: never CPU.
`docs/TODO.md:31-33` makes it a Phase 2 test — a GStreamer pipeline into the
SBC's hardware encoder, sub-15ms, no CPU bottleneck.

The cost of getting this wrong is not just the ~30ms of extra encode time. A
software encoder competes for CPU with the WebRTC stack and the serial bridge on
the same board, which makes latency *variable* — and jitter is harder to drive
through than a constant delay. **This principle currently has an unresolved
hardware dependency; see Decision D-1.**

### How a session is established

The rover has no public IP and needs none. Both endpoints dial *out*; nothing
ever dials in. This is what makes the design work behind arena WiFi or carrier
CGNAT, where inbound connections are impossible.

```
1. Rover boots; daemon opens WSS to the matchmaker through the Ingress (:443)
      registers identity + health: available / playing / charging
      (docs/TODO.md:51-53)
      ── this socket stays open; it is how the cloud "reaches" the rover

2. Driver queues. Matchmaker mints a room token, pushes it down that socket

3. Rover + browser both connect to the SFU, negotiate ICE

4. Direct UDP established on 50000–60000; media flows, Ingress drops out
      (README.md:43-45, docs/TODO.md:43-45)
```

Identity is therefore an **application-layer** fact, not a network one. A rover
is `rover-03` because an authenticated socket says so and the registry agrees —
which is what the StatefulSet stable identities `rover-00`…`rover-09` are for
(`README.md:75`, `docs/TODO.md:49-50`). The IP is transient and uninteresting.

### The two data paths

**Control (downstream), ~8 bytes at 60Hz.** Cheap, and will comfortably meet
budget:

```
Gamepad API → normalize to -1.0…+1.0 → data channel → SFU → rover daemon
  → sequence check, drop if stale → UART frame → MCU → clamp → PWM → ESC
```

**Video (upstream).** Expensive, and where the 120ms is actually won or lost:

```
CMOS sensor → ISP → HW H.264 encoder → RTP payload → SRTP → SFU (forward only)
  → browser jitter buffer → decode → compositor → display
```

Indicative budget — the two rows most often left unmeasured are marked:

| Stage | Budget |
|---|---|
| Camera exposure + readout ⚠ | 5–20ms |
| Hardware encode (`README.md:33-35` target) | < 15ms |
| Rover → SFU → browser | 20–50ms |
| Jitter buffer | 0–200ms ← **the biggest single lever** |
| Decode | 5–15ms |
| Compositor + vsync + display ⚠ | 8–25ms |

Default jitter buffers are tuned for watching video, not driving. For teleop the
target is near-zero, accepting occasional artifacts: a driver can compensate for
a glitch and cannot compensate for lag.

> Explore on your own: `chrome://webrtc-internals` exposes `jitterBufferDelay`,
> RTT, and loss live. For true end-to-end measurement, film a millisecond timer
> on your screen with the rover camera and photograph both — that is the only
> method that captures exposure and display latency too.

### Failure model

The design is best judged by what happens when each piece dies:

| Failure | Behavior | Guaranteed by |
|---|---|---|
| Network drops mid-corner | MCU stops seeing frames → neutral; LiDAR still active | `docs/TODO.md:27-30` |
| SBC kernel panic | Identical — the UART simply goes silent | `CLAUDE.md:28-29` |
| Encoder starves the CPU | SBC late, MCU unaffected; degraded control, safety intact | Domain table above |
| Driver aims at a wall | LiDAR clamp overrules regardless of link | `README.md:46-49` |
| Link degrades but survives | Fleet auto-kill: ping > 250ms or pack < 3.5V | `docs/TODO.md:56-57` |

Every row is handled **on the vehicle**, with no packet needing to arrive. That
is the concrete meaning of `README.md:21-24`.

<!-- GEN:concepts END -->

## Decisions

<!-- GEN:decisions topic=system-architecture-and-decisions level=code depth=deep src=931b9c4 START -->

### Settled

These are committed in the design docs. Each is a refusal of a reasonable default.

| # | Decision | Rationale | Source |
|---|---|---|---|
| S-1 | WebRTC transport — SRTP video, SCTP data channel | Only mainstream stack with unreliable delivery + NAT traversal + browser support | `README.md:74` |
| S-2 | SFU topology (LiveKit; Mediasoup as alternate) | Forwards packets without transcoding; both peers dial out to one reachable host | `README.md:75`, `docs/TODO.md:39` |
| S-3 | Data channel unreliable + unordered | Stale input is worse than dropped input | `CLAUDE.md:24-25` |
| S-4 | Media bypasses the load balancer | LB adds a hop and a buffer to the traffic least able to afford them | `CLAUDE.md:26-27` |
| S-5 | Guardrails on the MCU, not the SBC | Must hold when the link dies | `CLAUDE.md:28-29` |
| S-6 | Split SBC (Linux) / MCU (bare metal) over serial UART | Linux is non-deterministic; PWM timing cannot be preempted | `docs/TODO.md:25-26`, `README.md:79` |
| S-7 | Hardware video encoding, never CPU | Software encode blows budget and adds jitter | `CLAUDE.md:23` |
| S-8 | AWS Mumbai (ap-south-1) | Packets stay in-country between driver and rover | `README.md:67-68` |
| S-9 | Go matchmaker — queue, health, token minting | — | `docs/TODO.md:51-53` |
| S-10 | Browser client on the HTML5 Gamepad API | Zero install; the product is "a browser tab" | `README.md:3`, `docs/TODO.md:41-42` |
| S-11 | K8s StatefulSets for stable rover identities | `rover-00`…`rover-09` survive rescheduling | `docs/TODO.md:49-50` |
| S-12 | Prometheus + Grafana; auto-kill at ping > 250ms or < 3.5V | Degraded teleop must stop itself | `docs/TODO.md:54-57` |
| S-13 | Failsafe is a link timeout, not a stop command | A dead link cannot deliver anything | `README.md:21-24` |

### Open

Ordered by how much other work they block.

**D-1 · Onboard computer: Pi 5 or Jetson?** — `docs/TODO.md:63`

The highest-leverage open decision, and it needs re-examination before hardware
spend. `README.md:78` lists "Raspberry Pi 5 or Jetson, GStreamer with hardware
H.264", and `README.md:33-35` assumes both have hardware encoder blocks.

> ⚠️ **Unverified external finding, not a repo claim:** the Pi 5's VideoCore VII
> is widely reported to have **dropped the H.264 hardware encoder** that the Pi 4
> had, leaving software x264 as the only H.264 path. If that holds, S-7 is
> unsatisfiable on a Pi 5 and `docs/TODO.md:31-33` cannot pass. Jetson's NVENC is
> a genuine hardware encoder. **Verify against current Raspberry Pi documentation
> before ordering** — do not treat this paragraph as settled.

Blocks: all of Phase 2. Recommendation: verify, and if confirmed, Jetson.

**D-2 · MCU: Arduino, STM32, or RP2040?** — `docs/TODO.md:64`

Requirements implied by S-5/S-6: one UART for the SBC, one UART or I²C for the
LiDAR, two PWM outputs, a hardware watchdog. Leaning RP2040 — 3.3V logic matches
the SBC with no level shifter, two UARTs, 16 PWM slices, and a second core that
can hold the safety loop while core 0 handles serial. A 5V Arduino needs level
shifting on the SBC's RX line. Blocks: `docs/TODO.md:27-30`.

**D-3 · Uplink: 4G/5G modem or arena WiFi?** — `docs/TODO.md:65`

WiFi is lower latency and easier to debug; cellular is what makes the arena
location-independent and is the harder case (weak uplink, CGNAT, IP changes on
handover requiring ICE restart). Suggest building on WiFi and treating cellular
as a later hardening milestone, since it only makes an already-solved path
harder. Blocks: Phase 3 latency validation realism.

**D-4 · SBC → MCU link: USB CDC or GPIO UART?**

Not yet in `docs/TODO.md`; implied by `docs/TODO.md:25-26`. USB needs no config
and powers the MCU; GPIO UART is more deterministic and survives USB
re-enumeration. Suggest USB for bench work, GPIO UART for the vehicle.

**D-5 · Process layout on the SBC**

`README.md:76` fixes Go for the matchmaker but nothing states the rover daemon's
language. If the daemon is Go, it can write the serial port directly and no C
process is needed on the SBC at all — C stays on the MCU where determinism is
guaranteed. If a C-only vendor SDK forces a second process, prefer shared memory
with atomic latest-value semantics over any queue-shaped IPC (Principle 2).

**D-6 · Camera, resolution, and codec**

Unspecified beyond "GStreamer with hardware H.264" (`README.md:78`). For driving,
temporal resolution beats spatial — 720p60 over 1080p30. Also unsettled: CBR vs
VBR, and whether to use intra-refresh instead of periodic keyframes (avoids a
bitrate spike and a latency bump every GOP).

**D-7 · Jitter buffer target**

The single biggest lever in the video budget and currently unspecified. Cannot be
chosen analytically — must be measured on the real link, then fixed as a number.

**D-8 · TURN fallback policy**

If UDP is blocked, WebRTC falls back to relayed TCP/443 and the session will
connect while badly exceeding budget. Undecided whether to allow a degraded
session or refuse it. Relates to S-12: a silently-degraded rover is worse than an
offline one, so a relayed-candidate metric is likely wanted either way.

**D-9 · Driver authentication and session policy**

`docs/TODO.md:51-53` covers rover health and token minting but not who may
queue, session length, or what happens at session end.

**D-10 · Ops access plane**

Undocumented. `ssh` to a CGNAT rover needs an overlay (WireGuard/Tailscale).
Should stay strictly separate from the media path so driving latency never
depends on it.

**D-11 · Battery telemetry path**

`docs/TODO.md:54-57` requires pack voltage for the auto-kill rule, but nothing
states how it is sensed. Most likely an MCU ADC on a divider, reported back up
the UART — which makes the SBC↔MCU link bidirectional, a detail D-4 must carry.

<!-- GEN:decisions END -->

## Check yourself

<!-- GEN:quiz topic=system-architecture-and-decisions level=code depth=deep src=931b9c4 START -->

**Q1.** The rover is mid-corner at speed when the cellular link drops completely.
What stops it?

- A) The matchmaker detects the dead session and sends a stop command
- B) The SBC's LiveKit daemon notices disconnection and zeroes the throttle
- C) The MCU's link timeout expires and it drives the motors to neutral
- D) The browser sends an e-brake packet on `onclose`

<details><summary>✅ Reveal answer</summary>

**C** — `CLAUDE.md:28-29` places the guardrails on the motor MCU precisely
because "they must hold when the network link dies," and `docs/TODO.md:27-30`
requires the loop "survive loss of the link."

A and D are wrong for the same reason: both require a packet to *arrive* over the
link that just died. `README.md:21-24` names this exact fallacy — if the answer
depends on a packet arriving, the answer is a wall. B is the most tempting
distractor because the SBC is a real component that really does run the LiveKit
client (`README.md:56-57`), but it sits on the wrong side of the trust boundary: it
can crash or hang, and it is explicitly *not* where safety lives.

</details>

**Q2.** How is the control data channel configured, and why?

- A) `ordered: true, maxRetransmits: 3` — control input is too important to lose
- B) `ordered: false, maxRetransmits: 0` — a stale input is worse than a dropped one
- C) `ordered: true, maxRetransmits: 0` — ordering is cheap, retransmits are not
- D) Reliable TCP fallback, since control packets are only 8 bytes

<details><summary>✅ Reveal answer</summary>

**B** — stated identically in `CLAUDE.md:24-25` and `docs/TODO.md:39-40`.

A inverts the reasoning: `README.md:37-41` explains a retransmitted stick
position is stale, and the next fresh one is only 16ms behind it — so reliability
actively harms you. C is subtle and wrong: ordering is *not* free, because
enforcing it means holding a newer packet while waiting for an older one, which
is the same staleness in different clothing. D confuses size with cost — the
problem with TCP is head-of-line blocking, which a small payload does not avoid
(`README.md:10-19`).

</details>

**Q3.** Which traffic passes through the Nginx Ingress?

- A) Everything — it is the single entry point to the cluster
- B) Video only; control rides the direct UDP path
- C) Signaling and auth only; video and control both go direct
- D) Nothing; it exists solely for the Grafana dashboard

<details><summary>✅ Reveal answer</summary>

**C** — `CLAUDE.md:26-27`: "Signaling goes through the LB; media does not."
`README.md:43-45` adds that once the token is issued the browser binds UDP
directly to the SFU node, and "video and control packets never touch the LB."

A is the conventional cloud assumption the design deliberately refuses
(`README.md:10-19` lists load balancers that terminate and re-forward among the
causes of half-second lag). B splits the wrong way — both media *and* control
take the direct path, which is why the firewall opens a UDP range alongside 443
(`docs/TODO.md:37-38`). D contradicts `docs/TODO.md:43-45`, which requires the
session token to be authorized over WebSocket through the LB.

</details>

**Q4.** Why StatefulSets rather than a Deployment for the media server pods?

- A) StatefulSets get stable identities, so `rover-03` stays `rover-03`
- B) StatefulSets are scheduled with higher priority, reducing latency
- C) Deployments cannot mount the persistent volumes video recording needs
- D) StatefulSets pin pods to nodes, keeping UDP ports stable

<details><summary>✅ Reveal answer</summary>

**A** — `docs/TODO.md:49-50` asks for StatefulSets "for stable identities
(`rover-00` … `rover-09`)", and `README.md:75` repeats "stable rover identities."
This matters because identity in this system is an application-layer fact, not a
network one — a rover behind CGNAT has no stable address to be known by.

B invents a scheduling property StatefulSets do not have. C describes a real
StatefulSet capability (per-pod volumes) in the wrong role — nothing in
`docs/TODO.md` calls for recording. D is the sharpest distractor: it sounds like
it would support `CLAUDE.md:26-27`, but StatefulSets do not pin pods to nodes,
and stable *ports* are not what the direct-UDP design depends on.

</details>

**Q5.** Front LiDAR reads 20cm while the driver holds full throttle. What
happens?

- A) Motors freeze completely
- B) Max throttle is scaled down, but the rover still moves
- C) Nothing until 10cm
- D) The SBC drops the session and the fleet dashboard marks the rover stuck

<details><summary>✅ Reveal answer</summary>

**B** — `docs/TODO.md:27-30` defines two thresholds: below 30cm scale down max
throttle, at 10cm absolute motor freeze. `README.md:46-49` states the same pair.
At 20cm you are in the scaling band.

A applies the 10cm rule too early; C ignores the 30cm rule entirely. D describes
real machinery — the Grafana auto-kill of `docs/TODO.md:56-57` — but that rule
triggers on ping > 250ms or pack voltage < 3.5V, not on proximity, and it lives
in the cloud rather than on the MCU.

</details>

**Q6.** The firewall on the cloud VM opens TCP 443 and UDP 50000–60000
(`docs/TODO.md:37-38`). What must be opened *inbound* on the rover?

- A) UDP 50000–60000, so the SFU can reach it
- B) TCP 443, so the matchmaker can push room tokens
- C) Nothing
- D) A single UDP port per rover, forwarded on the arena router

<details><summary>✅ Reveal answer</summary>

**C** — nothing. Both the rover and the browser dial *out*; the cloud VM is the
only host with a public address. The rover registers by opening a WSS connection
through the Ingress (`docs/TODO.md:43-45`, `docs/TODO.md:51-53`) and the cloud
replies on that already-open socket.

B is the most tempting, because token push really is cloud→rover
(`docs/TODO.md:51-53`) — but the direction of *dialing* is rover→cloud, and NAT
permits the reply on an existing mapping. A inverts the ICE flow. D is what you
would need for a peer-to-peer design, and is impossible on carrier CGNAT anyway
— one of the reasons the SFU topology of `README.md:75` was chosen.

</details>

**Q7.** Why is video encoding required to run on hardware blocks rather than the
CPU?

- A) The SBC's CPU lacks the instruction set for H.264
- B) Software encode costs ~30ms and competes for CPU, making latency variable
- C) GStreamer only supports hardware encoders
- D) Hardware encoders produce smaller files, saving cellular bandwidth

<details><summary>✅ Reveal answer</summary>

**B** — `README.md:33-35` budgets under 15ms and requires frames never touch the
CPU for compression; `CLAUDE.md:23` states it as an invariant.
`README.md:10-19` names software encoders adding 50ms as one of the stacked
defaults that produce half-second lag. The compounding harm is that a CPU encoder
also contends with the WebRTC stack on the same board, so the cost is *jitter*,
not just delay.

A is false — any ARM CPU can encode H.264 in software; that is exactly the slow
path being refused. C is false; GStreamer supports both (`x264enc` and
`nvv4l2h264enc` alike). D confuses codec efficiency with encoder implementation —
bitrate is a rate-control setting, not a consequence of where encoding runs.

</details>

**Q8.** Which of these is still an **open** decision?

- A) Whether media bypasses the load balancer
- B) Whether guardrails run on the MCU or the SBC
- C) Whether the onboard computer is a Pi 5 or a Jetson
- D) Whether the data channel is unreliable and unordered

<details><summary>✅ Reveal answer</summary>

**C** — `docs/TODO.md:63` still asks "Pi 5 or Jetson for the onboard computer?",
and it is tracked here as D-1. It is the highest-leverage open item because
`README.md:33-35` assumes hardware H.264 encoding exists on whichever board is
chosen, and that assumption needs verification for the Pi 5 before hardware is
ordered.

A, B, and D are all settled invariants, recorded as S-4 (`CLAUDE.md:26-27`),
S-5 (`CLAUDE.md:28-29`), and S-3 (`CLAUDE.md:24-25`). The remaining open
questions in the repo are `docs/TODO.md:63-65` — onboard computer, MCU choice,
and uplink.

</details>

<!-- GEN:quiz END -->
