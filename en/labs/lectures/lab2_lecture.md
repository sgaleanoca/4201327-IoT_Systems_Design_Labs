# Lab 2 Lecture: Network & Routing — Fitting IPv6 Through a 127-Byte Door

**Duration**: 40 min (delivered before the hands-on lab)
**Audience**: Students about to run Lab 2 (6LoWPAN + Thread mesh on ESP32-C6)
**Pairs with**: [lab2.md](../lab2.md)
**Follows**: [Lab 1 Lecture](lab1_lecture.md) — students have seen the 127-byte frame and the RSSI-vs-distance curve.

---

## Learning goals

By the end of the lecture, students should be able to:

1. Explain why IPv6 cannot ride 802.15.4 unchanged, and what 6LoWPAN does about it (IPHC compression + fragmentation).
2. Classify a Thread node's IPv6 addresses (link-local, mesh-local, ML-EID, RLOC) and predict which one to use for a given packet.
3. Describe the Thread role model (Leader, Router, REED, End Device, SED) and place each role in the Functional viewpoint.
4. Predict the convergence behavior when a Router dies, and justify the < 2-minute healing target from first principles (MLE timers).
5. Contrast Thread's address-based mesh routing with BLE Mesh flooding and Zigbee tree routing — and justify when each wins.

---

## Structure at a glance

| Time | Segment | One-line purpose |
|---|---|---|
| 0–8 min | Recap + ISO placement | Where Lab 2 lives in the Functional viewpoint; what changed from Lab 1. |
| 8–20 min | Thread stack layer of the week: 6LoWPAN + IPv6 addressing + MLE routing | How an IPv6 packet actually moves across a Thread mesh. |
| 20–32 min | Alternative stacks: BLE Mesh (flood) vs Zigbee (tree) | Same mesh problem, three different answers. |
| 32–40 min | Lab bridge | Preview the tractor test and the address-classification task. |

---

## Segment 1 — Recap + ISO placement (0–8 min)

### Callback to last week

Last week we ended on one slide: **80–100 bytes of payload after the 127-byte frame is done**. And a 40-byte IPv6 header eats half of that. Today we pay that bill.

Quick refresher — the 802.15.4 frame the IPv6 packet has to fit into. The **MHR (MAC Header)** is what eats the first ~9 bytes before any payload:

```
802.15.4 MHR (variable, typically 9–23 bytes)
 0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Frame Control (2B)  | Seq # | Addressing  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                                 │
                                 ▼
           ┌─────────────────────────────────────────────┐
           │ Dst PAN ID (2B) │ Dst Addr (2 or 8B)        │
           │ Src PAN ID (0 or 2B) │ Src Addr (2 or 8B)   │
           └─────────────────────────────────────────────┘
```

| MHR field | Size | Purpose |
|---|---|---|
| Frame Control | 2 B | Frame type, addressing mode, security flag, ACK request |
| Sequence Number | 1 B | For ACK matching and duplicate detection |
| Dst PAN ID | 0 or 2 B | Target PAN (omitted on intra-PAN short-address frames) |
| Dst Address | 2 or 8 B | Short (16-bit) or extended (EUI-64) |
| Src PAN ID | 0 or 2 B | Often compressed away |
| Src Address | 2 or 8 B | Short or extended |

So before IPv6 even appears, the MAC layer has already consumed ~9–23 bytes. **Everything after the MHR is where the IPv6 packet lives.** That's the room we're fighting over.

Here's what those 40 bytes actually look like — the standard IPv6 header as defined by RFC 8200:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version| Traffic Class |           Flow Label                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Payload Length        |  Next Header  |   Hop Limit   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               |
+                         Source Address                        +
|                         (128 bits)                            |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               |
+                      Destination Address                      +
|                         (128 bits)                            |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                           40 bytes total
```

Ask the class: *"If IPv6 needs 40 bytes just for the header, and we have ~100 bytes total, how do we make this work?"* Do not answer yet. The two possible answers — **compress** or **fragment** — are what 6LoWPAN (RFC 6282 + RFC 4944) provides. Both.

### Where Lab 2 lives in ISO/IEC 30141

Draw the Functional viewpoint boxes on the board. Lab 1 was about the PED↔SCD boundary (radio waves, hardware). Lab 2 moves one step up the stack — the mesh itself lives inside **SCD**, and stable IPv6 addressing starts to touch **ASD (Application & Service Domain)** because that's how applications in later labs will reach devices.

> **Where RAID comes in**: RAID (Resource Access & Interchange) is where *external* consumers reach the IoT system through APIs and brokers. That's Lab 5 (Border Router), not today. Write RAID on the board in gray so students can see the full domain stack, and mark where we are today vs. where we're going.

### Functional roles — the table we'll fill today

| Thread role | ISO Functional role | Power profile | Radio duty |
|---|---|---|---|
| Leader | Network coordinator | Always on | 100% RX |
| Router | Forwarder | Always on | 100% RX |
| REED (Router-Eligible End Device) | Standby forwarder | Always on | 100% RX |
| End Device (FED) | Leaf, always-on | Always on | 100% RX |
| Sleepy End Device (SED) | Leaf, duty-cycled | Sleep most of the time | ~1% RX (poll interval) |

The **SED** row is the one that matters for the battery budget from Lab 1. Everything else draws ~30 mA continuously — a few hours on 2×AA. Only SEDs hit the 3-month target.

---

## Segment 2 — Thread stack layer of the week: 6LoWPAN + MLE (8–20 min)

### Part A: 6LoWPAN header compression (IPHC)

First, where does the IPv6 header sit inside an 802.15.4 frame? This is the budget we're fighting:

```
802.15.4 frame (127 bytes max, on the wire)
┌──────┬──────────────────────────────────────────────────────────┬─────┐
│ MHR  │ Payload = 6LoWPAN dispatch + IPv6 hdr + next hdr + data  │ FCS │
│ ~9 B │                     ~116 B                               │ 2 B │
└──────┴──────────────────────────────────────────────────────────┴─────┘
                │
                ▼  zoom into payload with UNCOMPRESSED IPv6 + UDP
        ┌──────────┬─────────────┬─────────────┬─────────────┐
        │ IPv6 hdr │ UDP hdr     │ App payload │             │
        │  40 B    │   8 B       │  ≤ 68 B     │             │
        └──────────┴─────────────┴─────────────┴─────────────┘

                ▼  same payload with 6LoWPAN IPHC + NHC
        ┌───┬───┬─────────────┬───────────────────────────────┐
        │IP │UD │ App payload │  (reclaimed room)             │
        │HC │ P │  ≤ 110 B    │                               │
        │2-6│ 2 │             │                               │
        └───┴───┴─────────────┴───────────────────────────────┘
```

The insight of RFC 6282: **most IPv6 header fields are predictable inside a local link**, so don't send them. To see why, look at what's actually in the 40-byte header:

| Field | Size | Typical value in Thread | Compressible? |
|---|---|---|---|
| Version | 4 bits | Always `6` | Elide — always 6 |
| Traffic Class | 8 bits | Usually `0` | Elide when 0 |
| Flow Label | 20 bits | Usually `0` | Elide when 0 |
| Payload Length | 16 bits | Varies | Elide — derivable from 802.15.4 frame length |
| Next Header | 8 bits | Usually UDP (17) | Flag + use NHC for UDP |
| Hop Limit | 8 bits | Usually 1, 64, or 255 | Encode common values in 2 bits |
| Source Address | 128 bits | Link-local, derived from MAC | Elide — regenerate from L2 src |
| Destination Address | 128 bits | Link-local, derived from MAC | Elide — regenerate from L2 dst |
| **Total** | **40 B** | | **→ as little as 2 B** |

Compression targets, in order of savings:

1. **Addresses (32 bytes → as little as 0).** In Thread, the link-local IPv6 address can be derived from the 802.15.4 MAC address. Sender and receiver both know the rule, so neither address needs to be on the wire.
2. **Version / Traffic Class / Flow Label (6 bytes → 0–1).** Always IPv6, usually no traffic class, usually no flow label. Compressible to a bitfield.
3. **Hop Limit (1 byte → 0).** Most packets use 64. When they don't, carry it.
4. **Next Header (1 byte → 0).** If the next header is UDP, flag it with one bit and compress UDP too (NHC).

```
Uncompressed IPv6:      40 bytes
Best-case IPHC:         2 bytes   (link-local, derived addresses, common values)
Typical IPHC in Thread: 4–6 bytes
```

Draw this on the board. Students should leave the session knowing the number **2** — the theoretical floor of IPHC — because it is the number that makes the 127-byte frame livable.

> **First-principles question to drop**: *"6LoWPAN compresses headers but not payload. Why?"* Answer expected: headers are predictable (known syntax, local context), payload is arbitrary. Payload compression is the application's job — that's what CBOR does in Lab 4.

### Part B: When compression isn't enough — fragmentation

Compression gets us back the header bytes, but some packets are still > 127 bytes total. Examples: a DTLS handshake record, an OTA firmware block, a large JSON diagnostic. For those, 6LoWPAN fragments.

```
┌──────────────────────────────────────┐
│ Frag-1: FRAG1 hdr (4B) + first chunk │  ─┐
├──────────────────────────────────────┤   ├─ reassembled by receiver
│ Frag-N: FRAGN hdr (5B) + next chunk  │  ─┘
└──────────────────────────────────────┘

Timeout: 60 seconds. Lose one fragment → entire IP datagram is retransmitted.
```

This is why the Lab 1 foreshadowing matters: **big packets are expensive on lossy radios**. The rule of thumb students should carry forward is "keep application payloads ≤ ~80 bytes after compression and you never fragment." This shapes Lab 4's CoAP/CBOR design.

### Part C: IPv6 addresses a Thread node carries

Students will run `ipaddr` in the lab and see ~5 addresses. They need a mental model for what each one is *for*, not just what it looks like.

| Address | Scope | Used for |
|---|---|---|
| **Link-local** `fe80::/64` | One radio hop | Neighbor discovery, MLE, CSMA peers |
| **Mesh-local EID (ML-EID)** `fdxx::/64` | Entire Thread mesh | Stable, device-identifying address for application traffic |
| **RLOC (Routing Locator)** `fdxx::ff:fe00:xxxx` | Entire Thread mesh | Encodes router ID + child ID; *changes* when topology changes |
| **Multicast** `ff02::1`, `ff03::1` | All-nodes (link-local / realm-local) | Broadcast-like behavior over mesh |

The trap: a student pings by RLOC, topology changes, and pings start failing — not because the network broke, but because the RLOC moved. **Application code should always target the ML-EID.** This is a real source of bugs in production Thread deployments.

### Part D: MLE and how the mesh actually routes

**MLE = Mesh Link Establishment** (RFC 7731-ish, Thread-specific in practice). It's the signaling protocol that does three things:

1. **Neighbor discovery** — who can I hear, and how well (LQI)?
2. **Leader election + Router ID assignment** — one Leader hands out Router IDs (1-byte, so max 32 Routers per network).
3. **Route propagation** — each Router periodically advertises its cost vector to every other Router, using a compact bitfield.

The routing itself is **distance-vector with link-quality weighting**:

```
Cost to destination = base_cost + link_quality_penalty + hop_count
LQI > 200 → penalty 0
LQI 100–200 → penalty 1
LQI < 100 → penalty 4 (bad link, avoid)
```

Draw a 3-node mesh on the board. A → B (LQI 220, cost 1), A → C direct (LQI 80, cost 4), B → C (LQI 210, cost 1). Ask: *"What's A's route to C?"* Answer: A → B → C, total cost 2, beats the direct cost-4 path. This is why Thread can pick a multi-hop path even when a direct link exists — **hops are cheap, bad links are expensive**.

### Why healing completes in < 2 minutes

MLE advertisement period is ~30 s. A Router is declared dead after ~3 missed advertisements → ~90 s detection. Add another ~15–30 s for neighbors to recompute routes and re-advertise. Total: **~120 s ceiling** — which is exactly Edwin's requirement in the lab brief. The number is not magical; it's three MLE intervals plus one route-propagation cycle.

> Tell students: when they measure convergence in Task C, they should expect something in the 60–120 s range. If they get 30 s, great — probably because they had few nodes. If they get > 180 s, something's wrong — check LQI and neighbor tables.

---

## Segment 3 — Alternative stacks: BLE Mesh & Zigbee (20–32 min)

Same architectural problem (multi-hop delivery over constrained radios). Three different answers.

### BLE Mesh: managed flooding

No routing tables. **Every node that hears a message rebroadcasts it** (with a TTL). The network converges by saturation rather than computation.

- **Pros**: no routing state, trivially self-heals (any relay works), excellent for commissioning-light consumer devices (lights, locks).
- **Cons**: bandwidth scales badly — every packet touches every relay. Latency is bounded by TTL, not by topology. No native IP.
- **When it wins**: small dense networks where you don't care about bandwidth (smart lighting in one building).

### Zigbee: tree + mesh hybrid

Addresses are assigned hierarchically by the Coordinator using a **Cskip** formula. Routing follows the tree by default, falls back to AODV-style mesh discovery when the tree fails.

- **Pros**: deterministic address assignment, simple for small networks.
- **Cons**: tree addressing wastes address space, re-parenting is painful, not IP-native (application framework is proprietary).
- **When it wins**: legacy installations; new designs rarely pick Zigbee over Thread.

### The comparison table — write this on the board

| Axis | Thread (IP mesh) | BLE Mesh (flood) | Zigbee (tree+mesh) |
|---|---|---|---|
| **Routing model** | Address-based, distance-vector | Managed flood, TTL-bounded | Hierarchical tree + reactive mesh |
| **State per node** | Router table (≤32 entries) | None (just replay cache) | Routing + neighbor tables |
| **Scalability** | ~250 devices/network, 32 routers | ~100s but bandwidth dies | ~few 100s |
| **IP-native?** | Yes (IPv6 + 6LoWPAN) | No (custom addressing) | No (ZCL application layer) |
| **Self-heal time** | ~60–120 s | Immediate (next flood) | ~30–60 s |
| **Best fit** | Sensor + actuator fleets needing IP | Dense consumer devices, low traffic | Legacy, specific vendor ecosystems |

### Why Thread won for GreenField

Two reasons, stated plainly:

1. **IP-native end-to-end.** A cloud service or a farmer's phone can address a sensor by its ML-EID. No translation layer, no proprietary gateway protocol. This becomes a decisive advantage once the Border Router is in place.
2. **Routing cost scales with topology, not with traffic.** On a 50-sensor field, BLE Mesh floods would saturate the air; Thread's routers forward only along computed paths.

> **Teaching hook**: "Thread didn't invent anything radical. It stapled IPv6 + 6LoWPAN + distance-vector routing to 802.15.4. The value is in the staple, not the parts." This reinforces the ISO/IEC 30141 message: reference architectures are about integration, not invention.

---

## Segment 4 — Lab bridge (32–40 min)

### What they are about to do

Walk through [lab2.md](../lab2.md) at high speed:

1. **Commissioning (Task A)** — form the network with a **non-default PANID**. This matters for trustworthiness: if every lab group uses 0x1234, they commission onto each other's networks.
2. **Address classification** — run `ipaddr` and classify every address into the four categories above. This is the 6LoWPAN deliverable. *(If your handout doesn't list this explicitly, tell students to follow SOP-02 §6.)*
3. **Far-field latency (Task B)** — force 3-hop topology and measure RTT vs Lab 1's 1-hop baseline. Expect 30–100 ms at 3 hops.
4. **Tractor test (Task C)** — continuous ping, unplug the middle Router, measure convergence. Expect 60–120 s.

### The puzzles to seed

Two this week. Don't answer either.

> *"Ping a Thread node by its RLOC. Then force a topology change (restart a Router). Ping the same RLOC. What happens, and why is ML-EID the right address to use for applications?"*

> *"Edwin wants < 2 minutes healing. MLE's advertisement interval is ~30 seconds. Do the math: what's the theoretical minimum healing time, and what happens if we reduce the interval to 5 seconds?"*

Answer to the second one (for your reference, not theirs): lower interval → faster healing, but every Router now broadcasts 6× more often → 6× more air time → 6× less time available for data, and worse SED parent-poll behavior. The 30 s default is a **duty-cycle vs. convergence trade-off**, not an arbitrary choice.

### Practical reminders

- Set the **same channel, PANID, and network key** on all devices. Easiest path: `dataset init new` on Device A, `dataset active -x` to export, paste into B and C with `dataset set active <hex>`.
- When comparing LQI, remember it's a receiver-side measurement. A and B will report different LQIs for the same link.
- Turn radios off when idle. Shared spectrum, shared responsibility — same trustworthiness point from Lab 1.
- If two groups accidentally form one network, they'll see each other in `neighbor table`. That's a PANID collision, not a bug in your code.

### What Lab 3 will answer

> *"We have a mesh that routes IPv6. How do applications talk over it — and why not just use HTTP?"*

Preview: CoAP (RFC 7252) is HTTP's semantics repackaged for constrained networks — UDP, 4-byte headers, native observe/pub-sub. Read the CoAP intro before next class.

---

## Instructor checklist

- [ ] Board ready with the four-address table (link-local / ML-EID / RLOC / multicast).
- [ ] 6LoWPAN compression arithmetic visible (40 → 2 bytes).
- [ ] Distance-vector cost example drawn out (3-node triangle with LQIs).
- [ ] Thread vs BLE Mesh vs Zigbee comparison table on the board during Segment 3.
- [ ] Live demo of `ipaddr` + `neighbor table` on a running board before students start Task A.
- [ ] Both puzzles posed and left unanswered at the end.

---

## References for students

- [lab2.md](../lab2.md) — the hands-on guide for today.
- [SOP-02: 6LoWPAN + Routing & Resilience](../sops/sop02_6lowpan.md) — the address-classification and fragmentation steps.
- [2_iso_architecture.md](../../2_iso_architecture.md) — Functional viewpoint and domains.
- [5_theory_foundations.md](../../5_theory_foundations.md) §2–§3 — deeper first-principles on IPHC and mesh routing.
- RFC 6282 — 6LoWPAN IPHC compression (the one to actually read).
- RFC 4944 — 6LoWPAN fragmentation.
- Thread Specification v1.3 — §4 (MLE), §5 (routing).
- ISO/IEC 30141:2024 — Functional viewpoint, Annex on communication patterns.
