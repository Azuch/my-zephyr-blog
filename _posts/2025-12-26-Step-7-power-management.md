---
layout: post
title: "From Demo to Production Firmware – Step 7: Power Is a First-Class Constraint"
date: 2025-12-26
categories: [embedded, firmware, architecture, power]
---

> **Step 7 is where firmware stops pretending power is infinite.**  
> Power is not an optimization — it is a design constraint.

# Part A – English Version

## Why Step 7 Exists

In demos, power is ignored.
In production, power defines behavior.

If your firmware:
- retries aggressively
- wakes up too often
- spins while waiting
- polls instead of sleeps

Then battery life is not a parameter — it is an accident.

Step 7 exists to make power **explicit and intentional**.

---

## Power Is Not Just Sleep()

Junior code often treats power like this:

```c
k_sleep(K_SECONDS(5));
