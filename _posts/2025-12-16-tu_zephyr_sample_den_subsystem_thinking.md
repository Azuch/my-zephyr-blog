---
layout: post
title: "Từ Zephyr Sample đến Subsystem Thinking"
date: 2025-12-15
author: ""
categories: [embedded, firmware, zephyr]
tags: [zephyr, firmware, subsystem, mcuboot, esp32]
---

# Từ Zephyr Sample đến Subsystem Thinking  
## Một hành trình “đỡ xấu hổ” khi viết firmware

Khi mới học Zephyr, mình làm đúng như những gì tài liệu và tutorial khuyên:

- Theo Digi-Key tutorial  
- Chạy sample  
- Copy code từ `samples/`  
- Test thấy chạy → OK

Mình dùng ESP32-S3 làm board học tập, Zephyr làm RTOS chính. Mọi thứ có vẻ ổn… cho đến khi mình đọc code của senior.

Code của senior không giống sample Zephyr.  
Nó dài hơn.  
Nó nhiều state hơn.  
Nó “khó đọc” hơn.

Nhưng kỳ lạ là: **nó nhìn rất đàng hoàng**.

Lúc đó trong đầu mình xuất hiện một câu hỏi rất khó chịu:

> *“Zephyr là RTOS lớn, có hàng nghìn contributor.  
> Vậy tại sao sample của Zephyr lại nhìn… tệ hơn code của senior?”*

Câu hỏi này dẫn tới cả một chuỗi nhận thức mới về **firmware production**, **subsystem pattern**, và **thế nào là code không làm mình xấu hổ khi người khác đọc**.

---

## Sample của Zephyr sinh ra để làm gì?

Câu trả lời ngắn gọn là:

> **Zephyr sample không sinh ra để làm firmware production.**

Nghe có vẻ hiển nhiên, nhưng mình đã không thực sự *hiểu* điều đó cho đến khi đối chiếu hai cách tiếp cận.

### Zephyr sample tối ưu cho ai?

Zephyr sample được viết cho:

- Người mới
- Người đang học API
- Người cần “một ví dụ chạy được”

Vì vậy, sample ưu tiên:

- Luồng code tuyến tính
- Ít abstraction
- Ít state machine
- Dễ debug từng dòng

Ví dụ điển hình là HTTP client sample:

- Tạo socket
- Connect
- Gửi request
- Close socket
- Lặp lại cho IPv4 / IPv6 / GET / POST / chunked

Code bị lặp, dài, và “xấu” — **nhưng cố tình xấu**.

Bởi vì mục tiêu của sample là:

> *“Cho bạn thấy API được dùng như thế nào, ít nhất một lần.”*

Không phải:

- Khả năng mở rộng
- Khả năng recover
- Chạy 24/7
- Maintain trong nhiều năm

---

## Code của senior: tại sao nhìn khác?

Code HTTP client của senior mình thì hoàn toàn khác:

- Có `context`
- Có `state machine`
- Có `run()` function
- Callback chỉ báo sự kiện, không điều khiển flow
- Cleanup tập trung ở một chỗ

Nó không giống sample.  
Nó giống… **subsystem**.

Và đó là lúc mình bắt đầu hiểu ra:

> **Zephyr sample dạy “cách dùng API”.  
> Subsystem dạy “cách suy nghĩ như firmware engineer”.**

---

## Subsystem pattern là gì?

Sau một hồi đào sâu (và hỏi rất nhiều câu “ngây thơ”), mình rút ra được vài đặc điểm rất rõ ràng:

Một subsystem-style module luôn có:

1. **State rõ ràng, explicit**
2. **Context trung tâm, sở hữu tài nguyên**
3. **Một hàm `run()` hoặc `process()` để tiến từng bước**
4. **Callback không chứa logic chính**
5. **Cleanup tập trung, không rải rác**
6. **Control flow “nhàm chán” một cách có chủ ý**

Nghe thì đơn giản, nhưng hầu hết code demo / tutorial đều **không có những thứ này**.

---

## Ví dụ nhỏ: Blink LED cũng nói lên rất nhiều

Blink LED là bài “hello world” của firmware.  
Nhưng chính vì nó đơn giản, nên rất dễ viết… dở.

### Blink kiểu tutorial

```c
void main(void)
{
    gpio_pin_configure(dev, PIN, GPIO_OUTPUT_ACTIVE);

    while (1) {
        gpio_pin_toggle(dev, PIN);
        k_sleep(K_MSEC(500));
    }
}
```

Chạy được.  
Nhưng nếu senior nhìn vào, họ sẽ nghĩ ngay:

- Logic bị nhốt trong `main`
- Không có lifecycle
- Không có state
- Không mở rộng được

### Blink kiểu subsystem thinking

```c
enum led_state {
    LED_STATE_UNINIT,
    LED_STATE_OFF,
    LED_STATE_ON,
    LED_STATE_ERROR,
};

struct led_ctx {
    const struct device *gpio;
    gpio_pin_t pin;
    enum led_state state;
};
```

Có state.  
Có context.  
Có ownership.

Main chỉ việc:

```c
while (1) {
    led_toggle(&ctx);
    k_sleep(K_MSEC(500));
}
```

Đây là khác biệt giữa:

- *“Code chạy được”*
- *“Code có thể sống lâu”*

---

## Vì sao phải đọc subsystem (ví dụ MCUboot)?

Một câu hỏi mình từng hỏi thẳng:

> *“Tại sao phải đọc subsystem? Sample không đủ à?”*

Câu trả lời rất đau nhưng rất thật:

> **Sample không dạy judgment.  
> Subsystem dạy judgment.**

Subsystem là nơi:

- Code phải sống nhiều năm
- Lỗi xảy ra là brick thiết bị
- Reviewer rất khắt khe
- Không có chỗ cho “chắc là không sao”

MCUboot là ví dụ hoàn hảo.

---

## Đọc MCUboot như thế nào cho đúng?

Sai lầm lớn nhất là:

> Cố hiểu hết crypto, flash, hash, signature…

Không cần.

Cách đọc đúng là:

- Đọc **luồng điều khiển**
- Đọc **state**
- Đọc **error path**
- Đọc **cleanup**

Thứ tự mình được hướng dẫn:

1. `bootutil_public.h` – contract
2. `bootutil.c` – flow tổng quát
3. `loader.c` – quyết định state
4. `swap_*` – xử lý tiến trình dài và có thể bị reset

Luôn tự hỏi:

- State nằm ở đâu?
- Nếu mất điện giữa chừng thì sao?
- Cleanup xảy ra ở đâu?
- Vì sao code lại “dài dòng” như vậy?

Sau khi đọc MCUboot, mình nhận ra:

> Code của senior mình **không copy MCUboot**,  
> nhưng **suy nghĩ giống hệt MCUboot**.

---

## Vậy Zephyr có “best practice” không?

Có. Nhưng không phải trong sample.

Zephyr dạy best practice qua:

- Subsystem
- Core code
- Coding guideline
- Cách maintain API lâu dài

Sample chỉ là:

> *Executable documentation.*

Một khi mình hiểu điều đó, mình không còn khó chịu khi thấy sample “xấu” nữa.

---

## Lộ trình học lại (cho chính mình)

Sau tất cả, mình rút ra lộ trình học firmware “đỡ xấu hổ” như sau:

1. **Dùng sample để học API**
2. **Dùng subsystem để học tư duy**
3. **Viết code như thể nó sẽ chạy 3 năm**
4. **Luôn hỏi: nếu lỗi xảy ra lúc 3h sáng thì sao?**
5. **Ưu tiên code boring hơn code clever**

ESP32-S3 vẫn là board học rất tốt.  
Zephyr vẫn là RTOS rất tốt.

Nhưng cách mình **nhìn code** đã thay đổi hoàn toàn.

---

## Kết luận

Nếu có một câu tóm gọn cả hành trình này, thì là:

> **Firmware tốt không phải là firmware ngắn.  
> Firmware tốt là firmware mà người khác đọc không phải hỏi thêm câu nào.**

Sample giúp bạn bắt đầu.  
Subsystem giúp bạn trưởng thành.

Và code của senior — nếu nhìn “xấu” lúc đầu — rất có thể là vì bạn đang **bước sang một level mới**.

---

Nếu bạn đang ở giai đoạn giống mình:

- Đã qua tutorial
- Đã chạy sample
- Bắt đầu thấy code của người khác “khác khác”

Thì chúc mừng.  
Bạn đang học **firmware engineering**, không còn chỉ là **firmware programming** nữa.
