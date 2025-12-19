---
layout: post 
title: "From Demo to Production Firmware – Step 1: Thinking Like a Device, Not a Script" 
date: 2025-01-02 
categories: [embedded, zephyr, firmware, architecture]
---
> **Step 1 is about mindset, not code.**
> Before architecture, before APIs, before Zephyr — we must change how we think about software that runs on a device.


# Part A – English Version

## Why Step 1 Exists

Most junior embedded engineers do not fail because they cannot write C. They fail because they unconsciously write **scripts**, not **devices**.

A script:

* starts
* does something
* exits or loops blindly

A device:

* boots
* runs forever
* experiences partial failure
* must recover without human help

Step 1 exists to force a mental shift *before* any technical decisions are made.

---

## The Script Mindset (Very Common)

A script mindset usually looks like this:

```c
while (1) {
    read_sensor();
    send_to_server();
    k_sleep(K_SECONDS(5));
}
```

This code is not "wrong". It often works in demos.

But ask the uncomfortable questions:

* What if `read_sensor()` fails once?
* What if Wi-Fi drops for 10 minutes?
* What if DNS works but HTTP hangs?
* What if this runs for 6 months?

The script mindset has no answers.

---

## Why Scripts Break in the Real World

Scripts assume:

* linear success
* short lifetimes
* human supervision

Devices assume:

* failures are normal
* time matters
* recovery is mandatory

When script-style code is forced into a device:

* retries appear randomly
* callbacks grow logic
* globals spread
* debugging becomes archaeology

Most production bugs are *architectural*, not algorithmic.

---

## The Device Mindset

Thinking like a device means accepting these truths:

1. **The device never finishes**
2. **Failure is a normal state**
3. **Time is a resource**
4. **Power is a resource**
5. **Recovery must be intentional**

Once you accept these, your design naturally changes.

---

## Devices Are Stateful by Nature

A script pretends to be stateless.

A device is always in *some state*:

* booting
* initializing hardware
* idle
* working
* waiting
* recovering

Ignoring this does not remove state. It only makes it implicit and unmanageable.

Step 1 is about deciding:

> **We will make state explicit.**

---

## Why This Is Independent of Zephyr

Everything in Step 1 applies equally to:

* bare metal
* Zephyr
* FreeRTOS
* Linux user space

Zephyr does not save you from poor thinking. It only gives you tools.

Good architecture is **RTOS-agnostic**.

---

## A Simple Mental Test

Before writing code, ask:

> "If this function fails 1 out of 1000 times, what happens next?"

If the honest answer is:

> "I don't know" or "It probably retries"

Then you are still in script mode.

---

## Ownership: The Hidden Problem

Script-style code has unclear ownership:

* callbacks decide logic
* drivers retry silently
* globals hold state

Device-style code has explicit ownership:

* one place decides control flow
* one place decides retry policy
* one place decides sleep

Step 1 plants the seed for this discipline.

---

## What Step 1 Does NOT Produce

At the end of Step 1, we do NOT yet have:

* a state machine
* modules
* drivers
* Zephyr APIs

That is intentional.

Step 1 only produces **clarity of intent**.

---

## The Commitment You Are Making

By finishing Step 1, you are committing to:

* designing for failure first
* making time and power visible
* keeping logic centralized
* writing boring code on purpose

This commitment will guide every later step.

---

## Final Thought (English)

> **Devices are not programs that run.**
> **They are systems that survive.**

Once you internalize this, architecture stops feeling complicated — it starts feeling necessary.

---

# Part B – Phiên bản tiếng Việt

## Vì sao cần Step 1

Rất nhiều kỹ sư embedded mới không thất bại vì họ kém C.

Họ thất bại vì họ đang viết **script**, không phải **thiết bị**.

Script:

* chạy từ đầu đến cuối
* giả định thành công
* ít quan tâm đến thời gian dài

Thiết bị:

* khởi động
* chạy vô hạn
* lỗi là chuyện bình thường
* phải tự phục hồi

Step 1 tồn tại để thay đổi cách suy nghĩ đó.

---

## Tư duy script (rất phổ biến)

```c
while (1) {
    read_sensor();
    send_to_server();
    k_sleep(K_SECONDS(5));
}
```

Code này thường chạy được. Nhưng câu hỏi là:

* Nếu sensor lỗi thì sao?
* Nếu mạng mất 10 phút thì sao?
* Nếu HTTP treo thì sao?
* Nếu chạy 6 tháng thì sao?

Script không trả lời được.

---

## Vì sao script sụp đổ ngoài thực tế

Script giả định:

* mọi thứ thành công
* vòng đời ngắn
* có người giám sát

Thiết bị giả định:

* lỗi là bình thường
* thời gian là tài nguyên
* phải tự cứu mình

Khi script bị ép làm thiết bị:

* retry mọc lung tung
* callback chứa logic
* global lan tràn
* debug trở thành ác mộng

---

## Tư duy thiết bị

Tư duy như một thiết bị nghĩa là chấp nhận:

1. **Thiết bị không bao giờ kết thúc**
2. **Lỗi là trạng thái bình thường**
3. **Thời gian là tài nguyên**
4. **Năng lượng là tài nguyên**
5. **Phục hồi phải có chủ ý**

Một khi chấp nhận, kiến trúc sẽ tự thay đổi.

---

## Thiết bị luôn có trạng thái

Script giả vờ không có state.

Thiết bị thì luôn ở một trạng thái nào đó:

* boot
* init
* idle
* làm việc
* chờ
* phục hồi

Không nói ra không làm state biến mất. Nó chỉ làm mọi thứ rối hơn.

Step 1 là quyết định:

> **Chúng ta sẽ làm state rõ ràng.**

---

## Điều này không phụ thuộc Zephyr

Tư duy này áp dụng cho:

* bare metal
* Zephyr
* FreeRTOS
* Linux

RTOS không cứu bạn khỏi tư duy kém.

---

## Một bài test đơn giản

Trước khi viết code, hãy hỏi:

> "Nếu hàm này lỗi 1/1000 lần thì chuyện gì xảy ra?"

Nếu câu trả lời là:

> "Không biết" hoặc "Chắc nó retry"

Bạn vẫn đang tư duy script.

---

## Ownership – vấn đề ẩn

Script thường có ownership mơ hồ:

* callback quyết định logic
* driver retry ngầm
* global giữ state

Thiết bị tốt có ownership rõ:

* một nơi quyết định luồng
* một nơi quyết định retry
* một nơi quyết định sleep

Step 1 chỉ gieo hạt giống cho kỷ luật này.

---

## Step 1 KHÔNG tạo ra

Sau Step 1, chúng ta chưa có:

* state machine
* module
* driver
* API Zephyr

Và điều đó là cố ý.

Step 1 chỉ tạo ra **ý định rõ ràng**.

---

## Cam kết bạn đang đưa ra

Hoàn thành Step 1 nghĩa là bạn chấp nhận:

* thiết kế cho lỗi
* thời gian và năng lượng phải thấy được
* logic tập trung
* code chán nhưng an toàn

Cam kết này dẫn dắt mọi bước sau.

---

## Lời kết (Tiếng Việt)

> **Thiết bị không phải chương trình chạy xong.**
> **Nó là hệ thống phải sống sót.**

Khi hiểu điều này, kiến trúc không còn phức tạp — nó trở nên tất yếu.
