---
layout: post
title: "From Demo to Production Firmware – Step 8: Buffering, Batching, and Data Integrity"
date: 2025-12-27
categories: [embedded, firmware, architecture, data]
---

> **Step 8 is about admitting the network will fail.**  
> Data must survive longer than connectivity.

After Step 7, the system is:

* reliable
* observable
* power-aware

Step 8 improves **robustness under long outages** and **efficiency under good networks** by decoupling **data production** from **data delivery**.

This step answers:

> **"What happens to data when the network is unavailable for a long time?"**

---

## 1. Purpose of Step 8

Without buffering:

* data is lost during outages
* sampling must wait for network
* retry pressure increases

With buffering:

* sensing continues independently
* network can recover later
* sending becomes more efficient

Step 8 is about **temporal decoupling**.

---

## 2. Design Philosophy

Senior rule:

> **"Never let a slow subsystem stall a fast one."**

Here:

* Sensor sampling is fast and predictable
* Network delivery is slow and unreliable

They should not block each other.

---

## 3. What Kind of Buffering (Be Conservative)

For embedded devices, start with:

* Fixed-size ring buffer
* In-RAM only (no flash yet)
* Drop-oldest or drop-newest policy

Avoid at this step:

* flash wear management
* databases
* complex queues

Simplicity first.

---

## 4. Data Model

Define a **single data record**:

```c
struct sample_record {
    int32_t temp_mdeg;
    int64_t timestamp_ms;
};
```

This is the *unit of truth*.

---

## 5. Buffer Context (Ownership)

```c
#define SAMPLE_BUFFER_SIZE 32

struct sample_buffer {
    struct sample_record buf[SAMPLE_BUFFER_SIZE];
    uint8_t head;
    uint8_t tail;
    uint8_t count;
};
```

Ownership rules:

* Buffer is owned by the application
* Only app_run() mutates it
* No concurrent access

---

## 6. Application Context Update

```c
struct app_ctx {
    enum app_state state;
    enum app_state recovery_state;

    uint32_t retry_count;
    k_timeout_t retry_delay;
    int last_error;

    struct sample_buffer samples;

    struct mcp9808_ctx sensor;
    struct net_ctx net;
    struct http_ctx http;
};
```

Temperature is no longer sent immediately.

---

## 7. SAMPLE State Update (Producer)

```c
case APP_STATE_SAMPLE:
    ret = mcp9808_sample(&ctx->sensor, &temp);
    if (ret < 0) {
        ctx->last_error = ret;
        ctx->recovery_state = APP_STATE_SENSOR_INIT;
        ctx->state = APP_STATE_ERROR;
        break;
    }

    buffer_push(&ctx->samples, temp, now_ms());
    ctx->state = APP_STATE_SEND;
    break;
```

Sampling:

* always happens
* never waits for network

---

## 8. SEND State Update (Consumer)

```c
case APP_STATE_SEND:
    if (buffer_empty(&ctx->samples)) {
        ctx->state = APP_STATE_WAIT;
        break;
    }

    rec = buffer_peek(&ctx->samples);
    ret = http_send_record(&ctx->http, rec);
    if (ret < 0) {
        ctx->last_error = ret;
        ctx->recovery_state = APP_STATE_NET_INIT;
        ctx->state = APP_STATE_ERROR;
        break;
    }

    buffer_pop(&ctx->samples);
    ctx->retry_count = 0;
    ctx->state = APP_STATE_SEND; /* try next */
    break;
```

SEND now:

* drains the buffer
* sends multiple records per wake
* stops on failure

---

## 9. Batching Policy

Implicit batching occurs because:

* multiple records are sent in one SEND phase
* one connection may deliver many samples

No extra batching logic required.

---

## 10. Buffer Overflow Policy (Explicit!)

Choose one:

* Drop oldest (recommended)
* Drop newest

Example:

```c
if (buffer_full()) {
    buffer_drop_oldest();
    LOG_WRN("Sample buffer full, dropping oldest");
}
```

Loss is **explicit and logged**, never silent.

---

## 11. Interaction with Power Management

Buffering allows:

* long sleeps during outages
* burst sending when awake

This improves battery life significantly.

---

## 12. What Step 8 Deliberately Avoids

* Flash persistence
* Reordering guarantees
* QoS logic

Those are later, product-specific steps.

---

## 13. Success Criteria for Step 8

Step 8 is complete when:

* Sampling continues during outages
* Buffered data is sent after recovery
* Buffer behavior is predictable
* Data loss (if any) is explicit

---

## 14. Architectural Status After Step 8

After Step 8, the device:

* tolerates long network outages
* uses network efficiently
* separates sensing from delivery

This is common in commercial IoT devices.

---

## 15. Next Steps

Optional next steps:

* Step 9: OTA (MCUboot integration)
* Step 10: Watchdog integration
* Step 11: Flash-backed buffering

Core architecture remains unchanged.
