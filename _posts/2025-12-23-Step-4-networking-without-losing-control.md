---
layout: post
title: "From Demo to Production Firmware – Step 4: Networking Without Losing Control"
date: 2025-12-23
categories: [embedded, zephyr, firmware, architecture]
---

> **Step 4 introduces the most dangerous subsystem in embedded firmware: networking.**
> Not because it is complex, but because it *pretends* to be helpful.


# Part A – English Version

## Why Step 4 Is Where Architectures Go to Die

Until now:

* sensor failures were local
* timing was predictable
* control flow was obvious

Networking changes everything:

* operations block for long, unknown times
* partial success is common
* libraries hide retries
* callbacks tempt you to put logic in the wrong place

Most firmware architectures quietly collapse at this step.

---

## The Lie Networking APIs Tell You

Most networking APIs are designed for **applications**, not **devices**.

They imply:

* the network is usually available
* failure is exceptional
* retrying is helpful

For an embedded device, all three assumptions are wrong.

---

## Blocking Is Not the Enemy

Junior engineers often believe:

> “Blocking calls are bad. We must be async.”

This is backwards.

Blocking calls are:

* honest
* explicit
* easy to reason about

Async calls hide:

* who owns control flow
* when retries happen
* where errors propagate

So in Step 4, we **intentionally choose blocking networking**.

---

## What We Actually Need from Networking

From the application’s point of view, networking is simple:

* initialize network stack
* send one measurement
* possibly fail

That’s it.

Everything else is policy.

---

## Network as a Subsystem

We treat networking the same way as the sensor:

* minimal API
* no retries
* no sleeping
* no decisions

The application remains in charge.

---

## Network Context

```c
struct net_ctx {
    int sock;
};
```

Nothing fancy.

---

## Minimal Network API

```c
int net_init(struct net_ctx *ctx);
int http_send_temp(struct net_ctx *ctx, int32_t temp_mdeg);
```

Return values:

* `0` on success
* negative errno on failure

No callbacks.

---

## Blocking HTTP Example (Zephyr)

```c
int http_send_temp(struct net_ctx *ctx, int32_t temp_mdeg)
{
    char payload[64];
    snprintk(payload, sizeof(payload),
             "{\"temp\": %d}", temp_mdeg);

    int ret = send(ctx->sock, payload, strlen(payload), 0);
    if (ret < 0) {
        return -errno;
    }

    return 0;
}
```

This function:

* blocks
* either succeeds or fails
* tells the truth

---

## Integrating NET_INIT into the State Machine

```c
case APP_STATE_NET_INIT:
    ret = net_init(&ctx->net);
    if (ret < 0) {
        ctx->last_error = ret;
        ctx->state = APP_STATE_ERROR;
        break;
    }
    ctx->state = APP_STATE_IDLE;
    break;
```

The network does not retry itself.

---

## Integrating SEND

```c
case APP_STATE_SEND:
    ret = http_send_temp(&ctx->net, ctx->last_temp_mdeg);
    if (ret < 0) {
        ctx->last_error = ret;
        ctx->recovery_state = APP_STATE_NET_INIT;
        ctx->state = APP_STATE_ERROR;
        break;
    }
    ctx->state = APP_STATE_WAIT;
    break;
```

Notice something important:

* SEND does not retry
* SEND does not sleep
* SEND does not reconnect

It reports. The application decides.

---

## The Introduction of recovery_state

This is a critical addition:

```c
enum app_state recovery_state;
```

Why?

Because after failure, we need to answer:

> “Where do we resume once the failure is handled?”

This is not the same as:

> “What failed?”

Separating these concepts keeps logic clean.

---

## Why We Do NOT Jump Directly to NET_INIT

Tempting but wrong:

```c
ctx->state = APP_STATE_NET_INIT;
```

This bypasses:

* centralized error handling
* retry policy
* backoff
* logging

With `recovery_state`, **all failures go through ERROR and WAIT**.

---

## What Step 4 Deliberately Avoids

Step 4 does NOT:

* use callbacks
* retry inside networking
* manage timeouts dynamically
* optimize throughput

Those come later, if needed.

---

## A Reviewer's Perspective

A reviewer can now clearly see:

* where network failures occur
* how they propagate
* who decides recovery

No hidden magic.

---

## Final Thought (English)

> **Networking should tell you the truth,
> even when that truth is uncomfortable.**

Blocking APIs help you keep control.

---

# Part B – Phiên bản tiếng Việt

## Vì sao Step 4 là nơi kiến trúc thường sụp đổ

Network làm mọi thứ phức tạp hơn:

* block lâu
* lỗi không rõ ràng
* thư viện “giúp đỡ” quá nhiều

Nhiều firmware chết ở đây.

---

## Lời nói dối của API mạng

API mạng thường giả định:

* mạng ổn định
* lỗi hiếm

Thiết bị thì ngược lại.

---

## Blocking không xấu

Blocking:

* rõ ràng
* dễ suy luận

Async thường:

* giấu logic
* giấu ownership

Vì vậy Step 4 **chọn blocking trước**.

---

## Network như một subsystem

* API tối thiểu
* không retry
* không sleep

Application quyết định tất cả.

---

## Tích hợp vào state machine

SEND chỉ:

* gọi
* nhận kết quả
* báo lỗi

---

## recovery_state là chìa khóa

Nó trả lời:

> “Sau khi xử lý lỗi, quay lại đâu?”

Không phải:

> “Lỗi gì xảy ra?”

---

## Vì sao không nhảy thẳng NET_INIT

Nhảy thẳng sẽ phá:

* retry policy
* backoff
* logging tập trung

ERROR + WAIT là bắt buộc.

---

## Step 4 KHÔNG làm gì

* không callback
* không tối ưu
* không async

Cố ý giữ đơn giản.

---

## Lời kết (Tiếng Việt)

> **Network tốt là network nói thật,
> dù sự thật đó khó chịu.**

Blocking giúp bạn giữ quyền kiểm soát.
