# Lab 4: Reliable Downlink & Actuator Control (CoAP CON)
> **Technical Guide:** [SOP-04: CoAP Downlink & Sleepy End Devices](sops/sop04_sensor_integration.md) — firmware paste, build steps, troubleshooting.
> **Lecture:** [lab4_lecture.md](lectures/lab4_lecture.md)

**GreenField Technologies — SoilSense Project**

**Phase:** Control Logic

**Duration:** 3 hours

**ISO Domains:** ASD (Application & Service), SCD (Sensing & Controlling)

---

## 1. Project Context

**From:** Edwin (Field Ops) via Daniela — *"The 'OPEN VALVE' command went out at 06:00. The seedlings dried out anyway. The valve didn't move until 06:14."*

You shipped efficient *uplink* in [Lab 3](lab3.md) — CoAP/CBOR + Observe. **Downlink is harder.** The valve node is a Sleepy End Device (SED): radio off ~99 % of the time. A GET/PUT addressed to it lands on its parent Router, and waits there until the SED wakes up to poll. Without CON's retransmit-on-loss, transient radio errors during that hand-off leave the client with no way to know whether the valve actually moved. **Mission:** add a CoAP `/act/valve` resource, drive it with **CON (Confirmable)** requests, and pick a poll period that trades latency for battery defensibly.

| Stakeholder | Their question | How this lab answers |
|---|---|---|
| **Edwin (Ops)** | How do I command a sleeping valve and *know* it opened? | CON + ACK gives end-to-end delivery confirmation. |
| **Daniela (Farmer)** | Why does the valve lag 30 s after I tap "open"? | You tune the SED's parent-poll period — that's the lag. |
| **ISO 30141 Auditor** | Is the actuation reliable? | CON retransmission, idempotent PUT, and a documented `/act/valve` contract in ASD. |

---

## 2. ISO/IEC 30141 placement

```mermaid
graph TD
    subgraph ASD [Application & Service Domain]
        Valve[CoAP /act/valve<br/>CBOR {v: 0|1}]
        Reliable["CON request → ACK<br/>retransmit on loss"]
    end
    subgraph SCD [Sensing & Controlling Domain]
        SED[Sleepy End Device<br/>poll period = X s]
        Parent[Parent Router<br/>holds mailbox]
    end
    Valve --> Reliable
    Reliable -.->|stored at parent until poll| Parent
    Parent -.->|delivered on next poll| SED

    style ASD fill:#f9f,stroke:#333
    style SCD fill:#bbf,stroke:#333
```

**Still mostly ASD with a foot in SCD.** Lab 3 added the uplink contract on ASD; today's downlink contract — `/act/valve`, PUT semantics, CBOR `{v: 0|1}`, CON delivery — is also ASD. The **SED poll period and the parent-mailbox buffering** are SCD: they belong to the device's communication subsystem, not to the application.

**Functional / management plane separation (ISO §6.2.2.3.3):** CoAP CON retransmits live on the functional plane; Thread MLE keeps the SED↔parent attachment alive on the management plane. The valve can re-parent (MLE) mid-flight without losing your in-flight CON (the new parent re-receives the retransmit).

---

## 3. The API contract — `/act/valve`

This is the artifact you cite in ADR-004. The firmware in [SOP-04 §3](sops/sop04_sensor_integration.md#3-create-mainvalve_democ) produces exactly this.

| | |
|---|---|
| **Resource** | `/act/valve` |
| **Transport** | CoAP / UDP / port 5683 |
| **Methods** | `PUT` (CON), `GET` (CON or NON) |
| **Content-Format** | `60` (`application/cbor`) |
| **Idempotency** | PUT same value twice → same state, ACK each time, no side effect on the second |
| **Reliability policy** | CON; ACK_TIMEOUT = 2 s, MAX_RETRANSMIT = 4, ACK_RANDOM_FACTOR = 1.5 → max wait ≈ 45 s before client gives up |

**Payload (4 bytes either direction):**

```
A1            map(1)
61 76         text(1) "v"
0X            unsigned 0 or 1   ; CBOR small-uint
```

| `v` | Wire bytes |
|---|---|
| 0 (closed) | `A1 61 76 00` |
| 1 (open)   | `A1 61 76 01` |

**CDDL schema (RFC 8610):**

```
valve-cmd = {
  v: 0 / 1   ; 0 = closed, 1 = open
}
```

**Response codes:** `2.04 Changed` on PUT success (piggybacked in the ACK); `2.05 Content` on GET; `4.00 Bad Request` if CBOR doesn't decode to `{v: 0|1}`; `4.05 Method Not Allowed` for anything but PUT/GET.

---

## 4. Execution

The firmware is provided in full — [SOP-04 §1–§3](sops/sop04_sensor_integration.md). Paste, build, flash three boards — **Node A** (Lab 3 `/env/temp` server, FTD), **Node B** (CLI client, FTD), **Node V** (the valve, MTD/SED running `/act/valve`) — and commission them onto the Thread mesh from Lab 2. **You author no C beyond the same two declaration lines as Lab 3.** Evaluation is done from the three `idf.py monitor` terminals.

### Task A — `/act/valve` round-trip
- From Node B's CLI: `coap put <valve_mleid> /act/valve con tlv 60 a1617601` (sets `v=1`).
- Confirm the valve's LED turns on **and** Node B's log shows `2.04 Changed`.
- `coap get <valve_mleid> /act/valve` — payload should come back as `a1617601`.
- Send `a1617600` (PUT, sets `v=0`). Confirm LED off + `2.04 Changed` + GET returns `a1617600`.
- **Evidence:** Node B log (PUT + GET pairs) + Node V log + LED photo / state observation.

### Task B — CON reliability under loss
- Disconnect Node V (pull USB or run `thread stop` on it).
- From Node B: `coap put <valve_mleid> /act/valve con tlv 60 a1617601`.
- **Observe** the retransmission schedule in Node B's log: 4 retransmits at roughly **2 s → 4 s → 8 s → 16 s** intervals (each doubled, with ±50 % jitter); the client then waits one more ACK-window before surfacing `timeout` (total envelope ≈ 45 s).
- Plug Node V back in (`thread start` or replug). Re-issue the PUT. Verify the ACK lands.
- **Evidence:** annotated Node B log showing the four retransmit intervals and final timeout, then the successful ACK after recovery.

### Task C — SED poll period vs latency (the headline trade-off)

The valve is a **Sleepy End Device**. It wakes every `poll_period` seconds, asks its parent "any mail?", drains the mailbox, processes, sleeps. The poll period sets the **worst-case downlink latency** and the **dominant energy cost**.

Run the same `coap put` three times with three poll periods and fill in the table from your own measurements ([SOP-04 §7](sops/sop04_sensor_integration.md#7-poll-period-experiment-task-c) walks the steps):

| `poll_period` | Worst-case latency (your measurement) | Estimated radio-on duty cycle | Estimated battery life on 2×AA |
|---|---|---|---|
| **1 s**   | ____ s | ____ % | ____ days |
| **5 s**   | ____ s | ____ % | ____ days |
| **30 s**  | ____ s | ____ % | ____ days |

Use ESP32-C6 RX current ≈ 75 mA and sleep current ≈ 5 µA. Energy per poll = `t_radio_on × I_rx`; one AA ≈ 2500 mAh; budget is 2 AA in series. Show the arithmetic.

**Deliverable in your DDR:** the table above with all three rows filled in, plus one sentence picking the value Edwin should run in the field and why.

---

## 5. Deliverables — DDR updates

Update [your DDR](../3_deliverables_template.md):

- **§2 Lab Log → "Lab 4: Downlink & CON" → To Edwin.** Two short paragraphs: did the valve respond reliably under loss, and what poll period buys what battery life.
- **§3 ADR-004: Poll period for SED actuators.** Context (Edwin's lag complaint, Daniela's battery complaint), decision (the period you picked), rationale (cite Task C numbers), status. Explicitly state the latency *and* battery numbers behind the choice.
- **§4 ISO Mapping.** Add `/act/valve` and the CDDL schema as ASD entries; mark **CON retransmission** as an ASD reliability mechanism and **SED parent-poll** as an SCD communication-subsystem mechanism.
- **§5 First Principles, Lab 4.** One sentence each: why PUT is safe to retransmit but POST is not; why CON's exponential backoff matches a lossy radio better than a fixed retry; why the SED poll period is a *policy*, not a CoAP parameter.
- **§6 Performance Baselines.** Fill in the "Lab 4: Downlink Latency" row with your chosen poll period's worst-case from Task C — target < 30 s at 5 s poll.
- **§7 Ethics & Sustainability.** Safety: what happens if the valve receives "OPEN" but the network dies before "CLOSE"? Design a failsafe (timeout on the actuator side, watchdog, default-closed). Sustainability: a 1 s poll buys you snappy response and ~6× shorter battery life — defensible only if a stakeholder needs it.
- **Energy calculation.** Extend your Lab 3 energy spreadsheet with the SED poll cost. At your chosen poll period, what's the dominant term — uplink Observe traffic (Lab 3), or the bare poll itself? Use the radio-RX current from [references.md](../references.md).

---

## Grading rubric (100 pts)

**Technical execution (40)** — `/act/valve` PUT/GET works (10) · CON retransmit + ACK verified under disconnect (15) · Three poll periods measured (15)

**ISO/IEC 30141 alignment (30)** — ASD downlink contract documented (15) · ASD/SCD split for reliability vs poll mechanism explained (15)

**Analysis (20)** — ADR-004 justification with latency *and* battery numbers (10) · Idempotency reasoning in context (10)

**Ethics (pass/fail)** — Safety: failsafe for "OPEN then network dies" described · Sustainability: the poll-period choice is defensible in stakeholder terms, not just "felt right"
