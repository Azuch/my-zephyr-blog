---
layout: post
title: "From Demo to Production Firmware – Step 2: The State Machine Skeleton"
date: 2025-12-21
categories: [embedded, zephyr, firmware, architecture]
---

> **Step 2 is where architecture becomes visible.**
> We introduce the simplest possible structure that can survive failure: a single, explicit state machine owned by the application.


# Part A – English Version

## Why Step 2 Exists

After Step 1, we accepted a critical truth:

> *A device is not a script. It is a system that runs forever and must survive failure.*

Step 2 answers the next obvious question:

> **Where does the system decide what to do next?**

If you do not answer this explicitly, the answer becomes:

* callbacks
* timers
* globals
* side effects

That is how firmware becomes unreviewable.

---

## The Core Rule of Step 2

> **There must be exactly one place that decides application control flow.**

Not:

* drivers
* callbacks
* interrupts
* background threads

One place.

We will call it the **application state machine**.

---

## Why a State Machine (and Not "Just a Loop")

Many engineers resist state machines because they imagine:

* complex diagrams
* switch-case monsters
* unreadable code

But the alternative is worse.

Without an explicit state machine:

* state still exists
* transitions still happen
* but nobody owns them

A state machine does not add complexity.
It **reveals** complexity that already exists.

---

## Defining Explicit States

We start with states that represent **lifecycle**, not features:

```c
enum app_state {
    APP_STATE_BOOT,
    APP_STATE_SENSOR_INIT,
    APP_STATE_NET_INIT,
    APP_STATE_IDLE,
    APP_STATE_SAMPLE,
    APP_STATE_SEND,
    APP_STATE_WAIT,
    APP_STATE_ERROR,
};
```

Notice what is missing:

* no "RETRY" state
* no "WIFI_FAIL" state
* no "HTTP_TIMEOUT" state

States represent *what the system is doing*, not *why it failed*.

---

## The Application Context (Ownership Made Concrete)

All mutable application state lives in **one struct**:

```c
struct app_ctx {
    enum app_state state;

    /* bookkeeping */
    uint32_t retry_count;
    int last_error;

    /* placeholders for future subsystems */
    void *sensor;
    void *net;
};
```

This struct is:

* owned by the application
* passed explicitly
* never global by accident

If a variable does not fit here, it probably does not belong.

---

## The Coordinator Function

This is the heart of the system:

```c
static void app_run(struct app_ctx *ctx)
{
    switch (ctx->state) {
    case APP_STATE_BOOT:
        ctx->retry_count = 0;
        ctx->state = APP_STATE_SENSOR_INIT;
        break;

    case APP_STATE_SENSOR_INIT:
        ctx->state = APP_STATE_NET_INIT;
        break;

    case APP_STATE_NET_INIT:
        ctx->state = APP_STATE_IDLE;
        break;

    case APP_STATE_IDLE:
        ctx->state = APP_STATE_SAMPLE;
        break;

    case APP_STATE_SAMPLE:
        ctx->state = APP_STATE_SEND;
        break;

    case APP_STATE_SEND:
        ctx->state = APP_STATE_WAIT;
        break;

    case APP_STATE_WAIT:
        ctx->state = APP_STATE_IDLE;
        break;

    case APP_STATE_ERROR:
        ctx->retry_count++;
        ctx->state = APP_STATE_WAIT;
        break;
    }
}
```

Nothing interesting happens yet — and that is perfect.

---

## Why This Function Is Intentionally Boring

This function:

* does not block
* does not sleep
* does not retry
* does not log

It only:

* looks at state
* decides the next state

This boredom is discipline.

---

## The Main Loop (Yes, Still while(1))

The main loop remains simple:

```c
void main(void)
{
    static struct app_ctx app = {
        .state = APP_STATE_BOOT,
    };

    while (1) {
        app_run(&app);
        k_sleep(K_MSEC(50));
    }
}
```

Key observation:

> `while(1)` is not the problem.
> *What you put inside it is.*

---

## Why We Avoid Multiple Threads

At this stage, **one thread is a feature**:

* no races
* no locks
* no priority bugs
* easier debugging

Concurrency is added later *only if required*.

---

## ERROR Is a State, Not a Side Effect

Notice:

```c
case APP_STATE_ERROR:
    ctx->retry_count++;
    ctx->state = APP_STATE_WAIT;
    break;
```

ERROR:

* is explicit
* is visible
* is unavoidable

We do not "handle" errors inline.
We **transition** to them.

---

## What Step 2 Deliberately Does NOT Do

Step 2 does not yet:

* talk to hardware
* talk to network
* retry intelligently
* sleep intelligently

This is intentional.

We are building a skeleton that will not change later.

---

## A Reviewer's Perspective

A senior reviewer looking at this code can immediately answer:

* Where is control flow decided? ✔
* Where is state stored? ✔
* Where do retries live? ✔ (not yet, but planned)
* Can this grow without chaos? ✔

That is the real goal.

---

## Final Thought (English)

> **If you cannot point to the one place that decides behavior,
> you do not control your firmware.**

Step 2 gives you that control.

---

# Part B – Phiên bản tiếng Việt

## Vì sao cần Step 2

Sau Step 1, chúng ta đã chấp nhận:

> *Thiết bị không phải script, mà là hệ thống phải sống sót.*

Câu hỏi tiếp theo là:

> **Ai quyết định hệ thống sẽ làm gì tiếp theo?**

Nếu bạn không chỉ ra rõ ràng, câu trả lời sẽ là:

* callback
* timer
* biến global
* side effect

Và đó là con đường dẫn đến firmware rối.

---

## Quy tắc cốt lõi của Step 2

> **Chỉ có một nơi duy nhất quyết định luồng điều khiển của ứng dụng.**

Không phải driver.
Không phải callback.

Một nơi duy nhất.

---

## Vì sao dùng state machine

Không có state machine không có nghĩa là không có state.

Nó chỉ có nghĩa là:

* state bị ẩn
* transition không ai chịu trách nhiệm

State machine không làm code phức tạp hơn.
Nó làm mọi thứ *rõ ràng hơn*.

---

## Định nghĩa state rõ ràng

```c
enum app_state {
    APP_STATE_BOOT,
    APP_STATE_SENSOR_INIT,
    APP_STATE_NET_INIT,
    APP_STATE_IDLE,
    APP_STATE_SAMPLE,
    APP_STATE_SEND,
    APP_STATE_WAIT,
    APP_STATE_ERROR,
};
```

State mô tả:

* hệ thống đang làm gì

Không mô tả:

* lỗi gì xảy ra

---

## app_ctx – ownership cụ thể hóa

```c
struct app_ctx {
    enum app_state state;
    uint32_t retry_count;
    int last_error;
    void *sensor;
    void *net;
};
```

Mọi state mutable đều nằm ở đây.

Không có magic.

---

## Hàm điều phối trung tâm

```c
static void app_run(struct app_ctx *ctx)
{
    switch (ctx->state) {
    case APP_STATE_BOOT:
        ctx->retry_count = 0;
        ctx->state = APP_STATE_SENSOR_INIT;
        break;
    /* ... */
    }
}
```

Hàm này:

* không sleep
* không block
* không retry

Chỉ quyết định state tiếp theo.

---

## Vòng lặp main

```c
while (1) {
    app_run(&app);
    k_sleep(K_MSEC(50));
}
```

`while(1)` không xấu.

Logic bên trong mới là vấn đề.

---

## ERROR là state, không phải hack

ERROR là một phần của thiết kế.

Không phải trường hợp đặc biệt.

---

## Step 2 CHƯA làm gì

Chưa:

* giao tiếp hardware
* retry
* quản lý năng lượng

Và đó là chủ ý.

---

## Góc nhìn reviewer

Reviewer có thể đọc và hiểu ngay:

* luồng điều khiển ở đâu
* state nằm ở đâu
* hệ thống có mở rộng được không

---

## Lời kết (Tiếng Việt)

> **Nếu bạn không chỉ ra được nơi quyết định hành vi,
> firmware không còn nằm trong tay bạn.**

Step 2 trao lại quyền kiểm soát đó.
