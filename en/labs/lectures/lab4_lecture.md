# Lab 4 Lecture: Reliable Downlink — Commanding a Sleeping Device

**Duration**: 40 min (delivered before the hands-on lab)

**Audience**: Students about to run Lab 4 (CoAP CON + Sleepy End Device on ESP32-C6)

**Pairs with**: [lab4.md](../lab4.md)

**Follows**: [Lab 3 Lecture](lab3_lecture.md) — students have a working `/env/temp` CoAP/CBOR server, understand Observe, and know the CoAP 4-byte header.

**Builds on**: [Lab 2 Lecture](lab2_lecture.md) — students know the Thread role table (Leader / Router / REED / FED / SED) and the SED's ~1 % radio duty.

---

## Learning goals

By the end of the lecture, students should be able to:

1. Explain why **downlink to a SED is structurally harder than uplink** — and locate exactly where the message waits (parent mailbox, not the air).
2. Decompose CoAP's **CON retransmission schedule** (ACK_TIMEOUT, MAX_RETRANSMIT, ACK_RANDOM_FACTOR) and predict the total wait before a CON gives up.
3. Choose between **CON and NON** for a downlink scenario, with the cost of each spelled out.
4. Reason about **idempotency** at the application layer — why PUT survives a retransmit and POST does not, and why this matters more on radio than on Ethernet.
5. Pick a **SED parent-poll period** that defensibly trades worst-case downlink latency against battery life, with arithmetic.

---

## Structure at a glance

| Time | Segment | One-line purpose |
|---|---|---|
| 0–8 min | Recap + ISO placement | Where Lab 4 sits: still ASD with a foot in SCD. The new angle is the SCD foot (SED). |
| 8–22 min | CoAP CON, retransmit, idempotency | How reliability works without TCP. |
| 22–32 min | Sleepy End Devices: where downlink actually waits | The mailbox model, poll period, energy arithmetic. |
| 32–40 min | Lab bridge | Preview `/act/valve`, the disconnect test, the three poll periods. |

---

## Segment 1 — Recap + ISO placement (0–8 min)

### Callback to last week

Last week we shipped efficient uplink: CoAP/CBOR + Observe, ~36 B per reading, push-on-change instead of poll. Daniela's "batteries die in 4 days" complaint was answered on the uplink side.

This week is the **other half of the API**: commanding the device, not reading from it. Edwin's complaint:

> *"The 'OPEN VALVE' command went out at 06:00. The seedlings dried out anyway. The valve didn't move until 06:14."*

Two questions to put on the board:

1. *"What was the valve doing between 06:00 and 06:14?"* — Sleeping. Radio off. Couldn't hear the command.
2. *"Where did the command actually live for those 14 minutes?"* — Not on the wire. **It was buffered at the SED's parent Router**, waiting to be picked up on the next poll. That place — the parent's mailbox — is the architectural fact this lab is built around.

### The stack you've actually built (and where Lab 5 will reframe it)

Three labs in, students have all seven OSI layers populated for the SoilSense radio side. Worth showing the picture explicitly — it's the cleanest single diagram of "what this course has built so far":

```
                OSI                          SoilSense (today)
   ─────────────────────────────  ────────────────────────────────────────
   Layer 7   Application          ┐
   Layer 6   Presentation         ├─ CoAP (Lab 3, today) · MQTT (Lab 0.5)
   Layer 5   Session              ┘    + CBOR payload presentation
   ─────────────────────────────  ────────────────────────────────────────
   Layer 4   Transport            ── UDP                          ┐
   Layer 3   Network              ── IP routing + 6LoWPAN (Lab 2) ├─ Thread
   ─────────────────────────────  ────────────────────────────────│
   Layer 2   Data Link            ── MAC                          ├─ IEEE
   Layer 1   Physical             ── PHY (Lab 1)                  ┘   802.15.4
```

Two takeaways:

1. **CoAP collapses L5–L7.** That's not a CoAP quirk — it's standard for constrained-device application protocols. MQTT does the same. The presentation layer (CBOR) is a *payload format choice* sitting inside the application byte stream, not a separate header on the wire.
2. **"Thread" is a stack label, not a layer.** It's IEEE 802.15.4 (L1–L2) + 6LoWPAN/IP/UDP (L3–L4) bundled with MLE for management. When someone says "running over Thread," that's what they mean: everything below the CoAP box.

> **Forward hook to Lab 5.** Next week we'll redraw this same picture sideways — not as layers within one device, but as **four networks chained end-to-end**: proximity (Thread, what you see above), access (Wi-Fi/Ethernet from the Border Router), services (CoAP↔HTTP bridge + broker), user (phone/browser). The dominant lens shifts from "where in the OSI stack" to "which network is the packet crossing right now." Same bytes; different framing question.

### Where Lab 4 lives in ISO/IEC 30141

Same domain, same lens as Lab 3: **mostly ASD with a foot in SCD.** The `/act/valve` URI, CBOR `{v: 0|1}` schema, PUT semantics, and the CON delivery contract are all ASD — a new service contract sitting next to `/env/temp`. The new wrinkle is that the **SCD foot is doing real work**: the SED's parent-poll period and the parent Router's per-child mailbox are SCD communication-subsystem mechanisms, and they *visibly* show up in the application's latency budget.

| Today's deliverable | ISO Functional element |
|---|---|
| The URI path `/act/valve` | ASD service identifier |
| The CBOR CDDL schema for `{v: 0|1}` | ASD data contract |
| PUT (CON) semantics + idempotency rule | ASD interaction pattern + reliability |
| SED poll period / parent mailbox | **SCD communication subsystem** |
| The CON retransmit ladder | ASD ↔ Transport boundary policy |

This is the cleanest "the same lab touches two domains, doing different jobs in each" example you'll get all semester. Use it.

### The bigger lens still doesn't change — yet

Brief callback to the Lab 3 aside: the dominant lens is still **Functional viewpoint, climbing the domain ladder.** Lab 5 is where it changes (networking pattern, Table A.4) and Lab 6 changes it again (Trustworthiness as a cross-cut). Until then, "mostly ASD with a foot in SCD" is the right shorthand.

> **Drawing for the board:** two pipes again — *Uplink* (Lab 3, `/env/temp`, easy: SED initiates, radio is already on) and *Downlink* (Lab 4, `/act/valve`, hard: cloud initiates, radio is off). Same Thread mesh underneath, fundamentally asymmetric problem on top.

---

## Segment 2 — CoAP CON, retransmit, idempotency (8–22 min)

### Part A: The four CoAP methods — same verbs as HTTP, sharper rules

Before reliability, names. CoAP keeps HTTP's four core methods — and tightens what each one means, because on a lossy radio you *will* see retransmits, and the verb is what tells the server whether re-applying is safe.

| Method | Code | Semantics | Idempotent? | Safe (no side effect)? | Typical SoilSense use |
|---|---|---|---|---|---|
| **GET** | `0.01` | Read the resource's current representation. | Yes | Yes | `GET /env/temp` (Lab 3), `GET /act/valve` (read state) |
| **POST** | `0.02` | Create a subordinate, append, or invoke a non-idempotent action. | **No** | No | `POST /events` (append a log entry); avoid for "set state" |
| **PUT** | `0.03` | Replace the resource's representation with the request body. | **Yes** | No | `PUT /act/valve {v:1}` (drive a state) |
| **DELETE** | `0.04` | Remove the resource. | Yes | No | Rare on devices; bridge / Border-Router cleanup |

Three rules students should leave with:

1. **GET is safe and idempotent.** Re-fetching never side-effects. CoAP can NON-cache GETs aggressively (proxies, Observe).
2. **PUT is idempotent, not safe.** Side-effects exist (the valve moved), but applying the same PUT twice lands the same state. *This is exactly what makes CON retransmits survivable.*
3. **POST is neither.** "Open the valve" looks like a verb, so beginners reach for POST — but POST a `+1` increment, retransmit it, increment twice. Use PUT for state.

The Observe option from Lab 3 is a GET extension, not a fifth method — same code `0.01`, with an `Observe: 0` option attached at registration time.

> **Two-minute aside students always need**: GET vs PUT vs POST on a constrained device isn't a stylistic choice. It changes what the *protocol itself* will do for you under loss. Pick PUT and the dedup cache + idempotent handler form a safety net. Pick POST and you've opted out — every retry could fire the action again.

### Part A.1: Why TCP-style "reliable connection" is the wrong model here

In Lab 0, HTTP/TCP delivered the command by holding a connection open and retransmitting at the segment level. That works because both endpoints are awake and the cost of keeping the socket open is just RAM.

For SoilSense, both assumptions break:

- **Both endpoints awake?** No. The valve is asleep ~99 % of the time.
- **Socket open is free?** No. TCP keep-alive is the *opposite* of what a battery wants.

CoAP's answer: **no connection at all.** Reliability is at the message layer, per-message, opt-in. You ask for it with CON. You skip it with NON.

```
CON (Confirmable) — receiver MUST ACK at the message layer
   Client                          Valve
      | CON [MID=0x1234, PUT v=1]    |
      |--------(wait 2 s)----------->|   ← if no ACK in ACK_TIMEOUT, retransmit
      | CON [MID=0x1234, PUT v=1]    |     same Message ID → server dedup catches it
      |--------(wait 4 s)----------->|
      | ACK [MID=0x1234, 2.04]       |   ← piggybacked response
      |<-----------------------------|
```

### Part B: The retransmit schedule — predict it before the lab

RFC 7252 §4.8 gives the timers:

| Constant | Default | Meaning |
|---|---|---|
| `ACK_TIMEOUT` | 2 s | First wait before retransmit. |
| `ACK_RANDOM_FACTOR` | 1.5 | Initial wait is uniformly random in `[ACK_TIMEOUT, ACK_TIMEOUT × 1.5]` — jitter to avoid synchronized retransmit storms. |
| `MAX_RETRANSMIT` | 4 | After this many retries, give up. |
| Backoff | × 2 | Each retransmit doubles the wait. |

Compute it on the board:

```
attempt 1: send,        wait 2–3 s   (random in [2, 3])
attempt 2: retransmit,  wait 4–6 s
attempt 3: retransmit,  wait 8–12 s
attempt 4: retransmit,  wait 16–24 s
attempt 5: retransmit,  wait 32–48 s
                         ─────────
                         total worst case ≈ 45 s before timeout
```

Students should leave with this number: **~45 s is the longest a CON can hang on a non-responsive endpoint.** In Lab 4 Task B, when they pull the valve's USB and send a PUT, they should expect roughly that envelope before the client surfaces `timeout`.

> **Trap to call out**: exponential backoff is a *good* match for a lossy radio because losses are correlated in time — if one packet collided, the next one issued 100 ms later probably collides too. Doubling the wait gives the channel time to clear. Constant-interval retransmits would just re-collide.

### Part C: Idempotency — why CON's "send the same MID again" is *safe* for PUT, *not* for POST

```
PUT /act/valve {v: 1}    → server sets valve=1 → ACK lost
                                                → client retransmits same MID
                                                → server's dedup cache catches it,
                                                  replays the same ACK
                                                → valve stays at 1 ✓

POST /counter/inc        → server inc → ACK lost
                                       → client retransmits same MID
                                       → dedup cache catches it ... usually.
                                       → BUT: dedup window is ~247 s (EXCHANGE_LIFETIME).
                                         A retransmit *after* the window expires
                                         increments twice. ✗
```

Two layers of defense, in order:

1. **CoAP's dedup cache** (transport-level). Same `(source-IP, source-port, MID)` arriving within the dedup window → server replays the previously-built response, doesn't re-invoke the handler. This is libcoap-3's default.
2. **Idempotent handler semantics** (application-level). PUT `{v:1}` twice in a row, even across dedup windows, must land the same state. POST is allowed to side-effect — you're on your own.

#### The dedup window, briefly

RFC 7252 §4.8.2 sets the window size to **EXCHANGE_LIFETIME ≈ 247 s** (libcoap rounds to ~120–247 s; some stacks shorten it to save RAM). Why that number?

```
EXCHANGE_LIFETIME = MAX_TRANSMIT_SPAN + (2 × MAX_LATENCY) + PROCESSING_DELAY
                  ≈     45 s          +   (2 × 100 s)     +     2 s         ≈ 247 s
```

- `MAX_TRANSMIT_SPAN` ≈ 45 s — the retransmit ladder you just drew on the board: time from the first CON until the last retransmit is sent.
- `MAX_LATENCY` ≈ 100 s — the worst-case **one-way** trip the spec assumes for the wider Internet. **It appears twice**: once for the request leg (a retransmit could still be in flight) and once for the response leg (the server's ACK could still be in flight back).
- `PROCESSING_DELAY` ≈ 2 s — slack for the server to actually build the response.

The sum is "how long could a duplicate of this exchange plausibly still arrive?" — that's how long the server must remember the `(peer, MID)` pair and the response it sent.

**Three practical consequences:**

- **MIDs are 16 bits.** A server talking to one client can dedup ~65 k exchanges per ~247 s — far more than any SoilSense node will ever do. No collision risk inside the window.
- **The cache costs RAM.** Each entry holds the `(peer, MID, full response PDU)` so a replay is byte-identical. Constrained servers cap it (libcoap defaults are fine for two or three peers; OpenThread's CoAP is tighter). When the cap is hit, the *oldest* entries are evicted — which means a slow retransmit can fall off the back and re-invoke the handler.
- **Cross-window retransmits hit the handler.** If a CON sits in a queue (BR offline for 5 minutes, link partitioned) and finally lands after the window has expired, the dedup cache has forgotten it — so the handler runs *again*. This is exactly why idempotent verbs matter: the transport-level cache is a comfort, not a guarantee.

> **Same rule as HTTP, harder consequences.** On Ethernet you'll hit the lost-ACK case ~never in normal operation. On a 250 kbps radio with retransmit-on-collision, you'll hit it during the lab. The rule is: **use PUT for state, POST for events.** Watch students who try to model "open the valve" as POST instead of PUT — that bug is real and shows up exactly when a stakeholder demos.

### Part D: CON vs NON — the decision matrix, again

Walk this through, **adding the actuator row to last week's matrix**:

| Use case | Type | Why |
|---|---|---|
| Periodic telemetry (`/env/temp`, Observe) | NON | Loss tolerable; next sample arrives soon. |
| Critical alarm | CON | Must arrive; retransmit on loss. |
| **Actuator command (`/act/valve`)** | **CON** | **Must arrive *and* be confirmed**. Stakeholder wants to know the valve actually opened. |
| Status read (`GET /act/valve` from dashboard) | NON usually, CON if "confirm reachability" | If the dashboard wants "is this device alive?" semantics, CON. Otherwise NON. |

The actuator row is the one that drives the lab. CON gives Edwin two things:
1. **End-to-end delivery** — retransmits cover transient loss on the radio.
2. **Confirmation** — the ACK is application proof the valve received the command. Combined with idempotent PUT, this is "fire-and-confirm" semantics over UDP.

> **First-principles question to drop**: *"CoAP gives you CON to make UDP reliable. Why didn't they just put TCP in CoAP?"* Expected: TCP needs the connection state (RAM, ports, keep-alives); TCP retransmits at the segment level, not the request level — so a stuck SYN-ACK can hang you for 60 s before your application sees it; TCP head-of-line blocking on a lossy link can stall *unrelated* requests sharing the connection. CON is *per-message* reliability — exactly the granularity a constrained app wants.

---

## Segment 3 — Sleepy End Devices: where downlink actually waits (22–32 min)

### Part A: Recap of the SED role from Lab 2

From the Lab 2 lecture's role table — only the SED row matters today:

```
SED (Sleepy End Device)
  - MTD child of a Router (its "parent")
  - Radio off ~99 % of the time
  - Wakes every poll_period to ask the parent: "any mail?"
  - All downlink traffic for the SED is buffered AT THE PARENT until poll
```

The parent's mailbox is in 802.15.4 + Thread's MLE; it's not a CoAP concept. CoAP doesn't even know the SED is asleep — it just sees a long network round-trip.

### Part B: The mailbox in the message timeline

Draw this on the board. This is the picture students must internalize:

```
                            poll_period = 5 s
   t = 0      Cloud (Node B): "open valve"  → CON PUT sent
   t = 0.1    Reaches valve's parent Router. Parent buffers it.
              (SED is asleep. Radio off.)
   t = 0.1 .. 5
              Parent holds the message. Burns no air. The valve
              has no idea anything is happening.
   t = 5      Valve wakes, polls parent: "any mail?"
              Parent: "yes, this PUT for you." Hands it over.
   t = 5.01   Valve runs PUT handler, sets GPIO, sends ACK.
   t = 5.02   ACK reaches client.

   Latency = 5 s  ≈  poll_period
```

The thing students *think* will happen — "the CoAP client times out because the radio doesn't see the destination" — does not happen. The parent Router is the SED's stand-in. The radio sees a normal Thread node responding; the latency just looks high.

This is also why the right CON `ACK_TIMEOUT` for a SED route is **at least one poll period**. Default `ACK_TIMEOUT` is 2 s; if your SED polls every 5 s, the client will retransmit *before* the SED has had a chance to wake. The retransmit is harmless (same MID, dedup catches it) but it costs air. Tune `ACK_TIMEOUT` up if you tune `pollperiod` up.

### Part C: The poll-period trade-off

The single most important number in the lab. Three values, three lifestyles:

| `poll_period` | Worst-case downlink latency | Radio duty cycle (poll itself) | Battery life on 2×AA |
|---|---|---|---|
| 1 s   | ~1 s   | ~1 %   | weeks |
| 5 s   | ~5 s   | ~0.2 % | months |
| 30 s  | ~30 s  | ~0.03 % | year+ |

Each poll handshake is roughly **5–10 ms of radio activity at ~75 mA**. The arithmetic:

```
poll_period = 5 s, poll_on_time ≈ 10 ms
average current ≈ (10 ms / 5000 ms) × 75 mA + sleep_current
              ≈ 0.002 × 75 000 µA + 5 µA
              ≈ 155 µA
2 × AA = ~5000 mAh available → ~5 000 000 / 155 ≈ 32 000 h ≈ 3.7 years (in theory)
```

In practice you take a 3–5× haircut for self-discharge, capacitor leakage, MLE overhead, and the actual uplink/Observe traffic from Lab 3. **Realistic = 6–12 months at poll_period = 5 s.** That's the number to put on Edwin's table.

> **First-principles question to drop**: *"poll_period = 1 s gives you 'snappy' valve control. Why not just always pick 1 s?"* Expected: a 5× battery-life hit, a 5× radio-on hit (means a 5× channel-utilization hit per node, which scales badly with fleet size), and — critically — *the dashboard does not need 1 s response*. Manual valve control is human-paced; 5 s is invisible. Engineers default to "snappy" because it feels safer to demo; the cost is paid 6 months later when batteries die early in the field. **Constraints pick the parameter, not preferences** — same teaching hook as Lab 3.

### Part D: When polling isn't enough — Thread CSL and the future of downlink

A two-minute aside. The poll model is fine for human-paced commands (open valve, set thermostat). It is *not* fine if you need millisecond-class downlink to a battery device — alarms, AV sync, locks. Thread 1.2+ introduced **CSL (Coordinated Sampled Listening)**: the parent and child agree on a *synchronized* listen window every N ms, ESP32-C6 supports it, and worst-case latency drops to ~10–50 ms while preserving most of the duty-cycle win.

We do not use CSL in this lab — the menuconfig matrix grows too fast — but mention it. It's the answer to the question "what if 5 s is still too slow?" Don't leave students thinking "SED = 5 s latency, deal with it."

---

## Segment 4 — Lab bridge (32–40 min)

### What they are about to do

Walk through [lab4.md](../lab4.md) at high speed:

1. **Task A — `/act/valve` round-trip.** Same SOP-paste pattern as Lab 3 ([SOP-04 §3](../sops/sop04_sensor_integration.md#3-create-mainvalve_democ)); two student-authored lines. Then `coap put` from Node B, verify the LED follows the CBOR value, verify the ACK code is `2.04 Changed`.
2. **Task B — CON under loss.** Pull the valve's USB *or* `thread stop` on it. Send a PUT. Watch the retransmit ladder in Node B's log: ~2 s, ~4 s, ~8 s, ~16 s, ~32 s, timeout. Replug. Re-send. ACK lands. The annotated log is the deliverable.
3. **Task C — Three poll periods.** Set `pollperiod` to 1 000 / 5 000 / 30 000 ms in turn. Measure round-trip latency for each. Plug into the battery arithmetic ([SOP-04 §7](../sops/sop04_sensor_integration.md#7-poll-period-experiment-task-c)). ADR-004 picks one.

### Practical reminders

- **`coap put ... con tlv 60 a1617601`** is the full client invocation. `con` = CON, `tlv 60` = Content-Format application/cbor, the hex is the 4-byte CBOR payload. Beginners forget the `tlv 60` and get `4.00 Bad Request` because the server can't decode without the Content-Format hint.
- **The valve board needs MTD + Sleepy End Device in `menuconfig`** — not the default FTD. If `state` reports `router`, the board is wrong-configured and Task C will be meaningless.
- **One MID, one logical operation.** The CoAP dedup cache uses the MID; if the client somehow rolls the MID on retransmit, the server will toggle twice. The libcoap CLI does this right, but the trap exists in DIY clients.
- **PUT, not POST.** Drive this home one more time. POST a `+1` increment? Different bug. PUT a state? Safe to retransmit.

### The puzzles to seed

Two this week. Don't answer either.

> *"You set `pollperiod = 30 s` for battery. CoAP's default `ACK_TIMEOUT` is 2 s with MAX_RETRANSMIT = 4 → ~45 s total. The valve polls every 30 s. What's the worst-case ratio of useless retransmits to useful ones, and what happens if every node in a 50-valve fleet hits the same retransmit storm at once?"*

> *"You change `/act/valve` from PUT to POST and from `{v: 0|1}` to `{action: 'toggle'}`. The dashboard team likes it — one resource, one action. What breaks the first time a CON ACK is lost?"*

The first puzzle previews channel utilization at fleet scale — the "fix one knob, break another" lesson. The second is the idempotency point dressed up: a *toggle* command is not safe to retransmit. The fix is either (a) go back to absolute state + PUT, or (b) carry a client-generated request ID and dedup at the application layer. Either way, **the protocol does not save you from a non-idempotent verb**.

### What Lab 5 will answer

> *"Our sensors and actuators all live on the Thread mesh. The dashboard runs on a phone over Wi-Fi. How does CoAP cross that boundary, and what changes about the contracts we've written?"*

Preview: the **Border Router** is the diagram for Lab 5. The networking pattern (Table A.4 — proximity / access / services / user networks) becomes the dominant lens; we stop climbing the domain ladder and start drawing pipes through the four networks. RAID (which has been gray on the board since Lab 1) finally lights up.

---

## Instructor checklist

- [ ] Uplink-vs-downlink asymmetry drawn on the board (radio off when cloud initiates → message waits at parent).
- [ ] CON retransmit ladder computed: 2 s → 4 s → 8 s → 16 s → 32 s → timeout (~45 s).
- [ ] Idempotency rule on the board: **PUT state, POST events.** With the lost-ACK retransmit example for each.
- [ ] CON/NON × use-case matrix on the board (telemetry, alarm, actuator command, status read).
- [ ] SED parent-mailbox timeline drawn — where the message *physically* waits during the poll period.
- [ ] Poll-period × battery table walked through with the 5 s case computed live (~155 µA, ~6–12 months realistic).
- [ ] CSL mentioned once as the answer to "what if 5 s is still too slow?" Don't go deep.
- [ ] Both puzzles posed and left unanswered.
- [ ] Optional live demo: `coap put` from a CLI to a SED with `pollperiod = 5000` so students see the 5 s pause on the timeline before the ACK.

---

## References for students

- [lab4.md](../lab4.md) — the hands-on guide for today.
- [SOP-04: CoAP Downlink & Sleepy End Devices](../sops/sop04_sensor_integration.md) — firmware paste and the poll-period experiment.
- [5_theory_foundations.md](../../5_theory_foundations.md) §4–§5 — CoAP reliability, idempotency, SED parent-poll.
- [2_iso_architecture.md](../../2_iso_architecture.md) — Functional viewpoint; the SCD communication subsystem.
- RFC 7252 §4 — CoAP messaging model, including the §4.8 retransmit parameters (the part to actually read).
- RFC 7252 §5.8.2 — PUT semantics and idempotency.
- Thread Specification v1.3 — §3 (MTD/SED roles), §4 (MLE child polling).
- ISO/IEC 30141:2024 — Functional viewpoint, §6.2.2.3.3 (functional/management separation).
