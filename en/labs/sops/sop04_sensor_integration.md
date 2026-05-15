# SOP-04: CoAP Downlink, CON Reliability & Sleepy End Devices

> **Lab guide:** [Lab 4](../lab4.md) — read it first; the `/act/valve` contract, tasks, and DDR deliverables live there.
> **This SOP:** firmware paste, build/flash/test steps, poll-period experiment, troubleshooting. Total student-authored C: **two lines** (a forward declaration and a function call in `app_main`), exactly as in [SOP-03](sop03_coap_basic.md).

The Lab 3 project (`lab03`, the OpenThread CLI + `coap_demo.c` server) is the starting point. You'll keep its `/env/temp` server running on Node A and extend it with `/act/valve` on a **third board** (the SED valve). The Lab 2 commissioning still applies — same dataset on all three nodes. ESP-IDF v5.1+.

---

## 1. Project & build configuration

Clone the Lab 3 project as `lab04`, or open `lab03` and add a second source file. Either way the only build-time changes are:

1. A new source file `main/valve_demo.c` (full source in §3 below) added to `CMakeLists.txt`.
2. **Sleepy End Device** mode enabled in `menuconfig` for the valve board only.

Edit `main/CMakeLists.txt` to include `valve_demo.c`:

```cmake
idf_component_register(SRCS "esp_ot_cli.c" "coap_demo.c" "valve_demo.c"
                       INCLUDE_DIRS "."
                       REQUIRES esp_event esp_netif nvs_flash openthread vfs
                                ot_examples_common ot_led
                                espressif__esp_ot_cli_extension coap driver)
```

`driver` is added so we can toggle a GPIO for the valve LED.

For the **valve board**, run `idf.py menuconfig` and set:

```
Component config → OpenThread → Device Type → Minimal Thread Device (MTD)
Component config → OpenThread → MTD → Enable Sleepy End Device
Component config → OpenThread → Sleepy End Device polling period → 5000   # ms; Task C varies this
Component config → Power Management → Enable light sleep
```

The Node A (server, `/env/temp`) and Node B (client CLI) boards keep their Lab 3 settings (FTD).

> **One firmware image, three roles.** All three boards run the same build artifact; the role is selected at runtime by what each one's `app_main` calls and what `menuconfig` set for SED. This keeps the build matrix small and matches the SOP-03 pattern.

---

## 2. Hook into `app_main` (your two lines)

Same pattern as [SOP-03 §2](sop03_coap_basic.md#2-hook-into-app_main-your-two-lines). Open the existing CLI source file (`esp_ot_cli.c` or `main.c` in `main/`).

Add at the top, after the includes:

```c
// Forward declaration — implemented in valve_demo.c
void start_valve_server(void);
```

And add one line at the very end of `app_main`:

```c
start_valve_server();
```

Two CoAP servers can't share UDP port 5683 in the same process, so per board you call **exactly one** of the two:

| Board | `start_coap_server()` (Lab 3, `/env/temp`) | `start_valve_server()` (today, `/act/valve`) |
|---|---|---|
| **Node A** (FTD, sensor server) | ✅ keep | ❌ comment out |
| **Node B** (FTD, CLI client)    | ❌ comment out | ❌ comment out (Node B drives via the CLI, runs no server) |
| **Node V** (MTD/SED, valve)     | ❌ comment out | ✅ keep |

The cleanest way to handle this without three separate builds is to wrap both calls in a runtime check on a board-identifying define (`CONFIG_BOARD_ROLE_*`) you set in `menuconfig`, or simply edit the file between flashes. Either approach matches the "one firmware image, three roles" pattern below.

---

## 3. Create `main/valve_demo.c`

Paste verbatim. CoAP server with PUT/GET on `/act/valve`, CBOR by hand, GPIO toggle, idempotent state machine.

```c
#include <string.h>
#include <stdint.h>
#include <stdbool.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "driver/gpio.h"

#include "coap3/coap.h"

static const char *TAG = "valve_demo";
#define COAP_PORT       5683
#define VALVE_GPIO      8           // on-board LED on most ESP32-C6 dev boards

static uint8_t g_valve_state = 0;   // 0 = closed, 1 = open
static coap_resource_t *g_valve_resource = NULL;

// CBOR helpers — payload is always exactly 4 bytes: A1 61 76 0X
//   A1     map(1)
//   61 76  text(1) "v"
//   0X     unsigned 0 or 1

static bool decode_valve_cbor(const uint8_t *buf, size_t len, uint8_t *out_v)
{
    if (len != 4)                                     return false;
    if (buf[0] != 0xA1)                               return false;  // map(1)
    if (buf[1] != 0x61 || buf[2] != 0x76)             return false;  // "v"
    if (buf[3] != 0x00 && buf[3] != 0x01)             return false;  // 0 or 1
    *out_v = buf[3];
    return true;
}

static size_t encode_valve_cbor(uint8_t v, uint8_t out[4])
{
    out[0] = 0xA1; out[1] = 0x61; out[2] = 0x76; out[3] = v ? 0x01 : 0x00;
    return 4;
}

static void apply_valve_state(uint8_t v)
{
    gpio_set_level(VALVE_GPIO, v);
    g_valve_state = v;
    ESP_LOGI(TAG, "valve -> %s (GPIO%d = %d)", v ? "OPEN" : "CLOSED", VALVE_GPIO, v);
}

// PUT handler — idempotent. Same input value twice = same state, ACK both times.
static void hnd_valve_put(coap_resource_t *resource,
                          coap_session_t  *session,
                          const coap_pdu_t *request,
                          const coap_string_t *query,
                          coap_pdu_t      *response)
{
    (void)resource; (void)session; (void)query;

    size_t       len = 0;
    const uint8_t *data = NULL;
    coap_get_data(request, &len, &data);

    uint8_t v;
    if (!decode_valve_cbor(data, len, &v)) {
        ESP_LOGW(TAG, "PUT /act/valve: malformed CBOR (%u B)", (unsigned)len);
        coap_pdu_set_code(response, COAP_RESPONSE_CODE_BAD_REQUEST);  // 4.00
        return;
    }

    apply_valve_state(v);
    coap_pdu_set_code(response, COAP_RESPONSE_CODE_CHANGED);          // 2.04
}

// GET handler — returns current state in the same CBOR shape.
static void hnd_valve_get(coap_resource_t *resource,
                          coap_session_t  *session,
                          const coap_pdu_t *request,
                          const coap_string_t *query,
                          coap_pdu_t      *response)
{
    (void)resource; (void)session; (void)request; (void)query;

    uint8_t buf[4];
    size_t  len = encode_valve_cbor(g_valve_state, buf);

    coap_pdu_set_code(response, COAP_RESPONSE_CODE_CONTENT);          // 2.05

    unsigned char encoded[4];
    coap_add_option(response, COAP_OPTION_CONTENT_FORMAT,
                    coap_encode_var_safe(encoded, sizeof(encoded),
                                          COAP_MEDIATYPE_APPLICATION_CBOR),
                    encoded);
    coap_add_data(response, len, buf);

    ESP_LOGI(TAG, "GET /act/valve -> %u", g_valve_state);
}

static void valve_server_task(void *pvParameters)
{
    (void)pvParameters;

    gpio_config_t io = {
        .pin_bit_mask = 1ULL << VALVE_GPIO,
        .mode = GPIO_MODE_OUTPUT,
    };
    gpio_config(&io);
    apply_valve_state(0);

    coap_address_t addr;
    coap_address_init(&addr);
    addr.addr.sin6.sin6_family = AF_INET6;
    addr.addr.sin6.sin6_port   = htons(COAP_PORT);
    addr.addr.sin6.sin6_addr   = in6addr_any;

    coap_set_log_level(COAP_LOG_WARN);
    coap_context_t *ctx = coap_new_context(NULL);
    if (!ctx) { ESP_LOGE(TAG, "coap_new_context failed"); vTaskDelete(NULL); return; }
    coap_context_set_block_mode(ctx, COAP_BLOCK_USE_LIBCOAP);

    if (!coap_new_endpoint(ctx, &addr, COAP_PROTO_UDP)) {
        ESP_LOGE(TAG, "coap_new_endpoint failed");
        coap_free_context(ctx); vTaskDelete(NULL); return;
    }

    g_valve_resource = coap_resource_init(coap_make_str_const("act/valve"), 0);
    coap_register_handler(g_valve_resource, COAP_REQUEST_PUT, hnd_valve_put);
    coap_register_handler(g_valve_resource, COAP_REQUEST_GET, hnd_valve_get);
    coap_add_resource(ctx, g_valve_resource);

    ESP_LOGI(TAG, "CoAP server listening on UDP/%d, resource /act/valve", COAP_PORT);
    while (1) coap_io_process(ctx, 1000);
}

void start_valve_server(void)
{
    xTaskCreate(valve_server_task, "valve_server", 6144, NULL, 5, NULL);
}
```

> **API note:** still **libcoap-3** opaque-PDU API. CON vs NON is a *client* choice — the server replies CON with a piggybacked ACK either way. Nothing in the handler distinguishes CON from NON. See [SOP-03 §3 API note](sop03_coap_basic.md#3-create-maincoap_democ).

---

## 4. Build, flash, commission

```bash
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor      # Node A — /env/temp server (Lab 3 firmware untouched)
idf.py -p /dev/ttyUSB1 flash monitor      # Node B — client CLI
idf.py -p /dev/ttyUSB2 flash monitor      # Node V — valve, SED, /act/valve server
```

Form the Thread mesh exactly as in [SOP-02](sop02_6lowpan.md): dataset on Node A, paste on Node B and Node V. Node V should report `state child` (not `router`) because it's an MTD/SED. On Node V's monitor:

```
I (5012) valve_demo: valve -> CLOSED (GPIO8 = 0)
I (5024) valve_demo: CoAP server listening on UDP/5683, resource /act/valve
```

Grab the valve's mesh-local EID — this is your target address:

```
> ipaddr mleid
fd11:22:33:44:0:0:0:3
```

---

## 5. Test from Node B's CLI

```
> coap start
> coap put fd11:22:33:44:0:0:0:3 /act/valve con tlv 60 a1617601
   ...wait up to one SED poll period...
coap response from fd11:22:33:44:0:0:0:3 with code 2.04

> coap get fd11:22:33:44:0:0:0:3 /act/valve
coap response from fd11:22:33:44:0:0:0:3 with payload: a1617601

> coap put fd11:22:33:44:0:0:0:3 /act/valve con tlv 60 a1617600
coap response from fd11:22:33:44:0:0:0:3 with code 2.04
```

Verify on Node V's monitor that the `valve -> OPEN` / `valve -> CLOSED` lines appear in order, and that the LED on GPIO 8 follows. `tlv 60` sets `Content-Format: 60` (application/cbor); `con` selects CON over NON.

The hex `a1617601` decodes to `{"v": 1}` — the contract from [lab4.md §3](../lab4.md#3-the-api-contract--actvalve).

---

## 6. CON reliability under loss (Task B)

On Node V, simulate the radio being unreachable — pull the USB cable, **or** type `thread stop` to keep the logs but kill the radio.

On Node B:

```
> coap put fd11:22:33:44:0:0:0:3 /act/valve con tlv 60 a1617601
```

Watch Node B's log. With default CoAP timers (RFC 7252 §4.8), the original CON is followed by **4 retransmits** spaced ~**2 s → 4 s → 8 s → 16 s** apart (each interval doubles, with ±50 % jitter from `ACK_RANDOM_FACTOR = 1.5`). After the 4th retransmit the client waits one last ACK window (up to ~32 s) before surfacing `timeout`. Total envelope ≈ 45 s — the `MAX_TRANSMIT_SPAN` from the lecture. Annotate those intervals in your DDR.

Reconnect Node V (`thread start` or replug). Re-issue the same `coap put`. The valve opens, the ACK comes back.

> **What you're seeing.** CoAP's CON retransmit lives at the message layer, not the application layer. The same Message ID is on the wire each time. When Node V finally re-receives the request, its libcoap dedup cache short-circuits the duplicate so the valve doesn't toggle twice — this is one half of why idempotency matters. The other half is your handler: PUT `{v:1}` twice = open, open. Safe.

---

## 7. Poll-period experiment (Task C)

The SED's parent-poll period controls **how long a downlink message sits in the parent's mailbox** before the valve drains it. Worst-case downlink latency ≈ one poll period (plus mesh hop time, negligible here).

For each value in the table below, change `OPENTHREAD_CONFIG_MAC_POLL_PERIOD` via `menuconfig` *or* at runtime on Node V:

```
> pollperiod 1000          # 1 s
> pollperiod 5000          # 5 s
> pollperiod 30000         # 30 s
```

Then from Node B, send a PUT and measure the wall-clock latency from PUT issued → `2.04 Changed` received:

```
> coap put fd11:22:33:44:0:0:0:3 /act/valve con tlv 60 a1617601
```

| `poll_period` | Measured worst-case latency | Notes |
|---|---|---|
| 1 s   | ~1 s   | snappy, but radio wakes 60×/min |
| 5 s   | ~5 s   | default; good balance |
| 30 s  | ~30 s  | best battery, awful UX for manual commands |

Energy estimate per poll cycle: poll handshake ≈ 5–10 ms of RX/TX activity on ESP32-C6 at ~75 mA. At 5 s poll, the SED averages roughly `10 ms / 5000 ms = 0.2 %` radio duty cycle → ~150 µA average → months on 2×AA. At 1 s poll, ~1 % duty → ~750 µA → weeks. The arithmetic — and the picked value — goes in ADR-004.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `coap put` returns timeout even though valve is up | `pollperiod` is large; CoAP gave up before the SED woke. Lower `pollperiod` or raise the CoAP client's `ACK_TIMEOUT`. |
| Valve toggles twice on a single PUT | The handler has a side effect outside `apply_valve_state`, *or* a retransmit arrived *after* the dedup cache expired (RFC 7252 EXCHANGE_LIFETIME ≈ 247 s — see the lecture). Idempotency of PUT is the safety net; check that `apply_valve_state` is the only place mutating state. |
| `4.00 Bad Request` on every PUT | Content-Format option missing or wrong. The CLI argument must be `tlv 60`, not `tlv 50` (that's JSON). |
| Valve becomes a Router instead of a child | `menuconfig` left Device Type as FTD. Switch to MTD + Sleepy End Device and re-flash. |
| `pollperiod` not recognized | Older ESP-IDF. Use `csl period` / `csl timeout` on v1.3 Thread, or set the value at build time in `menuconfig`. |
| LED never turns on but logs say `valve -> OPEN` | GPIO 8 is not the LED on your board variant. Check the schematic; common alternates are GPIO 7, 15. |

---

## Appendix — Optional: end-to-end soak test (stretch goal)

**Not required for the lab.** If you've finished the deliverables and want to see what "real" downlink load looks like, run this loop from Node B for an hour and graph the latencies:

```bash
# pseudocode — wrap your CLI driver of choice
while true; do
  t0=$(date +%s%N)
  coap put fd11:22:33:44:0:0:0:3 /act/valve con tlv 60 a161760$((RANDOM % 2))
  t1=$(date +%s%N)
  echo $(( (t1 - t0) / 1000000 )) >> latencies.txt
  sleep $((RANDOM % 10 + 5))
done
```

Plot `latencies.txt` as a histogram. You should see a bimodal distribution: a tight peak near *zero* poll wait (you got lucky and hit the SED just as it polled) and a broader peak near *one poll period* (you missed and waited for the next). The dashboard treatment of this — alongside the `/env/temp` Observe panel — comes back in Lab 6 with DTLS.
