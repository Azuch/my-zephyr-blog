---

layout: post
title: "From Demo to Production Firmware – Step 3: Sensor Integration Without Breaking Architecture"
date: 2025-12-22
categories: [embedded, zephyr, firmware, architecture]
---

> **Step 3 is the first time real hardware enters the system.**
> This is also the step where many firmware architectures silently collapse.



# Part A – English Version

## Why Step 3 Is Dangerous

Up to Step 2, everything was theoretical:

* states
* control flow
* ownership

Step 3 introduces **reality**:

* I2C can fail
* sensors can NACK
* timing can be wrong
* hardware can disappear

This is where junior firmware often breaks its own architecture by accident.

---

## The Common Mistake: Letting Drivers Think

A very common pattern:

```c
int read_sensor(void)
{
    for (int i = 0; i < 3; i++) {
        if (i2c_read(...) == 0) {
            return 0;
        }
        k_sleep(K_MSEC(10));
    }
    return -EIO;
}
```

This *looks* reasonable.

But it hides decisions:

* retry count
* retry delay
* failure severity

Those decisions now live inside the driver — where they do not belong.

---

## The Rule Introduced in Step 3

> **Drivers do not retry. Drivers report.**

Policy belongs to the application.
Drivers provide information, not decisions.

This single rule prevents massive complexity later.

---

## Choosing the Level of Abstraction

For Step 3, we intentionally use **raw I2C**, not the Zephyr sensor API.

Why?

* it makes ownership explicit
* it exposes failure clearly
* it avoids magic behavior

Abstractions can come later.

---

## MCP9808: What We Actually Need

From the application’s point of view, MCP9808 is simple:

* initialize
* read temperature
* possibly fail

That’s it.

We model it as a **subsystem**, not just a function.

---

## Sensor Context

```c
struct mcp9808_ctx {
    const struct device *i2c;
    uint16_t addr;
};
```

No retries.
No state machine.
Just enough context to talk to hardware.

---

## Minimal Driver API

```c
int mcp9808_init(struct mcp9808_ctx *ctx);
int mcp9808_sample(struct mcp9808_ctx *ctx, int32_t *temp_mdeg);
```

Return values:

* `0` on success
* negative errno on failure

No sleeping. No looping.

---

## Implementation Sketch

```c
int mcp9808_sample(struct mcp9808_ctx *ctx, int32_t *temp_mdeg)
{
    uint8_t reg = 0x05; /* ambient temp register */
    uint8_t buf[2];

    int ret = i2c_write_read(ctx->i2c, ctx->addr,
                             &reg, 1, buf, 2);
    if (ret < 0) {
        return ret;
    }

    int16_t raw = (buf[0] << 8) | buf[1];
    raw &= 0x0FFF;

    *temp_mdeg = (raw * 625) / 10;
    return 0;
}
```

Nothing clever happens here.

---

## Integrating Sensor Init into the State Machine

```c
case APP_STATE_SENSOR_INIT:
    ret = mcp9808_init(&ctx->sensor);
    if (ret < 0) {
        ctx->last_error = ret;
        ctx->state = APP_STATE_ERROR;
        break;
    }
    ctx->state = APP_STATE_NET_INIT;
    break;
```

The driver does not decide what failure means.
The application does.

---

## Integrating Sampling

```c
case APP_STATE_SAMPLE:
    ret = mcp9808_sample(&ctx->sensor, &ctx->last_temp_mdeg);
    if (ret < 0) {
        ctx->last_error = ret;
        ctx->state = APP_STATE_ERROR;
        break;
    }
    ctx->state = APP_STATE_SEND;
    break;
```

Again:

* no retry
* no delay
* no policy

---

## Why This Scales

Later, when we add:

* retries
* backoff
* power management

We will **not touch this driver**.

That is the payoff of discipline.

---

## A Reviewer's Perspective

A reviewer can immediately see:

* where hardware is accessed
* how failure propagates
* who decides recovery

No guessing.

---

## Final Thought (English)

> **Drivers should be boring reporters of reality,
> not decision-makers.**

Step 3 enforces that boundary.

---

# Part B – Phiên bản tiếng Việt

## Vì sao Step 3 nguy hiểm

Step 3 là lúc phần cứng thật xuất hiện:

* I2C có thể lỗi
* sensor có thể không trả lời
* timing có thể sai

Đây là nơi kiến trúc thường bị phá vỡ một cách vô thức.

---

## Lỗi phổ biến: để driver suy nghĩ

Retry trong driver có vẻ tiện.

Nhưng nó che giấu:

* policy
* timing
* mức độ nghiêm trọng

Và làm hệ thống khó kiểm soát.

---

## Quy tắc của Step 3

> **Driver không retry. Driver chỉ báo cáo.**

Quyết định thuộc về application.

---

## Vì sao dùng raw I2C

* rõ ràng
* không magic
* dễ review

Abstraction để sau.

---

## MCP9808 nhìn từ application

Chỉ cần:

* init
* sample
* báo lỗi

Không hơn.

---

## Context sensor

```c
struct mcp9808_ctx {
    const struct device *i2c;
    uint16_t addr;
};
```

---

## API tối thiểu

```c
int mcp9808_init(...);
int mcp9808_sample(...);
```

Không sleep. Không loop.

---

## Tích hợp vào state machine

Application quyết định:

* lỗi có nghiêm trọng không
* đi state nào tiếp theo

---

## Lợi ích lâu dài

Khi thêm retry, backoff, power:

* không đụng driver
* không phá kiến trúc

---

## Góc nhìn reviewer

Reviewer hiểu ngay:

* lỗi đi đâu
* ai chịu trách nhiệm

---

## Lời kết (Tiếng Việt)

> **Driver tốt là driver buồn tẻ.
> Nó chỉ nói sự thật, không đưa ra quyết định.**

Step 3 đặt ranh giới đó.
