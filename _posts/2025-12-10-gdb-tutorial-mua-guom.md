---
layout: post
title: "Bài 4: Gươm Báu Đã Sẵn, Giờ Là Lúc 'Múa'! - Thực Hành GDB Với Dự Án Blinky"
---

Ở các phần trước, chúng ta đã tốn không ít mồ hôi công sức để rèn nên một thanh ["Ỷ Thiên Kiếm" (cài đặt thành công OpenOCD)](/2025/12/09/openocd-espressif-love-triangle.html) và học được bộ "Cửu Âm Chân Kinh" (hiểu rõ [bản chất JTAG](/2025/12/08/esp32-s3-jtag-cuu-roi.html)). Giờ là lúc thanh gươm báu được tuốt ra khỏi vỏ.

Hôm nay, đại ca sẽ không nói lý thuyết nữa, mà sẽ kể lại một màn "truyền công" thực tế cho một chú em "gà mờ". Qua đó, các chú sẽ thấy được những đường GDB cơ bản nhất được "múa" trên một bài toán kinh điển: làm cho con LED "lên bờ xuống ruộng".

`[MEME: Một nhân vật võ hiệp đang múa kiếm cực ngầu]`

---

## Màn 1: Dựng "Võ Đài" & Sai Lầm Ngớ Ngẩn

Chuyện bắt đầu khi tôi quăng cho chú em một cái task "dễ như ăn cháo": "Ê, tạo cho anh một project Blinky trên con ESP32-S3 đi, rồi dùng JTAG để debug xem nó chạy thế nào". Chú em hí hửng nhận lời, một lúc sau nó vứt cho tôi một "đống" và hồn nhiên hỏi: "Đại ca, em làm y hệt trên mạng, sao nó không chạy?".

Tôi liếc mắt qua, và đây là "kiệt tác" của nó.

### Dựng "Võ Đài" (Các file cấu hình)

Chú em cũng biết tạo file, nhưng chắc là chỉ copy paste thôi chứ không hiểu gì.

**`prj.conf`:**
```ini
# Bật GPIO và Logging
CONFIG_GPIO=y
CONFIG_LOG=y
```

**`boards/esp32s3_devkitc.overlay`:**
```devicetree
/ {
    aliases {
        led0 = &my_led;
    };
};

&gpio0 {
    my_led: led_0 {
        gpios = <&gpio0 2 GPIO_ACTIVE_HIGH>;
        label = "User LED";
    };
};
```

### "Hạt Sạn" Trong Bát Cơm - `src/main.c`

Và đây, "tuyệt phẩm" `main.c` mà chú em nó chép được:

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>

// Lỗi sai #2: Lồng ALIAS vào thẳng macro, một sự sáng tạo đi vào lòng đất
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(DT_ALIAS(led0));

// Lỗi sai #1: Không "khai báo hộ khẩu" cho module
// LOG_MODULE_REGISTER(main, LOG_LEVEL_DBG);

void main(void)
{
	int ret;

    // Lỗi sai #3: Dùng hàm is_ready() sai phiên bản
	if (!gpio_is_ready(led.port)) {
		// LOG_ERR(...)
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
		k_msleep(1000);
	}
}
```

---

## Màn 2: Đại Ca Ra Tay "Chẩn Bệnh"

Nhìn vào "thảm họa" này, tôi chỉ muốn hét lên: "Trời ơi, nó là một thằng ngốc!". Nhưng thôi, với vai trò là một người thầy, tôi phải kiên nhẫn.

`[MEME: Any meme thể hiện sự bất lực, facepalm]`

> **Đại ca:** "Chú em, lại đây anh bảo. Code của chú có 3 'tử huyệt', y hệt như 3 cái huyệt đạo bị điểm sai, khiến nội công tẩu hỏa nhập ma, chương trình không thể chạy được."

### Lỗi #1: Vô Danh Vô Phận - Thiếu `LOG_MODULE_REGISTER`
> **Đại ca:** "Chú thấy cái `LOG_ERR` mà anh comment lại không? Chú mà bật nó lên là trình biên dịch nó chửi cho sấp mặt ngay. Tại sao? Vì chú chưa 'khai báo hộ khẩu' cho cái file `main.c` này với hệ thống log của Zephyr. Mỗi file muốn dùng `LOG_...` đều phải có dòng `LOG_MODULE_REGISTER(tên_module, LOG_LEVEL_DBG)` ở đầu. Nó như cái thẻ nhân viên vậy, không có thẻ thì bảo vệ nó không cho vào!"

### Lỗi #2: Tẩu Hỏa Nhập Ma - Sai Cú Pháp `GPIO_DT_SPEC_GET`
> **Đại ca:** "Há, cái dòng này mới thật là 'đê tiện' chứ hả: `GPIO_DT_SPEC_GET(DT_ALIAS(led0))`! Chú em à, `DT_ALIAS(led0)` nó trả về một cái **node identifier**, một cái định danh cho node thôi. Còn cái hàm `GPIO_DT_SPEC_GET` nó cần một cái **node identifier** làm tham số, chứ nó không nuốt cả một cái macro `DT_ALIAS` vào bụng được!"
>
> "Cách làm đúng của các cao thủ là phải tách bạch, rành mạch:"
> ```c
> // Bước 1: Lấy cái định danh của node từ alias
> #define LED0_NODE DT_ALIAS(led0)
>
> // Bước 2: Dùng cái định danh đó để lấy spec
> static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(LED0_NODE, gpios);
> ```
> "Thấy sự khác biệt chưa? Rõ ràng, tường minh! Code là phải như vậy!"

### Lỗi #3: Râu Ông Nọ Cắm Cằm Bà Kia - `gpio_is_ready`
> **Đại ca:** "Cuối cùng, cái hàm `gpio_is_ready(led.port)`. Chú dùng `gpio_dt_spec` là một struct xịn sò, hiện đại của Zephyr. Mà lại đi gọi hàm `gpio_is_ready()` của thời 'đồ đá', chỉ nhận vào một con trỏ `device*`. Nó giống như chú có một cái chìa khóa xe Ferrari (`struct gpio_dt_spec`) mà lại cố nhét vào ổ khóa xe đạp (`device*`) vậy."
>
> "Khi đã dùng `_dt` spec, thì tất cả các hàm liên quan cũng phải có đuôi `_dt` cho nó đồng bộ! Phải là `gpio_is_ready_dt(&led)`."

### Phiên bản "Hoàn Hảo"
Sau một hồi "chửi yêu", tôi đưa cho chú em phiên bản đã được sửa:

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/log/log.h>

// Đã có "thẻ nhân viên"!
LOG_MODULE_REGISTER(main_app, LOG_LEVEL_DBG);

#define LED0_NODE DT_ALIAS(led0)
#define SLEEP_TIME_MS 1000

// Đã tách bạch, rõ ràng
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(LED0_NODE, gpios);

void main(void)
{
	int ret;

    // Đã dùng đúng hàm _dt
	if (!gpio_is_ready_dt(&led)) {
		LOG_ERR("GPIO device %s is not ready", led.port->name);
		return;
	}

	ret = gpio_pin_configure_dt(&led, GPIO_OUTPUT_ACTIVE);
	if (ret < 0) {
		LOG_ERR("Failed to configure GPIO pin");
		return;
	}

	LOG_INF("Blinky sample started!");

	while (1) {
		ret = gpio_pin_toggle_dt(&led);
		if (ret < 0) {
			LOG_ERR("Failed to toggle GPIO pin");
			return;
		}
		k_msleep(SLEEP_TIME_MS);
	}
}
```

---

## Màn 3: "Truyền Thụ Võ Công" - Các Chiêu Thức GDB

Code đã chuẩn, giờ là lúc "múa gươm".
1.  **Build code:** `west build -b esp32s3_devkitc .`
2.  **Mở 2 Terminal:** Một cho OpenOCD, một cho GDB.
    *   Terminal 1: `openocd -f board/esp32s3-builtin.cfg`
    *   Terminal 2: `west debug`

Sau khi chạy `west debug`, chương trình sẽ tự động dừng lại ở `main`, và dấu nhắc `(gdb)` hiện ra, sẵn sàng nhận lệnh.

> **Đại ca:** "Giờ thì nhìn cho kỹ đây, ta sẽ dạy cho cậu 5 chiêu GDB căn bản nhất để hành tẩu giang hồ."

### Chiêu 1: `list` - "Soi Kịch Bản"
> "Chiêu này giúp chú xem code. Gõ `l` hoặc `list` để xem 10 dòng xung quanh vị trí hiện tại. Gõ `list main` để xem code của hàm `main`, hoặc `list 35` để xem code quanh dòng 35. Biết người biết ta, trăm trận trăm thắng!"

### Chiêu 2: `break` - "Đặt Bẫy"
> "Chiêu này dùng để đặt 'bẫy' ở những nơi hiểm yếu. Gõ `break 40` để đặt bẫy ở dòng 40 (dòng `gpio_pin_toggle_dt`). Khi chương trình chạy đến đây, nó sẽ dừng lại. Gõ `info b` để xem có bao nhiêu cái bẫy đã được giăng."

### Chiêu 3: `continue` - "Thả Hổ Về Rừng"
> "Gõ `c` hoặc `continue`. Chương trình sẽ chạy tự do cho đến khi... sập bẫy. Nó sẽ dừng ngay tại dòng 40 mà chúng ta đã đặt."

### Chiêu 4: `print` - "Móc Túi Xem Giấy Tờ"
> "Khi đã 'bắt' được chương trình, dùng chiêu này để 'lục soát' nó. Gõ `p led` để xem nội dung của cái struct `led`. Chú sẽ thấy nó chứa con trỏ `port` và số `pin`."

### Chiêu 5: `next` & `step` - "Đi Theo Dấu Chân"
> "Đây là hai chiêu dễ nhầm nhất. `n` (next) là đi qua một dòng, coi như nó là một bước. Nếu dòng đó là một hàm, nó sẽ chạy hết hàm đó rồi mới dừng. Còn `s` (step) cũng là đi một bước, nhưng nếu gặp hàm, nó sẽ 'nhảy' vào bên trong hàm đó. Dùng `s` khi muốn xem xét nội tình bên trong một hàm, dùng `n` khi chỉ muốn đi tiếp."

---

## Lời Kết: "Giờ Thì Ai Cũng Làm Được!"

> **Đại ca:** "Đấy, chỉ với 5 chiêu `l`, `b`, `c`, `p`, `n/s` là chú đã có thể ngao du trong thế giới GDB rồi. Giờ thì tự mình 'múa' tiếp đi, đặt thêm bẫy, 'in' thêm vài biến nữa xem sao."

Từ giờ, bug không còn là nỗi sợ, mà là một con mồi trong tầm ngắm của đại ca. Chúc mừng, đại ca đã tốt nghiệp khóa vỡ lòng GDB!

`[MEME: Một nhân vật game lên level hoặc nhận được item quý]`
