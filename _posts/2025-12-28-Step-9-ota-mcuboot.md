---
layout: post
title: "From Demo to Production Firmware – Step 9: OTA Is Controlled Self-Modification"
date: 2025-12-28
categories: [embedded, firmware, architecture, ota]
---

> **OTA is not a feature.**  
> It is the most dangerous thing your firmware can do.
# Step 9 – OTA Updates with MCUboot (Controlled Self-Modification)

After Step 8, the device is:

* reliable
* observable
* power-aware
* outage-tolerant

Step 9 adds the **most dangerous capability** a device can have:

> **The ability to modify its own firmware.**

This step is about doing that **safely, predictably, and explainably**.

---

## 1. Purpose of Step 9

OTA is not a feature. It is **risk management**.

Step 9 answers:

> **"How do we update firmware without bricking devices in the field?"**

MCUboot exists to answer exactly this question.

---

## 2. Why MCUboot Is a Subsystem, Not a Library

MCUboot is:

* a separate executable
* with its own state machine
* with strict ownership of flash

This matches everything you have learned so far.

Key idea:

> **The application never directly controls boot decisions.**

---

## 3. Roles and Responsibilities

### MCUboot (Bootloader)

Owns:

* flash layout
* image validation
* rollback decisions
* swap logic

Guarantees:

* a valid image always boots
* failed updates roll back

---

### Application (Your firmware)

Owns:

* downloading new firmware
* storing it in the update slot
* requesting an upgrade
* reporting status

Never decides *how* boot happens.

---

## 4. Mental Model (Two Images)

Typical MCUboot layout:

```text
+------------------+  
| MCUboot          |  
+------------------+  
| Image A (running)|  <-- currently active
+------------------+  
| Image B (update) |  <-- candidate
+------------------+
```

Only MCUboot decides which image is active.

---

## 5. OTA Is a Separate Failure Domain

OTA introduces new failures:

* download interruption
* power loss mid-update
* corrupted image
* incompatible firmware

Therefore OTA must:

* integrate with recovery logic
* not interfere with normal operation

---

## 6. Where OTA Fits in the Existing Architecture

OTA does **not** change the core state machine.

It is treated as a **background task**:

* Uses network subsystem
* Uses buffering concepts
* Uses WAIT for pacing

No new top-level states are required.

---

## 7. OTA Context

```c
struct ota_ctx {
    bool update_available;
    bool download_in_progress;
    size_t bytes_written;
};
```

OTA state is owned by the application, not MCUboot.

---

## 8. OTA Flow (High Level)

1. Check for update (periodically)
2. Download firmware image in chunks
3. Write image to MCUboot secondary slot
4. Mark image as pending
5. Reboot
6. MCUboot validates and swaps
7. Application confirms success

Each step can fail safely.

---

## 9. Integration with State Machine

OTA activity happens during:

* IDLE
* WAIT

Never during:

* SENSOR_INIT
* SEND
* critical paths

This avoids disrupting core functionality.

---

## 10. Failure Handling During OTA

Key rule:

> **OTA failure must never affect current firmware execution.**

Examples:

* Download fails → retry later
* Power loss → boot old image
* Validation fails → rollback

All failures are recoverable.

---

## 11. Requesting an Upgrade (Application Side)

Conceptually:

```c
boot_request_upgrade(BOOT_UPGRADE_TEST);
reboot();
```

After reboot:

* MCUboot tests new image
* Rollback if not confirmed

---

## 12. Confirming an Update

On successful boot:

```c
boot_write_img_confirmed();
```

This tells MCUboot:

> “This image is good. Keep it.”

Without confirmation, rollback occurs automatically.

---

## 13. Interaction with Retry & Power Policies

OTA obeys existing rules:

* backoff on failures
* sleep during WAIT
* log transitions

OTA never introduces special-case timing.

---

## 14. Logging OTA Events

Log only milestones:

* Update available
* Download started
* Download completed
* Upgrade requested
* Image confirmed

Never log every chunk.

---

## 15. What Step 9 Deliberately Avoids

* Delta updates
* A/B/C images
* Compression
* Encryption details

Those are advanced topics added later if needed.

---

## 16. Success Criteria for Step 9

Step 9 is complete when:

* Device can update itself safely
* Power loss during OTA is harmless
* Rollback works automatically
* OTA logic is explainable

---

## 17. Architectural Status After Step 9

After Step 9, the device:

* can evolve after deployment
* protects itself from bad updates
* matches industry best practice

This is the baseline for modern IoT products.

---

## 18. Final Note

MCUboot works well here because:

* you already think in subsystems
* you already centralize policy
* you already respect ownership

This is not an accident.
