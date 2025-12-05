---
layout: post
title: "Song Hùng Kỳ Hiệp: Đại Chiến Threading và Cú Tát 'Sấp Mặt' Của Lập Trình Nhúng"
---

Chào các chú em, lại là đại ca của các chú đây. Hôm nay, tôi sẽ không giảng lý thuyết suông nữa. Thay vào đó, tôi sẽ kể lại một câu chuyện "bi hài" có thật về hành trình của một chú "lính mới" (tôi sẽ giấu tên để giữ thể diện cho nó) khi cậu ta vật lộn với một bài toán threading cơ bản trong Zephyr RTOS.

Câu chuyện này là một bản tóm tắt "mặn như muối biển" về cuộc đối thoại giữa tôi và chú em đó. Nó không chỉ có code, mà còn có cả những cuộc tranh luận nảy lửa, những cú "lừa" của API, và những bài học xương máu về tư duy của một kỹ sư nhúng.

Ngồi thẳng lưng lên, pha một ly cà phê, và chuẩn bị tinh thần để lĩnh hội "chân kinh" nào!

## Phần 1: Thử thách "Bất Khả Thi" - Cú Tát Đầu Tiên Từ Hiện Thực

Mọi chuyện bắt đầu khi chú em kia vứt vào mặt tôi một đoạn code `main.c` và hồn nhiên hỏi: "Đại ca, sao nó không chạy?". Tôi liếc qua, và suýt nữa thì cái kính lão của tôi đã rớt xuống đất. Đây không phải là code, đây là một sự sỉ nhục đối với nghề lập trình!

![TheHell](https://i.pinimg.com/originals/bd/d6/9e/bdd69eec112566bf55d57821e49492b7.gif){: .mx-auto.d-block :}

```c
// Phiên bản "thảm họa" ban đầu
#include <zephyr/kernel.h>
#include <zephyr/drivers/sensor>
//...

void reader_thread_entry(void, void, void)
{
	mcp.sample_fetch();
	mcp.channel_get();
	// Then we will send the msg queue
	k_msleep(READER_SLEEP_MS);
}

void printer_thread_entry(void, void, void)
{
	// This is where we will print the msg queue
	printk();
	k_msleep(PRINTER_SLEEP_MS);
}

int main(void)
{
	const struct device *const mcp = DEVICE_DT_GET(DT_ALIAS(my_mcp9808));
    // ...
    // Tạo 2 thread rồi đi ngủ...
	return 0;
}
```

> **Phản ứng của đại ca (đã được kiểm duyệt):** "Chú em đang viết C hay đang ra lệnh cho robot bằng ngôn ngữ của riêng mình vậy? `mcp.sample_fetch()`? Chú tưởng đây là Python hay C++ à? Còn `printk()` không tham số thì nó in ra nỗi buồn của tôi à? Thread thì tạo ra để chạy một lần rồi 'thăng'? Chú đang làm phở bằng ống hút đấy à?"

Đoạn code này là một ví dụ điển hình của việc "ý tưởng lớn nhưng thực thi như trò đùa". Nó sai từ cú pháp cơ bản đến logic hệ điều hành. Đây chính là điểm khởi đầu cho cuộc phiêu lưu của chúng ta.

## Phần 2: Lập Kế Hoạch Tác Chiến - Hoa Sơn Luận Kiếm Bắt Đầu

Thay vì "vả" vào mặt chú em đó, tôi quyết định đóng vai một người thầy kiên nhẫn. Chúng tôi bắt đầu trận "Hoa Sơn Luận Kiếm" đầu tiên.

**Kế hoạch vạch ra:**
1.  **Sửa lỗi cú pháp:** Vứt ngay cái kiểu gọi hàm "sáng tạo" kia đi và thay bằng API chuẩn của Zephyr.
2.  **Thiết lập "Thùng thư":** Dùng `k_msgq` (Message Queue) để hai thread có thể "liếc mắt đưa tình" với nhau một cách an toàn.
3.  **Bắt Thread làm việc liên tục:** Thêm vòng lặp `while(1)`. Không có chuyện làm một lần rồi nghỉ hưu đâu!
4.  **Dạy Thread cách giao tiếp:** `reader_thread` sẽ dùng `k_msgq_put()` để "gửi thư". `printer_thread` sẽ dùng `k_msgq_get()` để "nhận thư".
5.  **Hoàn thiện cấu trúc dự án:** Dựng bộ khung `CMakeLists.txt`, `prj.conf`, và `.overlay` cho nó ra dáng một dự án đàng hoàng.

## Phần 3: Những Cuộc "Tranh Luận" Nảy Lửa - Các Hiệp Đấu

Đây là phần "mặn" nhất. Chú em kia không phải dạng vừa, cậu ta liên tục đặt ra những câu hỏi xoáy và thách thức lại cả "đại ca" này.

#### **Hiệp 1: Deadlock "Ảo Tưởng"**

> **Chú em:** "Đại ca ơi, lỡ `reader_thread` lỗi, nó đi ngủ. `printer_thread` chạy, nó gọi `k_msgq_get(..., K_FOREVER)`. Nó sẽ chiếm CPU và chờ mãi mãi. Deadlock rồi!"

Một câu hỏi cực kỳ sắc sảo! Nhưng lại sai về bản chất.

> **Đại ca đáp:** "Chú em nghĩ `K_FOREVER` là 'chiếm CPU và chờ' à? Sai! Nó có nghĩa là 'Nói với hệ điều hành: cho tôi đi ngủ, khi nào có thư thì gọi dậy'. Thread sẽ vào trạng thái 'ngủ' và tiêu thụ **0% CPU**. Đây chính là sự kỳ diệu của RTOS!"

![ExpandBrain](https://i.makeagif.com/media/10-28-2016/6xwwLc.gif){: .mx-auto.d-block :}

#### **Hiệp 2: `double` - Dùng Vây Cá Mập Nấu Canh Rau**

> **Chú em:** "Sao phải bỏ `double`? Dùng cho tiện mà."

> **Đại ca đáp:** "Chú đang so sánh một cái bánh mì (số nguyên) với một bữa tiệc 5 món (số thực) đấy. Trên vi điều khiển không có FPU, mọi phép toán với `double` đều phải giả lập bằng phần mềm, nó sẽ kéo cả một thư viện khổng lồ vào, làm chương trình của chú phình to và chạy như rùa bò. Dân chuyên nghiệp dùng mẹo in số nguyên: `%d.%02d`."

#### **Hiệp 3: `abs()` - Lập Trình Phòng Thủ hay Sự Hoang Tưởng?**

> **Chú em (gắt hơn):** "Nếu cảm biến nó gửi về dữ liệu 'dị' đến mức `val2` bị âm, thì đó là lỗi của cái cảm biến chết tiệt đó chứ! Code của mình chỉ cần phản ánh đúng cái sự 'hỏng' của nó thôi!"

> **Đại ca đáp:** "Chú đang nhầm lẫn! Dùng `abs()` không phải để 'chống đỡ cho một cái cảm biến chết tiệt'. Nó là để **tuân thủ đặc tả của `struct sensor_value`**. Nhiệm vụ của `printer_thread` là in ĐÚNG BẤT KỲ giá trị `sensor_value` hợp lệ nào. Đây là 'Lập trình phòng thủ'. Chúng ta xây những viên gạch LEGO hoàn hảo!"

## Phần 4: Màn Lột Xác Hoàn Hảo

Sau rất nhiều lần "đập đi xây lại", đây là "kiệt tác" cuối cùng. Nó không chỉ chạy, mà còn chạy một cách hiệu quả, an toàn và đúng với triết lý của RTOS.

![LifeGood](https://media.tenor.com/LLLJYVQJNVAAAAAM/chefs-kiss-french-chef.gif){: .mx-auto.d-block :}

```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/sensor>
#include <zephyr/drivers/i2c>
#include <stdio.h>
#include <stdlib.h>

#define STACK_SIZE 512
#define READER_SLEEP_MS 1000

#define MSG_SIZE sizeof(struct sensor_value)
#define MSG_QUEUE_SIZE 10

K_MSGQ_DEFINE(sensor_msgq, MSG_SIZE, MSG_QUEUE_SIZE, 4);

K_THREAD_STACK_DEFINE(printer_stack, STACK_SIZE);
K_THREAD_STACK_DEFINE(reader_stack, STACK_SIZE);

// ... Khai báo thread và device ...

void reader_thread_entry(void, void, void)
{
	int ret;
	struct sensor_value tmp;

	while (1) {
		ret = sensor_sample_fetch(mcp);
		if (ret < 0) {
			printk("Error fetching sample\r\n");
			continue;
		}

		ret = sensor_channel_get(mcp, SENSOR_CHAN_AMBIENT_TEMP, &tmp);
		if (ret < 0) {
			printk("Error getting channel data\r\n");
			continue;
		}

		ret = k_msgq_put(&sensor_msgq, &tmp, K_NO_WAIT);
		if (ret < 0) {
			printk("Failed to put message in queue\r\n");
		}

		k_msleep(READER_SLEEP_MS);
	}
}

void printer_thread_entry(void, void, void)
{
	struct sensor_value tmp;
	while (1) {
		if (k_msgq_get(&sensor_msgq, &tmp, K_FOREVER) == 0) {
			printk("Current temp is %d.%02d Celcius\r\n", tmp.val1, abs(tmp.val2 / 10000));
		}
	}
}

// ... Hàm main() ...
```

## Lời Kết: Nếu Code Mà Không Có Bug, Vậy Khác Gì Cá Muối?

Hành trình từ một đoạn "code rác" đến một chương trình hoàn chỉnh không chỉ là việc sửa lỗi cú pháp. Nó là một quá trình của **tư duy, lập luận, và tranh biện**. 

Hãy nhớ lấy, các chú em ạ. Code chỉ là công cụ. Tư duy của một kỹ sư mới là thứ tạo ra giá trị. Đừng bao giờ ngại đặt câu hỏi "tại sao". Vì nếu không có ước mơ trở thành "sư phụ", vậy kiếp dev của chú khác gì cá muối?

*Bài viết này có nhắc đến các khái niệm trong bài ["Xe độ vs. Xe đua F1"](https://azuch.io.vn/2025/12/04/dht11-pro-driver-analysis.html), các bạn có thể đọc lại để hiểu rõ hơn.*
