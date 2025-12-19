---
layout: post
title: "From Demo to Production Firmware – Step 6: Retry and Backoff as Policy"
date: 2025-12-25
categories: [embedded, zephyr, firmware, architecture]
---

> **Step 6 answers a deceptively simple question:**
> *When things fail repeatedly, how should a device behave over time?*

# Part A – English Version

## Why Step 6 Exists

After Step 5, our firmware:

* can fail safely
* can explain failures
* can recover structurally

But it still behaves like an impatient human.

Without an explicit retry policy, systems tend to:

* retry immediately
* retry too often
* retry forever

This hurts:

* power consumption
* network infrastructure
* backend services
* device credibility

Step 6 turns retries into **policy**, not accidents.

---

## The Core Insight

> **Retries are not a technical detail. They are a behavioral decision.**

How often you retry says something about:

* how valuable the data is
* how patient the device is
* how respectful it is to the network

This must be explicit.

---

## The Anti-Pattern: Inline Retries

A common mistake:

```c
for (int i = 0; i < 5; i++) {
    if (send() == 0) {
        return 0;
    }
    k_sleep(K_SECONDS(1));
}
```

Problems:

* retry count is hidden
* delay is arbitrary
* power impact is unclear
* logs are noisy

Most importantly: **policy is buried in code**.

---

## Centralizing Retry Policy

Retries belong to the application, not subsystems.

We extend the context:

```c
struct app_ctx {
    enum app_state state;
    enum app_state recovery_state;

    uint32_t retry_count;
    int last_error;

    k_timeout_t retry_delay;
};
```

Retry behavior is now visible state.

---

## Designing a Backoff Strategy

We start simple and deterministic:

```c
static k_timeout_t calc_backoff(uint32_t retry)
{
    if (retry < 3) {
        return K_SECONDS(10);
    } else if (retry < 6) {
        return K_MINUTES(1);
    } else {
        return K_MINUTES(5);
    }
}
```

Why this works:

* predictable
* debuggable
* power-friendly

Randomization can be added later.

---

## Applying Backoff in ERROR

```c
case APP_STATE_ERROR:
    ctx->retry_delay = calc_backoff(ctx->retry_count);

    LOG_ERR("Error %d, retry %u, wait %lld ms, recover to %s",
            ctx->last_error,
            ctx->retry_count,
            ctx->retry_delay.ticks,
            state_str(ctx->recovery_state));

    ctx->retry_count++;
    ctx->state = APP_STATE_WAIT;
    break;
```

The ERROR state now:

* decides delay
* logs policy
* remains centralized

---

## WAIT Becomes Meaningful

WAIT is no longer a dumb sleep.

```c
case APP_STATE_WAIT:
    k_sleep(ctx->retry_delay);
    ctx->state = ctx->recovery_state;
    break;
```

All waiting behavior flows through one state.

---

## Why This Is Predictable

Given logs like:

```
Error -110, retry 4, wait 60000 ms, recover to NET_INIT
```

An engineer can immediately answer:

* how long the device will be quiet
* what it will try next
* why it behaved that way

This is professionalism.

---

## Avoiding Retry Storms

Without backoff:

* thousands of devices retry together
* servers get overloaded
* outages cascade

With explicit policy:

* retries spread over time
* systems degrade gracefully

This matters at scale.

---

## What Step 6 Does NOT Do

Step 6 does not:

* add randomness
* distinguish error types
* persist retry state

Those are product-level decisions.

We start with clarity.

---

## A Reviewer's Perspective

A reviewer can now see:

* retry behavior clearly
* delays explicitly
* no hidden loops

And can reason about fleet behavior.

---

## Final Thought (English)

> **A polite device retries thoughtfully.
> An impatient one becomes noise.**

Step 6 teaches patience.

---

# Part B – Phiên bản tiếng Việt

## Vì sao cần Step 6

Sau Step 5, hệ thống đã:

* xử lý lỗi
* log rõ ràng

Nhưng nếu không có retry policy:

* hệ thống trở nên nóng vội
* tốn pin
* phá backend

Step 6 biến retry thành **hành vi có chủ ý**.

---

## Insight cốt lõi

> **Retry là quyết định hành vi, không phải chi tiết kỹ thuật.**

---

## Anti-pattern: retry inline

Retry trong code con:

* khó thấy
* khó review
* khó thay đổi

---

## Tập trung retry policy

Retry thuộc về application.

Context mở rộng rõ ràng.

---

## Backoff đơn giản nhưng hiệu quả

Deterministic backoff:

* dễ debug
* dễ giải thích

---

## ERROR quyết định policy

ERROR:

* tính delay
* log
* chuyển WAIT

Không nơi nào khác làm việc này.

---

## WAIT có ý nghĩa

WAIT là nơi duy nhất được ngủ.

---

## Tránh retry storm

Retry có kiểm soát giúp:

* hệ thống ổn định
* backend sống sót

---

## Step 6 KHÔNG làm gì

* không random
* không phân loại lỗi

Giữ rõ ràng trước.

---

## Lời kết (Tiếng Việt)

> **Thiết bị lịch sự retry có suy nghĩ.
> Thiết bị nóng vội trở thành gánh nặng.**

Step 6 dạy thiết bị kiên nhẫn.
