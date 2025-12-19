---
layout: post
title: "From Demo to Production Firmware – Step 0: Why This Series Exists"
date: 2025-01-01
categories: [embedded, zephyr, firmware, architecture]
---

> **This is Step 0 of a long-form series about building production-grade firmware with Zephyr.**
> Step 0 exists to align expectations, mindset, and goals *before* we write a single line of code.

---

# Part A – English Version

## Why This Series Exists

If you search for Zephyr tutorials today, you will find:

* official samples that “work”
* vendor demos that look clean at first glance
* blog posts that focus on APIs and drivers

And yet, when you look at **real production firmware**, it often looks *nothing like those samples*.

This creates a confusing gap for junior engineers:

> “If Zephyr is production-grade, why does production firmware look so different from Zephyr samples?”

This series exists to close that gap.

---

## Demo Code vs Device Firmware

Most tutorials teach you how to write **demo code**.

Demo code:

* runs once or for a short time
* assumes success
* prioritizes simplicity
* is written to *show features*

A **device**, however:

* runs forever
* fails constantly
* must recover automatically
* must be explainable months later

The difference is not skill.
The difference is **intent**.

---

## Why Zephyr Samples Look “Bad” (and Why That’s OK)

Zephyr samples are not bad code.
They are **context-free code**.

Their goals are:

* demonstrate an API
* compile everywhere
* be minimal

They are **not** designed to:

* survive long outages
* handle partial failures
* be maintained by a team

If you copy them directly into a product, you will eventually feel embarrassed when someone reviews your code.

This series teaches you **how to go from sample-style thinking to system-style thinking**.

---

## The Real Question This Series Answers

This is not a series about:

* MCP9808
* HTTP clients
* Wi-Fi drivers

Those are interchangeable.

The real question is:

> **How do professionals structure firmware so it is reliable, predictable, maintainable, and explainable?**

Once you understand that, APIs become trivial.

---

## Target Audience

This series is written for:

* junior embedded engineers who feel “something is missing”
* self-taught developers using ESP32 + Zephyr
* freelancers who want code they are not embarrassed to ship
* engineers transitioning from Arduino-style code to real firmware

It assumes:

* basic C knowledge
* basic RTOS concepts
* willingness to think before coding

---

## What This Series Is NOT

This series intentionally does **not**:

* chase performance
* show clever tricks
* optimize prematurely
* use advanced abstractions early

Everything here is:

* explicit
* boring
* deliberate

That is a feature, not a flaw.

---

## The Running Example

Throughout the series, we will use one concrete project:

> **Read temperature from an MCP9808 sensor and send it to a server**

This looks simple.

That is exactly why it is a good teaching vehicle.

Most real bugs happen in “simple” systems that were never designed to fail.

---

## The Core Principles You Will See Repeated

Over and over, the same ideas will appear:

* devices are not scripts
* failure is normal
* ownership must be explicit
* callbacks are not brains
* retries are policy, not accidents
* time and power must be visible

If these ideas feel repetitive, that is intentional.

---

## Structure of the Series

Each step is one post:

* Step 1: Thinking like a device, not a script
* Step 2: The state machine skeleton
* Step 3: Sensor integration without architectural damage
* Step 4: Networking without losing control
* Step 5: Logging for explainability
* Step 6: Retry and backoff policy
* Step 7: Power management
* Step 8: Buffering and batching
* Step 9: OTA with MCUboot
* Step 10: Watchdog and last-resort recovery

Each post:

* is standalone
* contains full code for that step
* builds on previous thinking without rewriting it

---

## Final Thought (English)

> **Good firmware is not impressive.
> It is calm under failure and boring to maintain.**

If you want to learn *that*, you are in the right place.

---

# Part B – Phiên bản tiếng Việt

## Vì sao loạt bài này tồn tại

Nếu bạn học Zephyr ngày hôm nay, bạn sẽ thấy:

* sample chính thức chạy được
* demo từ vendor trông có vẻ “đúng”
* tutorial tập trung vào API

Nhưng khi nhìn vào **firmware thực tế trong sản phẩm**, bạn sẽ thấy nó **rất khác**.

Điều này tạo ra một khoảng trống khó chịu cho người mới:

> “Nếu Zephyr là production-grade, tại sao code sản phẩm lại không giống sample?”

Loạt bài này tồn tại để trả lời câu hỏi đó.

---

## Demo code khác firmware thiết bị như thế nào

Demo code:

* chạy ngắn hạn
* giả định mọi thứ đều thành công
* viết để minh họa

Firmware thiết bị:

* chạy 24/7
* lỗi là chuyện bình thường
* phải tự phục hồi
* phải giải thích được sau nhiều tháng

Khác biệt không nằm ở kỹ năng, mà nằm ở **cách suy nghĩ**.

---

## Vì sao Zephyr sample trông “xấu” (và điều đó không sai)

Zephyr sample không phải code kém.

Nó là code **không có ngữ cảnh sản phẩm**.

Mục tiêu của sample:

* minh họa API
* đơn giản
* build được ở mọi nơi

Nó **không được thiết kế** để:

* chịu lỗi dài ngày
* dễ bảo trì
* dùng cho sản phẩm thực

Nếu bạn copy sample vào sản phẩm, sớm hay muộn bạn sẽ thấy… ngại khi người khác đọc code.

---

## Câu hỏi thực sự của loạt bài này

Không phải:

* MCP9808 dùng thế nào
* HTTP API ra sao

Mà là:

> **Làm sao thiết kế firmware để không sợ lỗi, không rối, không xấu hổ khi review?**

Khi hiểu được điều đó, công nghệ chỉ là chi tiết.

---

## Đối tượng đọc

Loạt bài này dành cho:

* người mới làm embedded nhưng cảm thấy “thiếu thiếu”
* người học Zephyr với ESP32
* freelancer muốn code sạch, dễ bàn giao
* người chuyển từ Arduino sang firmware nghiêm túc

Không cần cao siêu. Chỉ cần sẵn sàng suy nghĩ.

---

## Loạt bài này KHÔNG nhằm mục đích

* khoe trick
* tối ưu sớm
* viết code thông minh

Mục tiêu là:

* rõ ràng
* có kỷ luật
* dễ giải thích

Firmware tốt thường… rất chán.

---

## Bài toán xuyên suốt

Chúng ta sẽ dùng một ví dụ duy nhất:

> **Đọc nhiệt độ từ MCP9808 và gửi lên server**

Nghe đơn giản.

Nhưng chính những hệ thống “đơn giản” mới là nơi lỗi xuất hiện nhiều nhất.

---

## Những nguyên tắc sẽ lặp lại nhiều lần

* thiết bị không phải script
* lỗi là bình thường
* ownership phải rõ ràng
* callback không phải nơi quyết định logic
* retry là policy
* thời gian và năng lượng phải thấy được

Nếu bạn thấy lặp, nghĩa là bạn đang học đúng thứ quan trọng.

---

## Cấu trúc loạt bài

* Step 1: Tư duy như một thiết bị
* Step 2: Khung state machine
* Step 3: Tích hợp sensor đúng cách
* Step 4: Network không mất kiểm soát
* Step 5: Logging để giải thích hành vi
* Step 6: Retry và backoff
* Step 7: Quản lý năng lượng
* Step 8: Buffering và batching
* Step 9: OTA với MCUboot
* Step 10: Watchdog

Mỗi bài:

* độc lập
* có code đầy đủ
* không phá kiến trúc cũ

---

## Lời kết (Tiếng Việt)

> **Firmware tốt không phải để gây ấn tượng.
> Nó để sống sót, tự phục hồi và dễ bảo trì.**

Nếu bạn muốn học điều đó, hãy tiếp tục sang Step 1.
