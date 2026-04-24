# SOP-02: 6LoWPAN + Routing & Resilience

> **Main Lab Guide:** [Lab 2: 6LoWPAN Mesh Networking](../lab2.md)
> **ISO Domains:** RAID (Resource Access & Interchange), SCD (Sensing & Controlling Domain)
> **GreenField Context:** Building self-healing mesh network for "Far Field" coverage

## Objectives
- Enumerate and classify IPv6 addresses (link-local, mesh-local, ML-EID).
- Observe 6LoWPAN fragmentation.
- Evaluate re-attach and role changes (leader, routers, children).

## Context
This implementation guide provides step-by-step technical instructions for 6LoWPAN networking and Thread routing. It complements the [main lab guide](../lab2.md) which covers the "Tractor Test" scenario and stakeholder context.

## Project Setup

### 1. Create Project from ESP-IDF Example

Use the ESP-IDF extension in VS Code:
1. Press `Ctrl+Shift+P` to open the command palette.
2. Search for and run `ESP-IDF: Show Examples` selecting your ESP-IDF version.
3. Select `openthread/ot_cli` (OpenThread CLI Example).
4. Select the folder to create the project.

### 2. Explore the OpenThread CLI Example Code

The `openthread/ot_cli` example includes code for basic Thread CLI. Examine the main files in the created project directory:

- `main/esp_ot_cli.c`: Entry point for the CLI example
- OpenThread components for Thread network handling

### 3. Build and Flash the Project

The example uses default configurations suitable for Thread/6LoWPAN. No changes to sdkconfig are required for this basic lab.

Use the ESP-IDF toolbar in VS Code:
1. Click **ESP-IDF: Set Target** and select `esp32c6`.
2. Click **ESP-IDF: Build Project**.
3. Connect the ESP32-C6 and click **ESP-IDF: Flash Device**.
4. Click **ESP-IDF: Monitor Device**.

### 4. Explore Thread/6LoWPAN CLI Commands

Once in the device console, use the example's CLI commands (all commands are documented in `help`):

```bash
# View full help
help

# View interface status
ifconfig

# View assigned IPv6 addresses
ipaddr

# View multicast addresses
ipmaddr

# View router table
router table

# View neighbor table
neighbor table

# View routes
routes
```

### 5. Basic Thread Network Formation

**Configure Device A (Leader):**
```bash
# Create new network dataset
dataset init new

# Configure channel and PAN ID
dataset channel 15
dataset panid 0x1234

# Configure master key
dataset masterkey 00112233445566778899aabbccddeeff

# Activate dataset
dataset commit active

# Start interface and Thread
ifconfig up
thread start
```

**Configure Device B (Router/Child):**
```bash
# Get dataset from leader (on device A)
# dataset active -x

# Copy the hex dataset to device B
dataset set active <leader_hex_dataset>

# Start interface and Thread
ifconfig up
thread start
```

**Verify network formation:**
- Both devices should join the Thread network
- Use `state` to see roles (leader, router, child)
- Use `ipaddr` to see assigned IPv6 addresses

### 6. IPv6 Address Analysis in 6LoWPAN

**Types of addresses to observe:**

1. **Link-local**: Prefix `fe80::/10`, used for local communication
2. **Mesh-local**: Prefix derived from dataset (e.g., `fd11:22:33::/64`), used within the mesh
3. **ML-EID (Mesh-Local EID)**: Unique device address in the mesh

**Commands for analysis:**
```bash
# View all addresses
ipaddr

# View multicast addresses (including All-Thread-Nodes)
ipmaddr

# View dataset details (to understand prefixes)
dataset active
```

**Analysis:**
- Document each address type and its purpose
- Observe how addresses change when joining different networks
- Compare addresses between devices on the same network

### 7. 6LoWPAN Fragmentation Observation

**Force fragmentation with large pings:**
```bash
# Normal ping (no fragmentation)
ping fd11:22:33:0:0:0:0:1

# Ping with large payload (will force 6LoWPAN fragmentation)
ping fd11:22:33:0:0:0:0:1 size 200

# Even larger ping
ping fd11:22:33:0:0:0:0:1 size 500
```

**Observe in logs:**
- Look for "Fragment" or "Reassembly" messages in the logs
- Note the effective 6LoWPAN MTU (~1280 bytes IPv6, but fragmented at lower layers)
- Measure additional latency due to fragmentation

**Fragmentation analysis:**
- Compare response times between small and large pings
- Document the maximum size without fragmentation
- Observe behavior with multiple hops in the mesh

### 8. Resilience and Re-attach Evaluation

**Simulate leader failure:**
1. Identify which device is the leader (`state` command)
2. Turn off the leader device (disconnect USB)
3. Observe on the remaining device:
   ```bash
   # View state change
   state

   # View re-attach logs
   # (automatic logs will show MLE messages)
   ```

**Measure convergence times:**
- Record timestamp when leader is turned off
- Record when new leader is elected (`state` changes)
- Calculate convergence time

**Additional resilience tests:**
```bash
# View MLE (Mesh Link Establishment) messages
mle

# Force role change (if router)
# (restart device and observe re-attach)

# Test with 3+ devices to see mesh routing
router table
neighbor table
```

**Resilience analysis:**
- Document the sequence of events during re-attach
- Measure convergence times for different network sizes
- Observe how IPv6 addresses are maintained during changes

### 9. Thread Routing Analysis

**Commands for routing analysis:**
```bash
# View router table
router table

# View neighbor table
neighbor table

# View active routes
routes

# View network topology
neighbor table
```

**Routing experiments:**
1. **Direct routing:** Ping between directly connected devices
2. **Mesh routing:** Ping between devices with intermediaries
3. **Routing with failure:** Repeat after simulating failure

**Analysis:**
- Document how the mesh topology is built
- Observe changes in routing tables during resilience tests
- Compare efficiency of direct routing vs mesh

## Deliverables
- Table of IPv6 addresses classified by type
- Fragmentation logs with overhead analysis
- Convergence time measurements for re-attach
- Thread network topology diagrams with roles
- Comparative performance analysis with/without fragmentation

---

## Quick Command Reference (Lab 2 cheat-sheet)

Keep this section open on a second screen during the lab. Commands grouped by [lab2.md](../lab2.md) task.

### Task A — Commissioning

**On Node A (becomes Leader):**
```bash
dataset init new
dataset channel 20
dataset panid 0xBEEF
dataset networkname GreenField-G<team#>
dataset commit active
ifconfig up
thread start

# wait ~10 s:
state              # expect: leader
dataset active -x  # copy the hex blob that's printed
```

**On Nodes B and C:**
```bash
dataset set active <paste hex blob from A>
ifconfig up
thread start

# wait ~10 s:
state              # expect: router or child
```

**Verification (required deliverable):**
```bash
neighbor table     # confirm LQI > 100 for links you care about
ipaddr             # list all IPv6 addresses (classify link-local / ML-EID / RLOC)
router table       # see which nodes became routers
```

### Task B — 3-hop latency

**Step 1. Force A→B→C topology.** Pick one method:

- **Physical (recommended):** separate A and C by a wall or ≥15 m; keep B centered. Verify with `neighbor table` on A — C should be absent or LQI < 80.
- **MAC filter (fallback):**
  ```bash
  # on A and C respectively:
  extaddr                      # note each node's extended address

  # on A:
  macfilter addr denylist
  macfilter addr add <C_extaddr>
  # on C:
  macfilter addr denylist
  macfilter addr add <A_extaddr>
  ```

**Step 2. Get target address (on C):**
```bash
ipaddr mleid       # use THIS (mesh-local EID), not RLOC — stable across topology changes
```

**Step 3. Measure RTT (on A):**
```bash
ping <C_mleid> 64 20 500
# size=64 bytes, count=20 pings, interval=500 ms
```

Read average/min/max RTT from the summary line. Compare to Lab 1's 1-hop RTT.

**Step 4. Confirm the path went through B:**
```bash
router table       # on A — next-hop router toward C's RLOC16
```

### Task C — Tractor test (convergence)

**Start continuous ping on A:**
```bash
ping <C_mleid> 64 200 1000
# 200 pings, 1 per second — gives ~3 minutes of observation
```

**Physically unplug Node B.** Note the timestamp.

**Observe pings timing out, then resuming.** Note the resume timestamp.

**Convergence time = resume_ts − unplug_ts.** Target < 120 s.

**Post-mortem on A:**
```bash
router table       # new next-hop toward C
neighbor table     # B is gone
```

### Supporting commands

| Command | Purpose |
|---|---|
| `state` | My role: leader / router / child / detached |
| `ipaddr` | All IPv6 addresses on this interface |
| `ipaddr mleid` | Just the ML-EID (stable app address) |
| `ipaddr rloc` | Just the RLOC (topology-dependent) |
| `extaddr` | My 802.15.4 extended (EUI-64) MAC address |
| `rloc16` | My 16-bit short address |
| `neighbor table` | Who I can hear, with LQI and avg RSSI |
| `router table` | All routers in the mesh + cost/next-hop |
| `routes` | External routes (empty until Lab 5) |
| `dataset active` | Current network credentials |
| `dataset active -x` | Same, as hex blob for pasting to other nodes |
| `panid` / `channel` | Current PAN ID / channel |
| `thread stop` | Leave the network cleanly |
| `ifconfig down` | Bring radio interface down |
| `factoryreset` | Wipe everything, fresh start |

### Practical notes

1. **Use ML-EID, not RLOC, for pings.** RLOC changes when topology changes; ML-EID is stable. This verifies the puzzle from the lecture.
2. **Always `ifconfig up` before `thread start`.** Common first-time mistake.
3. **If `state` stays `detached` for > 30 s**, check that channel + panid + networkkey match. Use `dataset active -x` on the working node and paste the whole blob into the others.
4. **`ping` argument defaults vary by ESP-IDF version** — always specify size/count/interval explicitly.
5. **If two groups collide on PANID**, you'll see each other in `neighbor table`. Pick a unique PANID (e.g., `0xBE<team#>`).
