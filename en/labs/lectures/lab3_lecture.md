# Lab 3 Lecture: Application Protocol — Why HTTP Kills Batteries

**Duration**: 40 min (delivered before the hands-on lab)

**Audience**: Students about to run Lab 3 (CoAP + CBOR over Thread on ESP32-C6)

**Pairs with**: [lab3.md](../lab3.md)

**Follows**: [Lab 2 Lecture](lab2_lecture.md) — students have a working Thread mesh and know the 80–110 byte payload budget after IPHC.

**Builds on**: [Lab 0 (HTTP)](../../0_2_Minimal_IoT_Implementation_http.md) and [Lab 0.5 (MQTT)](../../0_3_Minimal_IoT_Implementation_mqtt.md) — students have already written an HTTP server with `httpd_uri_t` handlers and JSON responses, and an MQTT client with publish + subscribe. Lab 3 is a third pass at the same problem with a different protocol; lean on that muscle memory.

---

## Learning goals

By the end of the lecture, students should be able to:

1. Explain *quantitatively* why HTTP/JSON is the wrong application protocol for a battery-powered Thread node — in bytes, in radio-on milliseconds, and in handshakes.
2. Decompose a CoAP message into its 4-byte header + token + options + payload, and predict the on-wire size for a given resource path.
3. Choose between **CON** and **NON**, and between **Polling** and **Observe**, given a use case (sensor telemetry vs. actuator command vs. alarm).
4. Encode a small object in CBOR by hand and explain why it beats JSON without being a custom binary format.
5. Locate CoAP and CBOR inside the ISO/IEC 30141 Functional viewpoint — specifically the **ASD** service contract — and contrast against MQTT for non-constrained stacks.

---

## Structure at a glance

| Time | Segment | One-line purpose |
|---|---|---|
| 0–8 min | Recap + ISO placement | Where Lab 3 lives in ASD; what changed from Lab 2. |
| 8–22 min | Thread stack layer of the week: CoAP message, CON/NON, Observe | How a request/response actually crosses the mesh. |
| 22–32 min | Payload layer: CBOR vs JSON; placing CoAP next to HTTP and MQTT (which they already used) | Same data, three answers — they've shipped two already. |
| 32–40 min | Lab bridge | Preview the `/env/temp` resource, the Observe trigger, and the packet-size audit. |

---

## Segment 1 — Recap + ISO placement (0–8 min)

### Callback to last week

Last week we ended with a working Thread mesh that routes IPv6 in **80–110 bytes of payload** after IPHC. Today we're going to spend that payload — and the question is *how cheaply*.

Open with Daniela's two complaints from the lab brief, on the board:

1. *"Batteries die in 4 days."*
2. *"Dashboard takes forever to update."*

Both are application-protocol problems, not radio problems. The radio is fine; we're keeping it on too long, and we're polling when we should be pushing. That's what this lab fixes.

### Why HTTP/JSON is the wrong default — do the arithmetic on the board

You wrote this exact code in [Lab 0](../../0_2_Minimal_IoT_Implementation_http.md):

```c
static esp_err_t sensor_get_handler(httpd_req_t *req)
{
    float temp = 20.0 + (esp_random() % 100) / 10.0;
    char buffer[100];
    snprintf(buffer, sizeof(buffer), "{\"temperature\": %.1f}", temp);
    httpd_resp_set_type(req, "application/json");
    httpd_resp_send(req, buffer, HTTPD_RESP_USE_STRLEN);
    return ESP_OK;
}
```

That handler shipped 10 bytes of useful data per call. Let's count what actually went on the wire to deliver it. On HTTP/TCP/IPv6 over a Thread link:

```
TCP 3-way handshake:        3 packets (~60 B each after IPHC) ─┐
HTTP GET /env/temp + hdrs:  1 packet  (~150 B request line +   │  ≥ 6 packets
                                       Host, User-Agent, etc.) │  before any
HTTP/1.1 200 OK + hdrs:     1 packet  (~120 B status + Content-│  data flows
                                       Type + Content-Length)  │
TCP FIN/ACK teardown:       2 packets (~60 B each)            ─┘
```

Six packets, ~500 bytes of overhead, to carry 10 bytes of payload — and every one of those packets needs the radio on. **The radio is the battery.** ESP32-C6 in Thread RX draws ~75 mA; every extra millisecond of radio-on directly subtracts from battery life.

CoAP collapses this to **one request packet + one response packet**, no handshake, no teardown, no tear-down ACK. Same REST semantics — `GET /env/temp` returns a representation — for roughly an order of magnitude less air time. The handler you'll write today is the structural twin of the Lab 0 one, just bound to a CoAP resource instead of an HTTP URI.

> **First-principles question to drop**: *"HTTP was designed in 1991 for fast wired networks. What three of its assumptions break on a battery-powered radio mesh?"* Expected answers: (1) a TCP connection is cheap to open, (2) headers are free because bandwidth is high, (3) text/ASCII is fine because parsing is cheap. None of these hold on Thread.

### A 2-minute aside: domains are not the whole standard

Before we place Lab 3 on the map — a course-level note. So far we've talked almost exclusively about **the six functional domains** (PED, SCD, ASD, OMD, UD, RAID) because Labs 1–4 are about *building physical and functional layers*, and the domain map is the cleanest way to anchor "where does this code live."

But the domain map is **one diagram**, inside **one viewpoint**, in a much bigger standard. ISO/IEC 30141 organises everything around **six viewpoints**:

| Viewpoint | What it asks | Where you meet it |
|---|---|---|
| **Foundational** | What is an IoT system at all? | Lab 0 — many-to-many digital + physical interaction |
| **Functional** | Where do the system's functions live? | Labs 1–4 — *the domain ladder we've been climbing* |
| **Business** | What value does it create, for whom, at what cost? | Stakeholder tables in every lab |
| **Usage** | Who uses it, in what role, across the lifecycle? | Lab 7 — the health dashboard |
| **Trustworthiness** | Is it safe, secure, reliable, resilient, private? | Lab 6 — DTLS and the §9 audit |
| **Construction** | How is it actually built and deployed? | Lab 8 — the integration recap |

Two things this means:

1. **Labs 1–4 climb the Functional viewpoint's domain ladder** — PED in Lab 1, SCD in Lab 2, ASD today, more ASD with downlink in Lab 4. That's the right narrative *for now*.
2. **From Lab 5 on, the dominant lens changes.** Lab 5 is about the **networking pattern** (Table A.4: proximity / access / services / user networks). Lab 6 is about **Trustworthiness** as a cross-cut. Lab 7 is about the **Usage** viewpoint. Lab 8 wraps the whole system in the **Construction** viewpoint. By the end you should be able to describe SoilSense from any of the six viewpoints, not just by mapping it to domains.

The standard also gives you, beyond viewpoints and domains: **patterns** (Tables A.3–A.5: enterprise system, networking, usage), the **Component capability model** (transducer / data / interface / supporting / latent — you already used this in Lab 0), and the **trustworthiness characteristics list** (availability, confidentiality, integrity, reliability, resilience, safety, privacy). We'll meet each of those in turn.

Why does this matter for today? Because the question "is CoAP SCD or ASD?" only has a clean answer once you know the standard isn't asking you to file every line of code into a bin — it's asking you to describe the same system through several lenses. Today's lens is still **Functional**, and within that, today is **mostly ASD with a foot in SCD**. Two labs from now, the lens changes. Stay tuned.

> **Drawing for the board (climb visualization):** ladder of labs vs. domains, primary domain bolded, foot-in-previous as a thin mark. Labs 1–4 fill the ladder cleanly; Labs 5–8 are sketched as branching off into the other viewpoints.

---

### Where Lab 3 lives in ISO/IEC 30141

Draw the Functional viewpoint domain stack again. Lab 1 sat on the **PED ↔ SCD** boundary (radio waves and hardware). Lab 2 was **mostly SCD with a foot in ASD** — the Thread mesh is the device's communication subsystem (SCD), and stable IPv6 addressing started to make device endpoints reachable to applications (ASD). Lab 3 reverses the weighting: **mostly ASD with a foot in SCD** — the URI, the methods, the CBOR contract, the Observe semantics are squarely application-and-service concerns; the mesh + UDP transport underneath is just the SCD baggage we already paid for in Lab 2. Lab 4 stays mostly ASD (downlink + reliability); from Lab 5 the dominant lens changes (see the aside above).

| Today's deliverable | ISO Functional element |
|---|---|
| The URI path `/env/temp` | ASD service identifier |
| The CBOR schema | ASD data contract |
| GET / Observe semantics | ASD interaction pattern |
| CoAP message + UDP transport | ASD ↔ Network boundary |

The ASD contract you write today is the same contract the dashboard team, the OTA team, and the auditor will all read. **A URI and a schema is what an "API" means in IoT** — there's no Swagger, there's `/.well-known/core` and a CBOR-CDDL schema. We won't formalize the schema today, but Daniela's dashboard people will hold us to it next week.

### Where this is *not* — RAID

RAID (Resource Access & Interchange) is where outside consumers — phones, clouds, third-party integrators — reach the IoT system through public APIs. CoAP today is *internal* to the Thread mesh. The Border Router (Lab 5) is what bridges CoAP↔HTTP for RAID. Mark RAID on the board in gray again so students see the gap they're filling.

### Functional/Management plane separation (ISO §6.2.2.3.3)

The lab brief mentions this — make it concrete. Draw two parallel pipes:

```
Functional plane (today):    [App] ─ CoAP ─ UDP ─ IPv6 ─ 6LoWPAN ─ 802.15.4
Management plane (Lab 2):    Thread MLE, leader election, route propagation
```

The two planes share the radio but not the protocol. You can rip out CoAP and put MQTT-SN in its place without touching MLE. You can change the Leader without touching `/env/temp`. **This is why standards organizations care about layering** — it lets the firmware team and the network team ship independently.

---

## Segment 2 — Thread stack layer of the week: CoAP (8–22 min)

### Bridge from what they've built

Before opening the CoAP spec, map it onto idioms students already wrote in Labs 0 and 0.5. CoAP introduces *no new mental model* — only new bytes on the wire.

| Lab 0 / 0.5 idiom | CoAP equivalent | Same role |
|---|---|---|
| `httpd_uri_t { .uri = "/api/sensor", .method = HTTP_GET, .handler = ... }` | `coap_resource_init("env/temp")` + `coap_register_handler(... COAP_REQUEST_GET, ...)` | Bind a path to a handler. |
| `httpd_resp_send(req, buf, len)` | `coap_add_data_blocked_response(...)` | Send a response body. |
| `cJSON_Parse(...)` / `cJSON_GetObjectItem(...)` | `tinycbor` decoder, or hand-rolled bytes | Parse a request payload. |
| `esp_mqtt_client_subscribe(client, "iot/sensor", 1)` (Lab 0.5) | Client sends `GET /env/temp` with `Observe: 0` | Register interest in a stream of values. |
| Broker delivers `MQTT_EVENT_DATA` to subscriber | Server sends a NON notification to the registered observer | Push a new value to interested party. |

Two structural differences students must internalize:

1. **No broker.** In Lab 0.5, ESP32 connected outward to Mosquitto. In CoAP, there is no broker — every device is both client and server. The Thread mesh *is* the network; observers and observees find each other by IPv6 address.
2. **No persistent connection.** HTTP held a TCP socket open for the request/response. MQTT held one open for the lifetime of the program. CoAP is **datagram-per-message** — each request is one UDP packet, no state at the transport layer.

### Part A: CoAP message format

Draw this on the board. It is the single most important picture of the lecture.

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Ver| T |  TKL  |     Code      |          Message ID           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Token (TKL bytes, 0–8)                                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Options (variable, delta-encoded)                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  0xFF  |  Payload (variable, e.g. CBOR)                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                       4 bytes fixed header
```

| Field | Size | Meaning |
|---|---|---|
| Ver | 2 bits | Always `01` (CoAP v1) |
| T | 2 bits | CON / NON / ACK / RST |
| TKL | 4 bits | Token length, 0–8 |
| Code | 8 bits | `c.dd` form: `0.01` GET, `0.03` PUT, `2.05` Content, `4.04` Not Found |
| Message ID | 16 bits | Duplicate detection on the UDP layer |
| Token | 0–8 B | **Request–response correlation** — independent of Message ID |
| Options | variable | Uri-Path, Content-Format, Observe, ETag, etc. — delta-encoded |
| Payload | variable | Marker `0xFF` then CBOR/JSON/text bytes |

**Two questions students always ask**:

1. *"Why both Message ID and Token?"* Message ID is for the UDP transport (de-dup an ACK against a CON). Token is for the application (correlate "this response is for that request"). Observe needs Token because notifications arrive long after the original request — Message ID would have rolled over by then.
2. *"Why delta-encoded options?"* Two reasons: smaller on the wire, and forces options into a defined order. Servers can parse them in one pass.

### Part B: CON, NON, ACK, RST — reliability without TCP

```
CON (Confirmable) — the receiver MUST ACK
   Client                          Server
      | CON [MID=0x1234, GET]        |
      |----------------------------->|
      | ACK [MID=0x1234, 2.05, data] |   ← piggybacked response
      |<-----------------------------|

NON (Non-confirmable) — fire and forget
   Client                          Server
      | NON [MID=0x1234, GET]        |
      |----------------------------->|
      | NON [MID=0x5678, 2.05, data] |
      |<-----------------------------|
```

The decision matrix. Write this on the board:

| Use case | Type | Why |
|---|---|---|
| Periodic telemetry (every minute) | NON | One drop is fine; the next sample arrives soon. |
| Critical alarm (frost detected) | CON | Must be delivered; retransmit on loss. |
| Actuator command (open valve) | CON | The valve must actually open. |
| Observe notification | NON (default) | Loss is acceptable; the next change re-notifies. |
| Observe notification of *critical* state | CON | RFC 7641 allows occasional CON to verify the registration. |

CoAP's retransmit on CON is exponential backoff with random jitter — the SDK handles this; students don't implement it. But they should know **the timeout is ~2 s, max ~45 s, max 4 retries**. If a server is flapping at 100% loss, CoAP gives up — by design, not by accident.

> **Trap to call out**: NON does not mean "best effort to deliver." It means "do not require an ACK at the message layer." Loss is invisible. If the application needs to know whether the data arrived, that's CON or an application-layer ACK. Don't paper over this distinction in the lab report.

### Part C: Idempotency — why GET/PUT survive a lost ACK

```
PUT /light "1"  → server sets light=1 → ACK lost
                                       → client retransmits
                                       → server sets light=1 again ✓
```

PUT is idempotent: re-applying it lands you in the same state. POST `/counter/inc` is not — re-applying increments twice. **Use PUT for state, POST for events.** This is the same rule as HTTP; the difference is that on a lossy radio, you'll actually hit the lost-ACK case in normal operation, not just in chaos tests.

### Part D: Observe — the energy win

The lab's whole point. Polling burns the radio for nothing when data hasn't changed:

```
Polling — radio on twice per minute, even on a still day
   Client → GET /env/temp → 24.5°C
   (60 s)
   Client → GET /env/temp → 24.5°C  ← same value, paid for the radio anyway
   (60 s)
   Client → GET /env/temp → 24.5°C  ← still
```

Observe (RFC 7641) flips it: register once, server pushes only on change.

```
   Client → GET /env/temp, Observe: 0 → 24.5°C, Observe: 1
   ...
                            (temp drifts to 24.6°C — change < 0.5°C, no notify)
   ...
                            (temp jumps to 26.0°C)
   Server → 2.05 Content    26.0°C, Observe: 2 (NON)
```

The change-threshold logic — `abs(new - old) > 0.5` in Task B — lives in the application, not in CoAP. CoAP just gives you the registration + token + sequence number machinery. The point students must take away: **Observe shifts the policy of "when to talk" from the client (who doesn't know if the data changed) to the server (who does).** That's the energy win, and it's structural, not a code optimization.

The Observe sequence number is 24 bits with wrap-around; the client uses it to drop reordered notifications. If a notification is lost, the next one arrives anyway with a higher seq — there's nothing to repair. Observe is intentionally **eventually consistent**, not lossless.

> **First-principles question to drop**: *"Observe pushes notifications instead of polling. Why doesn't every IoT system just use Observe everywhere?"* Expected answers: (1) the server has to keep state per registered observer (tokens, sequence numbers, addresses), (2) NAT/firewall traversal — push only works if the server can reach the client, which fails the moment a Border Router or cellular path is in the way (this is exactly Lab 4's problem with sleeping nodes), (3) registrations expire and must be refreshed. Observe is a *trade*, not a free lunch.

---

## Segment 3 — Payload + alternative app protocols (22–32 min)

### Part A: CBOR — the payload-layer compression

JSON is ASCII. Every brace, every quote, every digit is a byte. CBOR (RFC 8949) is a **binary** encoding of the same data model — objects, arrays, numbers, strings — using **type-prefixed length-or-value bytes**. The schema-less promise of JSON survives; the verbosity does not.

Walk through `{"t": 24.5}` on the board:

```
JSON:  { " t " :   2 4 . 5 }      → 11 bytes (with space)
       7B 22 74 22 3A 32 34 2E 35 7D

CBOR:  A1 61 74 F9 4E 40
       │  │  │  │  └──┴── 16-bit half-precision float, value 24.5
       │  │  └── ASCII 't'
       │  └── text-string of length 1
       └── map of 1 pair                        → 6 bytes
```

The savings come from three places:

1. **Structure markers are 1 byte**, not punctuation: `A1` = "map of 1", instead of `{` … `:` … `}`.
2. **Numbers are binary**: half-precision float for typical sensor ranges = 2 bytes vs. 4 ASCII digits + decimal point + sign.
3. **Strings carry their length**, no need to scan for closing quote.

Number students should leave with:

| Payload | JSON | CBOR | Ratio |
|---|---|---|---|
| `{"id": "soil-07", "t": 24.5, "rh": 67, "ts": 1715000000}` | 53 B | 31 B | 1.7× |

CBOR is roughly **1.5–2× smaller than JSON** in real telemetry. Combined with CoAP's header savings, the full uplink shrinks from the multi-packet HTTP exchange they wrote in Lab 0 to a single sub-100-byte UDP datagram.

> **First-principles question to drop**: *"CBOR is 1.7× smaller than JSON. Why don't we use a hand-rolled binary format and get 5×?"* Expected: schema evolution, debuggability, tooling. CBOR is a deliberate compromise — small enough for the radio, structured enough for the cloud team to decode without your firmware changelog. Custom binary formats are how Lab 1's "we'll just ship raw `struct`s" plans die six months later.

### Part B: CDDL — describing the schema

You won't write CDDL today, but mention it: **CDDL (RFC 8610)** is to CBOR what JSON Schema is to JSON. It's how you write down the contract:

```
env-reading = {
  t  : float16,        ; temperature in °C
  rh : uint .size 1,   ; relative humidity, 0–100
  ts : uint            ; epoch seconds
}
```

This is the artifact that goes in ADR-003 alongside the URI. Cloud team can generate decoders from it.

### Part C: Placing CoAP next to HTTP and MQTT — three protocols, one course

Students have now shipped working code in all three. Put the comparison table on the board and let them fill in the rows themselves before you reveal yours:

| Aspect | HTTP (Lab 0) | MQTT (Lab 0.5) | CoAP (Lab 3) |
|---|---|---|---|
| Transport | TCP | TCP | UDP |
| Header / message | ~200–500 B | ~2 B fixed + topic | **4 B fixed** |
| Pattern | Request/response | Pub/sub via broker | Request/response **+ Observe** |
| Who initiates | Dashboard polls device | Device connects to broker, then both publish | Either side, peer-to-peer |
| Broker required? | No (direct connection) | **Yes** | No |
| Battery-friendly? | No (TCP + headers) | No (TCP + persistent socket) | **Yes** (UDP + 4-B header) |
| Best fit | Wi-Fi gadgets, browser-facing APIs | Cloud fan-out, Wi-Fi fleets | Constrained mesh (Thread/6LoWPAN) |

The takeaway is not "CoAP wins." The takeaway is **constraints pick the protocol**:

1. The dashboard team in Lab 5 will probably keep using HTTP/MQTT on the Wi-Fi side — that's fine. The Border Router translates at the boundary. RAID is exactly the domain that does this kind of bridging.
2. CoAP is the right choice on the radio side because the radio is the battery, and HTTP/MQTT both keep the radio on for orders of magnitude longer per reading.
3. **Observe is CoAP's answer to MQTT subscribe** — same idea (push-when-changed instead of poll), no broker required.

> **Teaching hook**: "Pick your protocol based on your *constraints*, not your *preferences*. You'll meet engineers who insist MQTT is the One True IoT Protocol. They built fleets on Wi-Fi. You're building one on 250 kbps radios with battery budgets. Different problem, different answer."

---

## Segment 4 — Lab bridge (32–40 min)

### What they are about to do

Walk through [lab3.md](../lab3.md) at high speed:

1. **Task A — `/env/temp` GET, returns CBOR float.** Build on the SOP-03 scaffold, swap the JSON `/sensor` for a CBOR `/env/temp`. The point isn't writing CBOR by hand; it's wiring `tinycbor` (or equivalent) into the response builder and setting `Content-Format: application/cbor` (60).
2. **Task B — Observe with change threshold.** Add the registration handler, store the observer's token + endpoint, push a notification only when `abs(new - old) > 0.5`. The threshold is the policy; the CoAP machinery is the mechanism. Keep them separate in the code.
3. **Task C — Packet-size audit.** Capture with Wireshark on the Border Router host (or log `coap_pdu_get_length`) and *compute* the equivalent HTTP exchange. The deliverable is a number: "CoAP exchange = N bytes, equivalent HTTP = M bytes, ratio = M/N."

### Practical reminders

- **Use the Token, not the Message ID, to match Observe notifications to the registration.** Beginners mix these up; the bug looks like "all my notifications are 'lost'" because the client is matching on Message ID and seeing all-different.
- **CBOR Content-Format is `60`**, not `50` (that's `application/json`). Wrong Content-Format → server returns 4.15 Unsupported Content-Format and the dashboard shows nothing.
- **NON for Observe notifications** by default; CON occasionally to verify the observer is still alive (RFC 7641 §4.5). Don't CON every notification — you'll re-add the ACK round-trip you just removed.
- The `/env/temp` value can be mocked for this lab (a counter, or a sine wave) — the real ADC integration is Lab 4. Don't let students get stuck on hardware.

### The puzzles to seed

Two this week. Don't answer either.

> *"You implemented Observe with a 0.5°C threshold. The temperature drifts 0.4°C every minute for an hour, then jumps 1°C in one second. How many notifications does the server send? What's the worst-case staleness of the dashboard's value?"*

> *"You move the same `/env/temp` server behind the Border Router (Lab 5) so a phone on Wi-Fi can read it. Wi-Fi is fine. Thread is fine. The phone gets a 4.04 Not Found. What changed at the boundary, and which protocol gave up first?"*

The first puzzle is to make students notice that the threshold filters *out* small changes — fine for a slow drift, but they need a fallback "send at least every N minutes" heartbeat for the dashboard to know the device is alive. (That's a recurring gotcha; don't tell them, let it bite once.)

The second puzzle previews Lab 5: Observe registrations don't survive a CoAP↔HTTP proxy unless the proxy explicitly bridges them, and the URI path may need rewriting.

### What Lab 4 will answer

> *"Our sensor side is now efficient. But Edwin says actuator commands fail when the valve is asleep. How do you reliably send a command to a device that's off most of the time?"*

Preview: CoAP CON + retransmit isn't enough when the destination is genuinely unreachable for 10 minutes at a time. We'll need the **Mailbox / Proxy** pattern and the **Sleepy End Device** parent-poll behavior from Lab 2 to come together.

---

## Instructor checklist

- [ ] HTTP-vs-CoAP byte arithmetic visible on the board (≥ 6 packets / ~500 B vs. 2 packets / ~80 B).
- [ ] CoAP 4-byte header bitmap drawn out with all five fixed fields labeled.
- [ ] CON/NON × Polling/Observe decision matrix on the board.
- [ ] CBOR encoding of `{"t": 24.5}` walked through byte by byte.
- [ ] HTTP / MQTT / CoAP three-column table on the board (let students fill it from memory of Labs 0 / 0.5 first).
- [ ] Both puzzles posed and left unanswered at the end.
- [ ] One live demo: a `coap get` from a second node hitting the SOP-03 `/sensor` endpoint, so students see the request/response on `idf.py monitor` before they extend it.

---

## References for students

- [lab3.md](../lab3.md) — the hands-on guide for today.
- [SOP-03: Thread/CoAP Basic](../sops/sop03_coap_basic.md) — the implementation scaffold.
- [5_theory_foundations.md](../../5_theory_foundations.md) §4 — CoAP message format, CON/NON, Observe, idempotency.
- [2_iso_architecture.md](../../2_iso_architecture.md) — Functional viewpoint; ASD service contracts.
- RFC 7252 — CoAP (the one to actually read; skim §3 message format and §4.2 reliability).
- RFC 7641 — Observe option.
- RFC 8949 — CBOR.
- RFC 8610 — CDDL (skim only; for ADR-003).
- ISO/IEC 30141:2024 — Functional viewpoint, §6.2.2.3.3 (functional/management separation).
