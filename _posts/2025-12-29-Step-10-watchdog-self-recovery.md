---
layout: post
title: "From Demo to Production Firmware – Step 10: Watchdog and Last-Resort Recovery"
date: 2025-12-29
categories: [embedded, firmware, architecture, watchdog]
---

> **Step 10 is about humility.**  
> Assume everything above can fail.

After Step 9, the system is:

* reliable
* observable
* power-aware
* outage-tolerant
* updatable

Step 10 adds the **final safety net**:

> **Automatic recovery when the software itself gets stuck.**

This step assumes: *even good code can fail in bad ways*.

---

## 1. Purpose of Step 10

The watchdog answers one brutal question:

> **"What if the system stops making progress?"**

Examples:

* deadlock
* infinite loop
* stuck driver
* memory corruption
* unforeseen corner case

No amount of retry logic can fix these.

---

## 2. Watchdog Philosophy

Senior rule:

> **"The watchdog is not a feature. It is a judge."**

Key principles:

* Watchdog does not understand logic
* Watchdog only observes *liveness*
* Reset is not failure; it is recovery

---

## 3. What the Watchdog Should Mean

A watchdog reset means:

> "The application failed to uphold its contract of progress."

It does NOT mean:

* the device is broken
* the design is wrong

It means the system protected itself.

---

## 4. Where to Feed the Watchdog (Critical)

**Never feed the watchdog in callbacks or drivers.**

Correct rule:

> **Feed the watchdog only when the system makes forward progress.**

In this architecture, that means:

* after successful state transitions
* after SEND success
* after successful WAIT completion

---

## 5. Application Context Update

```c
struct app_ctx {
    enum app_state state;
    enum app_state recovery_state;

    uint32_t retry_count;
    k_timeout_t retry_delay;
    int last_error;

    struct sample_buffer samples;

    bool progress_made;

    struct mcp9808_ctx sensor;
    struct net_ctx net;
    struct http_ctx http;
};
```

`progress_made` is a **logical signal**, not hardware-specific.

---

## 6. Defining "Progress"

Progress means:

* State changes
* Successful SEND
* Completion of WAIT
* Successful SENSOR_INIT
* Successful NET_INIT

Progress does NOT mean:

* looping
* retrying without delay
* logging

---

## 7. Feeding the Watchdog (Conceptual)

```c
if (ctx->progress_made) {
    watchdog_feed();
    ctx->progress_made = false;
}
```

This check happens once per main loop iteration.

---

## 8. Timeout Selection

Watchdog timeout must be:

* Longer than the longest valid blocking operation
* Shorter than "forever"

Example:

* HTTP timeout: 30s
* DNS timeout: 10s
* Watchdog: 60–90s

This ensures:

* real hangs trigger reset
* slow networks do not

---

## 9. Interaction with WAIT and Power Management

Important rule:

> **Do not feed the watchdog before long sleeps.**

Correct behavior:

* Feed watchdog
* Enter WAIT sleep
* Wake up
* Feed again after progress

This ensures sleep itself is not mistaken for a hang.

---

## 10. Handling Reset After Watchdog

On boot:

* MCUboot runs first
* Application starts fresh
* State = BOOT

Optional:

* Persist reset reason
* Log "watchdog reset"

The system treats reset as recovery, not error.

---

## 11. Logging Watchdog Events

Log sparingly:

* Watchdog started
* Watchdog fed (DEBUG only)
* Reset detected

Example:

```text
Reset reason: watchdog
```

---

## 12. What Step 10 Deliberately Avoids

* Feeding watchdog everywhere
* Multiple watchdogs
* Complex health metrics

The watchdog must remain simple and strict.

---

## 13. Success Criteria for Step 10

Step 10 is complete when:

* System resets on true hangs
* Normal slow operations do not trigger resets
* Watchdog logic is explainable
* Reset is safe and recoverable

---

## 14. Architectural Status After Step 10

After Step 10, the device has:

* graceful failure handling
* controlled retries
* safe self-reset

This is **production-hardened firmware**.

---

## 15. Final Note

The watchdog is the last safety net.

If it ever triggers:

* the system did its job
* not everything went wrong

That mindset is critical in production systems.
