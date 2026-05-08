# SOP-03: Thread / CoAP / CBOR + Observe

> **Lab guide:** [Lab 3](../lab3.md) — read it first; the API contract, tasks, and DDR deliverables live there.
> **This SOP:** firmware paste, build/flash/test steps, optional dashboard, troubleshooting. Total student-authored C: **two lines** (a forward declaration and a function call in `app_main`).

The Thread mesh from [Lab 2](../lab2.md) is a prerequisite. Two ESP32-C6 boards. ESP-IDF v5.1+. If your CLI rejects `dataset masterkey`, use `dataset networkkey` — they're aliases on older builds.

---

## 1. Project & build configuration

Create the project from the OpenThread CLI example and target the C6:

```bash
idf.py create-project-from-example "$IDF_PATH/examples/openthread/ot_cli" lab03
cd lab03
idf.py set-target esp32c6
```

The default flags from the `ot_cli` example are fine — OpenThread / FTD / CLI on, IPv6 on. No `menuconfig` changes are required for this lab.

> **Two CoAP stacks are in play, and neither is a `menuconfig` toggle.** OpenThread's built-in CoAP API powers the `coap` CLI sub-command on Node B; it ships enabled by default in ESP-IDF v5.1+ via OpenThread's compile-time config (`OPENTHREAD_CONFIG_COAP_API_ENABLE=1` in the OpenThread component's `openthread-core-esp32x-ftd-config.h`). **libcoap** powers the server in `coap_demo.c` on Node A and is added as a managed component via `idf_component.yml`.

Open `main/idf_component.yml` and add `espressif/coap` to the existing `dependencies:` block:

```yaml
dependencies:
  espressif/coap:
    version: "^4.3.0"
  # ... keep the existing entries (esp_ot_cli_extension, idf, ot_led, ot_examples_common)
```

Edit `main/CMakeLists.txt` to add `coap_demo.c` as a source and the components the file uses:

```cmake
idf_component_register(SRCS "esp_ot_cli.c" "coap_demo.c"
                       INCLUDE_DIRS "."
                       REQUIRES esp_event esp_netif nvs_flash openthread vfs
                                ot_examples_common ot_led
                                espressif__esp_ot_cli_extension coap)
```

> The CLI source file may be named `esp_ot_cli.c` or `main.c` depending on your ESP-IDF version. Use whichever is in `main/`.

Run `idf.py reconfigure` once after saving so the component manager fetches libcoap into `managed_components/espressif__coap/` before the next build.

---

## 2. Hook into `app_main` (your two lines)

Open the existing CLI source file. Add at the top, after the includes and before `app_main`:

```c
// Forward declaration — implemented in coap_demo.c
void start_coap_server(void);
```

And add **one line at the very end of `app_main`**, after the existing initialization block (after `esp_openthread_start()` and any `#if CONFIG_OPENTHREAD_*` blocks):

```c
start_coap_server();
```

That is the entire student-authored change to existing files.

---

## 3. Create `main/coap_demo.c`

Paste verbatim. Complete server: CBOR encoding by hand, Observe push, 0.5 °C threshold gate.

```c
#include <string.h>
#include <math.h>
#include <stdint.h>
#include <stdbool.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "esp_random.h"

#include "coap3/coap.h"

static const char *TAG = "coap_demo";
#define COAP_PORT 5683

static float g_current_temp = 24.5f;
static float g_last_notified_temp = 24.5f;
static coap_resource_t *g_env_temp_resource = NULL;

// CBOR encoder for {"t": <float16>} — six bytes.
//   A1            map(1)
//   61 74         text(1) "t"
//   F9 hh ll      float16, big-endian, IEEE 754 half-precision

static uint16_t float32_to_float16(float f)
{
    uint32_t x;
    memcpy(&x, &f, sizeof(x));
    uint32_t sign = (x >> 16) & 0x8000;
    int32_t  exp  = ((x >> 23) & 0xFF) - 127 + 15;
    uint32_t mant = (x >>  13) & 0x3FF;
    if (exp <= 0)   return (uint16_t)sign;                  // underflow → ±0
    if (exp >= 31)  return (uint16_t)(sign | 0x7C00);       // overflow → ±inf
    return (uint16_t)(sign | ((uint32_t)exp << 10) | mant);
}

static size_t encode_env_temp_cbor(float value, uint8_t out[6])
{
    uint16_t h = float32_to_float16(value);
    out[0] = 0xA1; out[1] = 0x61; out[2] = 0x74;
    out[3] = 0xF9; out[4] = (uint8_t)(h >> 8); out[5] = (uint8_t)(h & 0xFF);
    return 6;
}

// /env/temp GET handler — libcoap-3 opaque-PDU API
static void hnd_env_temp_get(coap_resource_t *resource,
                             coap_session_t  *session,
                             const coap_pdu_t *request,
                             const coap_string_t *query,
                             coap_pdu_t      *response)
{
    (void)session; (void)request; (void)query;

    uint8_t buf[6];
    size_t  len = encode_env_temp_cbor(g_current_temp, buf);

    coap_pdu_set_code(response, COAP_RESPONSE_CODE_CONTENT);  // 2.05

    unsigned char encoded[4];
    coap_add_option(response, COAP_OPTION_CONTENT_FORMAT,
                    coap_encode_var_safe(encoded, sizeof(encoded),
                                          COAP_MEDIATYPE_APPLICATION_CBOR),
                    encoded);
    coap_add_data(response, len, buf);

    ESP_LOGI(TAG, "GET /env/temp -> %.2f C (6 B CBOR)", g_current_temp);
}

// Drives Observe notifications. Mocks a slowly drifting sensor;
// in Lab 4 this is replaced by the real ADC reading.
static void temp_update_task(void *arg)
{
    coap_context_t *ctx = (coap_context_t *)arg;
    while (1) {
        float delta = ((float)(esp_random() % 1000) / 1000.0f - 0.5f) * 0.6f;   // ±0.3
        if ((esp_random() % 10) == 0) delta += (esp_random() & 1) ? 1.5f : -1.5f;
        g_current_temp += delta;

        float diff = fabsf(g_current_temp - g_last_notified_temp);
        if (diff > 0.5f) {
            ESP_LOGI(TAG, "Δ=%.2f C exceeds threshold; notifying observers", diff);
            g_last_notified_temp = g_current_temp;
            if (g_env_temp_resource) coap_resource_notify_observers(g_env_temp_resource, NULL);
        }

        coap_io_process(ctx, 0);
        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}

static void coap_server_task(void *pvParameters)
{
    (void)pvParameters;

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

    g_env_temp_resource = coap_resource_init(coap_make_str_const("env/temp"), 0);
    coap_register_handler(g_env_temp_resource, COAP_REQUEST_GET, hnd_env_temp_get);
    coap_resource_set_get_observable(g_env_temp_resource, 1);
    coap_add_resource(ctx, g_env_temp_resource);

    ESP_LOGI(TAG, "CoAP server listening on UDP/%d, resource /env/temp", COAP_PORT);

    xTaskCreate(temp_update_task, "temp_update", 4096, ctx, 4, NULL);
    while (1) coap_io_process(ctx, 1000);
}

void start_coap_server(void)
{
    xTaskCreate(coap_server_task, "coap_server", 6144, NULL, 5, NULL);
}
```

> **API note:** this is the **libcoap-3** opaque-PDU API (`coap_pdu_set_code`, `coap_add_data`, `coap_resource_notify_observers`). Tutorials that access `request->code` directly are libcoap-2 and will not compile on current ESP-IDF.

---

## 4. Build, flash, commission

```bash
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor      # Node A (Server)
idf.py -p /dev/ttyUSB1 flash monitor      # Node B (Client) — same firmware, used as CLI
```

Form the Thread mesh exactly as in [SOP-02](sop02_6lowpan.md). On Node A, after `state` shows `leader`, copy `dataset active -x` and paste it on Node B with `dataset set active <hex>`. Node A's monitor should print:

```
I (4321) coap_demo: CoAP server listening on UDP/5683, resource /env/temp
```

On Node A, grab the address Node B will target:

```
> ipaddr mleid
fd11:22:33:44:0:0:0:1
```

---

## 5. Test from Node B's CLI

Confirm the CoAP CLI is available:

```
> help
... ipaddr ... ifconfig ... coap ... thread ...
```

Then:

```
> coap start
> coap get fd11:22:33:44:0:0:0:1 /env/temp
coap response from fd11:22:33:44:0:0:0:1 with payload: a16174f94e40

> coap observe fd11:22:33:44:0:0:0:1 /env/temp
   ...notifications arrive whenever Node A crosses the 0.5 °C threshold...

> coap cancel fd11:22:33:44:0:0:0:1 /env/temp
```

The hex `a16174f94e40` decodes to `{"t": 24.5}` — that's the contract from [lab3.md §3](../lab3.md#3-the-api-contract--envtemp). Decode by hand or paste into [cbor.me](https://cbor.me).

---

## 6. Packet-size audit (for Task C)

Add this one line inside `hnd_env_temp_get` just before `coap_add_data` if you want the payload length logged:

```c
ESP_LOGI(TAG, "CoAP response payload bytes: %u", (unsigned)len);
```

The total CoAP message size is fixed by the protocol — fill the right column with **your** measured payload:

| Layer | Bytes | Note |
|---|---|---|
| CoAP fixed header | 4 | Ver/T/TKL + Code + Message ID |
| Token | 4 | Default in OpenThread CLI |
| Uri-Path `env` | 4 | Option header (1) + 3 chars |
| Uri-Path `temp` | 5 | Option header (1) + 4 chars |
| Content-Format | 2 | Value 60 fits in one byte |
| Payload marker `0xFF` | 1 | |
| Payload (CBOR) | 6 | From your log |
| **Total CoAP** | **26 B** | |
| UDP header | 8 | |
| IPv6 header (uncompressed) | 40 | After 6LoWPAN IPHC: ~2–6 B |
| **Total over 802.15.4 (compressed)** | **~36 B** | |

Compare this against the HTTP equivalent for the same payload (see the [lecture's](../lectures/lab3_lecture.md) per-packet breakdown — ≥ 6 packets, ~500 B). The deliverable is the ratio.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `coap get` returns `4.04 Not Found` | The CLI strips the leading `/`; the resource is registered as `env/temp`. Path on the wire and registration must match. |
| `coap observe` returns 2.05 once and never again | `coap_resource_set_get_observable(..., 1)` must be called **before** `coap_add_resource`. |
| Notifications stop arriving | Node B left the network. Re-run `coap observe ...`. |
| Build error: `'coap_pdu_t' has no member named 'code'` | Tutorial code is libcoap-2. Use the v3 accessors in §3 above. |
| `coap` not in `help` output | OpenThread was built without `OPENTHREAD_CONFIG_COAP_API_ENABLE` (rare on ESP-IDF v5.1+; default is on). Update ESP-IDF, or set the define in the OpenThread component's `openthread-core-esp32x-ftd-config.h`. |
| `state` stays `detached` on Node B | PANID / channel / network key mismatch. Re-paste `dataset active -x` from Node A; never type by hand. |

---

## Appendix — Optional: a local dashboard (stretch goal)

**This is not required for the lab.** The two `idf.py monitor` terminals already show everything you need to score the rubric. Skip this unless you've finished the deliverables and want to see the same Chart.js view as the Lab 0 / 0.5 dashboards, with live counters for "notifications received" and "1 Hz polls avoided".

The dashboard reads Node B's monitor stream (no firmware change, no Border Router) and decodes each Observe notification's CBOR payload.

**1. Pipe Node B's monitor through `tee`** (replaces the plain `idf.py monitor` you launched in §4):

```bash
idf.py -p /dev/ttyUSB1 monitor | tee /tmp/nodeB.log
```

`tee` preserves the live console you already use *and* writes a copy to the file the dashboard reads. Linux/macOS native; on Windows use Git Bash, WSL, or PowerShell's `Tee-Object`.

**2. In a third terminal:**

```bash
pip install flask
python tools/dashboard_coap.py --log /tmp/nodeB.log
```

Open `http://localhost:5000`. Same Chart.js view as Labs 0 / 0.5; the stats card shows the live notification count and how many polls a 1 Hz client would have spent over the same window.

**Why we keep this optional in Lab 3:** the goal of this lab is to understand the full stack from the CLI — `coap get`, the raw `a16174f9...` bytes, the `notifying observers` log line. Once those are clear, a graphical view adds polish but no insight. We bring the dashboard back as a first-class artifact in Lab 6, where it pairs naturally with secured CoAP (DTLS).
