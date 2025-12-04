---
layout: post
title:  "Chào Zephyr, Tạm Biệt Thế Giới!"
date:   2025-12-04 12:00:00 +0700
categories: zephyr embedded
---

Hello, World? Xưa rồi Diễm! Dân nhúng chúng tôi chào thế giới bằng cách... nháy một cái đèn LED. Và hôm nay, chúng ta sẽ làm điều đó với một "tay to" trong làng RTOS: **Zephyr**!

Chào mừng đến với bài viết đầu tiên trên blog, nơi chúng ta sẽ cùng nhau khám phá thế giới hệ thống nhúng đầy ma thuật (và đôi khi là... ma ám) với Zephyr RTOS. Nếu bạn là một người tò mò, một sinh viên, hay một kỹ sư đang tìm kiếm một cơn gió mới (pun intended!), thì bạn đã đến đúng nơi rồi đấy.

### Zephyr RTOS: Nó là cái gì mà ghê vậy?

Nói một cách đơn giản, Zephyr là một hệ điều hành thời gian thực (Real-Time Operating System - RTOS) mã nguồn mở, được thiết kế để chạy trên các thiết bị có tài nguyên hạn chế. Tưởng tượng bạn có một con vi điều khiển bé tí tẹo, và bạn muốn nó làm hàng tá việc cùng một lúc một cách đáng tin cậy. Đó là lúc Zephyr bước vào và tỏa sáng.

Nó được chống lưng bởi Linux Foundation, nên anh em cứ yên tâm về độ "uy tín" và sự phát triển trong tương lai nhé.

### Màn Chào Sân Kinh Điển: Blinky!

Nói nhiều không bằng làm thật. Dưới đây là đoạn code "huyền thoại" giúp bạn bắt một con LED trên board mạch phải "chớp mắt" theo ý mình. Đây là ví dụ cho một board ESP32 quen thuộc.

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>

/* 1000 msec = 1 sec */
#define SLEEP_TIME_MS   1000

/* The devicetree node identifier for the "led0" alias. */
#define LED0_NODE DT_ALIAS(led0)

static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(LED0_NODE, gpios);

void main(void)
{
    int ret;

    if (!gpio_is_ready_dt(&led)) {
        return;
    }

    ret = gpio_pin_configure_dt(&led, GPIO_OUTPUT_ACTIVE);
    if (ret < 0) {
        return;
    }

    while (1) {
        ret = gpio_pin_toggle_dt(&led);
        if (ret < 0) {
            return;
        }
        k_msleep(SLEEP_TIME_MS);
    }
}
```

Đừng lo lắng nếu bạn chưa hiểu hết các `DT_ALIAS` hay `gpio_dt_spec`. Chúng ta sẽ "mổ xẻ" chúng trong các bài viết sau. Cứ coi như đây là một câu thần chú, và nó hoạt động!

### Rồi sao nữa?

Đây chỉ là phát súng mở màn. Trong các bài viết tiếp theo, chúng ta sẽ đi sâu hơn vào các chủ đề hardcore hơn: Device Tree, trình điều khiển (drivers), Bluetooth Low Energy, và nhiều thứ hay ho khác.

Hãy ở lại và cùng tôi phiêu lưu trong thế giới của Zephyr nhé!
