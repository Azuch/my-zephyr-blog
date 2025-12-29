---
layout: post
title: "From Demo to Production Firmware – Step 11: Future Features and Optional (Or not to) Evolutions"
date: 2025-12-30
categories: [embedded, zephyr, firmware, architecture]
---

> **Step 11 is not about adding more code.**
> It is about learning when *not* to add code — and how to evolve safely when you must.


# Part A – English Version

## Why Step 11 Exists

After Step 10, the system is complete in a very real sense:

* it controls its own flow
* it survives failures
* it retries politely
* it manages power
* it buffers data
* it updates safely
* it recovers from hangs

Many products ship successfully at this point.

So why Step 11?

Because real projects never stop here.

Clients ask for more.
Markets change.
Requirements grow.

Step 11 teaches you how to **extend without destroying**.

---

## The Most Important Lesson

> **A good architecture does not demand features.
> It resists them until they are justified.**

This step is about resisting unnecessary complexity.

---

## Category 1 – Reliability Extensions (Common)

These features increase survivability but add complexity.

### Flash-backed buffering

When to add:

* data loss is unacceptable
* power loss is frequent

What changes:

* buffer moves from RAM to flash
* wear-leveling becomes a concern

What stays the same:

* ownership (application)
* SEND drains, SAMPLE fills

Architecture survives unchanged.

---

### Reset reason tracking

When to add:

* fleet monitoring exists
* unexplained resets appear

What changes:

* boot code records reset cause

What stays the same:

* watchdog logic
* recovery behavior

---

## Category 2 – Power Optimization (After Measurement)

Never optimize power blindly.

### Deep sleep modes

When to add:

* battery-powered devices
* long WAIT periods

Debate:

* suspend RAM vs retain RAM
* wake sources

WAIT remains the only entry point.

---

### Adaptive sampling

When to add:

* environment changes slowly
* power is scarce

Policy belongs in application state machine.

Drivers remain dumb.

---

## Category 3 – Network & Data Evolution

### Batching and compression

When to add:

* payload overhead dominates
* connections are expensive

SEND logic extends, SAMPLE does not.

---

### Protocol changes (MQTT, CoAP)

When to add:

* backend requires it

What does NOT change:

* control flow
* retry policy
* buffering model

Only the net subsystem changes.

---

## Category 4 – Security & Trust

### Credential rotation

When to add:

* long-lived devices
* regulated environments

Must be:

* explicit
* observable
* reversible

Never silent.

---

### Secure provisioning

When to add:

* manufacturing scale
* device identity matters

Often happens before Step 1 in real products.

---

## Category 5 – Concurrency (Danger Zone)

### Multiple threads

When to add:

* strict latency requirements
* blocking is unacceptable

Cost:

* races
* complexity
* harder reviews

Most devices never need this.

---

## Category 6 – Developer Experience

### Documentation & diagrams

When to add:

* more than one developer
* future maintenance

Your blog series is already doing this.

---

### Architecture checklist

Create a reusable checklist:

* failure path explicit?
* retry policy centralized?
* WAIT owns sleep?
* watchdog fed on progress?

This scales *you* as an engineer.

---

## How Seniors Decide What to Add

They ask, in order:

1. What happens if we do nothing?
2. What breaks first?
3. Who pays the cost?
4. Can we explain this change?

If the answer is unclear, they wait.

---

## When to Stop

This is critical:

> **Not every product needs every feature.**

A calm, boring system that ships is better than a perfect one that never does.

---

## Final Thought (English)

> **Architecture is not about adding power.
> It is about controlling growth.**

Step 11 teaches restraint.

---

# Part B – Phiên bản tiếng Việt

## Vì sao cần Step 11

Sau Step 10, hệ thống đã đủ tốt để ship.

Nhưng dự án thực tế luôn tiếp tục phát triển.

Step 11 dạy cách **mở rộng mà không phá vỡ**.

---

## Bài học quan trọng nhất

> **Kiến trúc tốt không đòi hỏi feature.
> Nó chống lại feature cho đến khi thật sự cần.**

---

## Nhóm 1 – Tăng độ tin cậy

### Buffer lưu flash

Thêm khi:

* không được mất dữ liệu

Không đổi:

* ownership
* luồng SEND/SAMPLE

---

### Theo dõi reset

Giúp hiểu hệ thống ngoài thực tế.

---

## Nhóm 2 – Tối ưu năng lượng

Chỉ làm sau khi đo.

WAIT vẫn là trung tâm.

---

## Nhóm 3 – Network & dữ liệu

Thay protocol không thay kiến trúc.

---

## Nhóm 4 – Bảo mật

Credential, provisioning cần rõ ràng.

---

## Nhóm 5 – Concurrency

Nguy hiểm.

Chỉ dùng khi bắt buộc.

---

## Nhóm 6 – Trải nghiệm developer

Checklist giúp bạn không lặp sai lầm.

---

## Cách senior quyết định

Họ hỏi:

* nếu không làm thì sao?
* ai chịu hậu quả?

---

## Khi nào nên dừng

> **Firmware chán nhưng ship được tốt hơn firmware hoàn hảo nhưng không bao giờ xong.**

---

## Lời kết (Tiếng Việt)

> **Kiến trúc là nghệ thuật kiềm chế.**

Step 11 khép lại hành trình này.
