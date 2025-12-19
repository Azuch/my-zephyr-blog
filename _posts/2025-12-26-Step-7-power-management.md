---
layout: post
title: "From Demo to Production Firmware – Step 7: Power Is a First-Class Constraint"
date: 2025-12-26
categories: [embedded, firmware, architecture, power]
---

# Step 7 – Power Management (Make WAIT Meaningful)

After Step 6, the system is:

* reliable
* observable
* polite under failure

Step 7 makes it **energy-aware**.

This step answers:

> **"What should the device do while nothing useful is happening?"**

---

## 1. Purpose of Step 7

Most of the device’s lifetime is spent in:

```
WAIT
IDLE
```

If power management is not addressed:

* batteries drain quickly
* devices run hot
* energy budgets are unpredictable

Step 7 ensures:

* WAIT actually saves power
* retries do not waste energy
* power behavior is explainable

---

## 2. Power Management Philosophy

Senior firmware rule:

> **"Sleep whenever you are not making progress."**

Important clarifications:

* Power management is a *policy*, not a driver feature
* WAIT is the only state allowed to sleep deeply
* Other states must remain responsive

---

## 3. Power Domains (Mental Model)

Think in terms of power domains:

| Domain      | Examples           |
| ----------- | ------------------ |
| CPU         | active vs idle     |
| Radio       | Wi-Fi / BLE on/off |
| Peripherals | I2C, sensors       |

Step 7 coordinates these domains at the **application level**.

---

## 4. WAIT State Becomes Power-Aware

WAIT already enforces time.
Now it also enforces **low power**.

Conceptual behavior:

```text
WAIT:
  - shut down radio if possible
  - allow RTOS idle
  - sleep for retry_delay
```

WAIT still does not decide policy.
It only executes it.

---

## 5. Application Context Update

```c
struct app_ctx {
    enum app_state state;
    enum app_state recovery_state;

    int32_t last_temp_mdeg;

    uint32_t retry_count;
    int last_error;
    k_timeout_t retry_delay;

    bool low_power_allowed;

    struct mcp9808_ctx sensor;
    struct net_ctx net;
    struct http_ctx http;
};
```

`low_power_allowed` is a **policy flag**, not a hardware detail.

---

## 6. Deciding When Low Power Is Allowed

Rules:

* Allowed in WAIT
* Allowed in IDLE
* NOT allowed in:

  * SENSOR_INIT
  * NET_INIT
  * SAMPLE
  * SEND

Reason:

* those states expect forward progress

This keeps timing predictable.

---

## 7. WAIT State (Power-Aware Form)

Conceptual implementation:

```c
case APP_STATE_WAIT:
    if (ctx->low_power_allowed) {
        net_suspend(&ctx->net);
    }

    k_sleep(ctx->retry_delay);

    if (ctx->low_power_allowed) {
        net_resume(&ctx->net);
    }

    ctx->state = ctx->recovery_state;
    break;
```

Notes:

* Radio suspend/resume is explicit
* WAIT still owns all sleeping

---

## 8. Why We Do NOT Sleep in Other States

Sleeping in other states causes:

* hidden latency
* missed events
* unpredictable behavior

By centralizing sleep:

* timing is explicit
* debugging is easier
* watchdog integration is simpler

---

## 9. Logging Power Transitions

Power transitions are logged sparingly:

```text
Entering low-power WAIT for 60s
Exiting low-power WAIT
```

This helps:

* battery debugging
* field diagnostics

---

## 10. Interaction with Retry Policy

Backoff + power saving work together:

* short backoff → light sleep
* long backoff → deep sleep

Policy mapping is simple and explainable.

---

## 11. What Step 7 Deliberately Avoids

* No dynamic frequency scaling
* No complex sleep states
* No hardware-specific tuning

Those belong to product-specific optimization later.

---

## 12. Success Criteria for Step 7

Step 7 is complete when:

* Device sleeps during WAIT
* No sleep occurs during active states
* Power behavior matches retry policy
* Logs explain power transitions

---

## 13. Architectural Status After Step 7

After Step 7, the system is:

* architecturally complete
* operationally mature
* energy-aware

This is the baseline for real IoT products.

---

## 14. Next Steps

Optional next steps:

* Step 8: Data buffering / batching
* Step 9: OTA updates
* Step 10: Watchdog integration

None of these change core control flow.

