---
layout: post
title: "From Demo to Production Firmware – Step 5: Logging That Explains Behavior"
date: 2025-12-24
categories: [embedded, zephyr, firmware, architecture]
---

> **Step 5 does not change behavior.**
> It changes whether humans can understand that behavior.


# Part A – English Version

## Why Step 5 Exists

Up to Step 4, our firmware:

* has explicit control flow
* integrates real hardware
* talks to the network
* survives failure structurally

But there is still a critical gap:

> **If this device fails in the field, can we explain what happened?**

If the answer is “we can attach a debugger” or “we can reproduce it locally”, the firmware is not production-ready.

Production firmware must explain itself **after the fact**.

---

## Logging Is Not Debugging

Many junior engineers treat logging as:

* `printf` spam
* something enabled only during bring-up
* a substitute for a debugger

That mindset leads to:

* noisy logs
* missing critical information
* performance problems
* unreadable output

In production firmware:

> **Logging is a communication channel to future humans.**

---

## The Core Rule of Step 5

> **Logs must describe system behavior, not implementation details.**

Bad logs:

```
read sensor
sending http
sending http
error
error
```

Good logs:

```
State: IDLE -> SAMPLE
State: SAMPLE -> SEND
Error -110, retry 2, recover to NET_INIT
```

One tells a story. The other does not.

---

## Log State Transitions, Not Actions

The most important thing to log is **state transitions**.

Why?

* states represent intent
* transitions represent decisions
* everything else is noise

We add a helper:

```c
static const char *state_str(enum app_state s)
{
    switch (s) {
    case APP_STATE_BOOT: return "BOOT";
    case APP_STATE_SENSOR_INIT: return "SENSOR_INIT";
    case APP_STATE_NET_INIT: return "NET_INIT";
    case APP_STATE_IDLE: return "IDLE";
    case APP_STATE_SAMPLE: return "SAMPLE";
    case APP_STATE_SEND: return "SEND";
    case APP_STATE_WAIT: return "WAIT";
    case APP_STATE_ERROR: return "ERROR";
    default: return "UNKNOWN";
    }
}
```

And log only on change:

```c
if (prev_state != ctx->state) {
    LOG_INF("State: %s -> %s",
            state_str(prev_state),
            state_str(ctx->state));
}
```

This alone makes the system readable.

---

## Centralized Error Logging

Another critical rule:

> **An error should be logged exactly once.**

If errors are logged:

* in drivers
* in callbacks
* in SEND
* in WAIT

You lose signal in noise.

We log errors **only when entering ERROR**:

```c
case APP_STATE_ERROR:
    LOG_ERR("Error %d, retry %u, recover to %s",
            ctx->last_error,
            ctx->retry_count,
            state_str(ctx->recovery_state));
    ctx->retry_count++;
    ctx->state = APP_STATE_WAIT;
    break;
```

Everything important is in one line.

---

## Why ERROR Is the Perfect Logging Point

At ERROR:

* the failure has already happened
* the system is about to decide policy
* the context is complete

Logging earlier:

* lacks context
* leads to duplication

Logging later:

* loses cause

ERROR is the sweet spot.

---

## Logging Success (Sparingly)

Success logs matter — but only when they confirm progress.

Example:

```c
LOG_INF("Temperature %d mdeg sent successfully", ctx->last_temp_mdeg);
```

This tells us:

* the device is alive
* the data path works

Do not log every success.

---

## Log Levels Discipline

A simple policy works well:

* `LOG_ERR` – state-changing failures
* `LOG_INF` – state transitions, milestones
* `LOG_DBG` – bring-up only

Production systems should run mostly at `INF + ERR`.

---

## What Step 5 Deliberately Avoids

Step 5 does not introduce:

* metrics
* persistent logs
* tracing buffers
* cloud logging

Those are product decisions later.

Here we focus on **clarity**, not analytics.

---

## A Reviewer's Perspective

A reviewer reading your logs should be able to:

* reconstruct execution flow
* identify failure causes
* understand recovery behavior

Without reading the code.

That is the bar.

---

## Final Thought (English)

> **If your logs cannot explain your firmware,
> your firmware is not finished.**

Step 5 makes behavior observable.

---

# Part B – Phiên bản tiếng Việt

## Vì sao cần Step 5

Sau Step 4, firmware đã:

* có kiến trúc rõ ràng
* xử lý lỗi
* giao tiếp network

Nhưng câu hỏi quan trọng vẫn là:

> **Khi thiết bị lỗi ngoài thực tế, ta có giải thích được không?**

Nếu cần debugger hoặc phải đoán, firmware chưa sẵn sàng cho production.

---

## Logging không phải debug

Logging không phải là `printf` cho vui.

Nó là:

> **Kênh giao tiếp với con người trong tương lai.**

---

## Quy tắc cốt lõi

> **Log hành vi hệ thống, không log chi tiết triển khai.**

Log tốt kể câu chuyện.

---

## Log state transition

State nói lên ý định.

Transition nói lên quyết định.

Đó là thứ cần log.

---

## Log lỗi tập trung

Lỗi chỉ nên log **một lần**.

Tại state ERROR.

---

## Vì sao ERROR là nơi log lý tưởng

Ở ERROR:

* biết lỗi gì
* biết retry bao nhiêu
* biết sẽ phục hồi ra sao

Context đầy đủ.

---

## Log thành công có chọn lọc

Chỉ log khi:

* xác nhận hệ thống đang tiến triển

Không spam.

---

## Kỷ luật log level

* ERR: lỗi thay đổi state
* INF: transition, mốc quan trọng
* DBG: bring-up

---

## Step 5 KHÔNG làm gì

* không metrics
* không cloud log
* không tracing

Chỉ tập trung vào **dễ hiểu**.

---

## Góc nhìn reviewer

Reviewer chỉ cần log là hiểu được:

* hệ thống chạy thế nào
* lỗi gì xảy ra
* phục hồi ra sao

---

## Lời kết (Tiếng Việt)

> **Nếu log không giải thích được firmware,
> firmware chưa hoàn thành.**

Step 5 làm hệ thống trở nên quan sát được.
