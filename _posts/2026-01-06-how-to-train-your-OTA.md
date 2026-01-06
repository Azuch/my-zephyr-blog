---
layout: post
title: "From Demo to Production Firmware – Step 9.1: How to train your OTA"
date: 2026-01-06
categories: [embedded, zephyr, firmware, architecture]
---

> **Step 9.1 is just add more info to MCUBoot and OTA.**
> **MCUboot và Zephyr: Bí kíp luyện Failsafe OTA Updates cho hệ thống Embedded**

Chào các huynh đệ đã quay trở lại với series "Embedded cho người lười nhưng thích tỏ ra nguy hiểm". Hôm nay, chúng ta sẽ mổ xẻ một trong những chủ đề vừa "sexy" vừa "nhức nách" nhất trong giới embedded: **OTA - Over-the-Air Updates**. Cụ thể hơn, chúng ta sẽ nói về "vệ sĩ" MCUboot và cách nó kết hợp với Zephyr RTOS để thực hiện những màn cập nhật firmware thần sầu mà không sợ "brick" device.

Nỗi ám ảnh lớn nhất của một dev embedded là gì? Là sau khi bấm nút "Update", con device yêu quý trị giá vài chục đến vài trăm đô của khách hàng bỗng hóa thành cục gạch, không hơn không kém. MCUboot sinh ra là để nỗi đau đó không bao giờ xảy ra.

Trong bài viết này, chúng ta sẽ cùng nhau:
1.  Hiểu rõ MCUboot là cái quái gì và tại sao nó là vệ sĩ đáng tin cậy.
2.  Xem lại hành trình chúng ta đã "thuần hóa" MCUboot trong Zephyr như thế nào.
3.  Vẽ ra toàn bộ kịch bản một pha OTA từ A-Z.
4.  Bàn về các "luật chơi" của giới giang hồ OTA để trông cho chuyên nghiệp.

---

## Phần 1: MCUboot - Gã vệ sĩ "Bất Tử"

Tưởng tượng MCUboot là một gã bảo kê ở cửa một quán bar siêu sang. Nhiệm vụ của nó rất đơn giản:
1.  Chỉ cho phép "khách xịn" (firmware đã được ký tên, xác thực) vào quán.
2.  Nếu có "khách xịn" mới muốn vào, nó sẽ tạm đưa vào một "phòng chờ" (slot-1).
3.  Khi có lệnh, nó sẽ hoán đổi vị trí giữa "khách cũ" trong quán (slot-0) và "khách mới" trong phòng chờ.
4.  **Và đây là điều ăn tiền:** Sau khi khách mới vào quán, nó sẽ tạm thời "để mắt" tới. Nếu vị khách mới này không tự "xác nhận" (confirm) là mình "ổn", trong lần khởi động tiếp theo, gã bảo kê sẽ tống cổ vị khách mới này ra và mời vị khách cũ quay trở lại.

Đây chính là cơ chế **failsafe** thần thánh của MCUboot. Nó đảm bảo rằng kể cả khi bạn có lỡ tay update một bản firmware lỗi, thiết bị vẫn có thể tự động quay về phiên bản ổn định trước đó.

Các khái niệm cốt lõi:
*   **Primary Slot (Slot 0):** "Quán bar", nơi chứa firmware đang hoạt động.
*   **Secondary Slot (Slot 1):** "Phòng chờ", nơi chứa firmware mới được tải về.
*   **Image Swapping:** Hành động hoán đổi firmware giữa hai slot.
*   **Image Confirmation:** Hành động firmware mới phải tự thực hiện để chứng minh mình "khỏe mạnh". Nếu không, MCUboot sẽ thực hiện **revert** (quay lại phiên bản cũ).
*   **Scratch Partition:** Một "sân trung chuyển" nhỏ để MCUboot sử dụng trong quá trình swap, đảm bảo quá trình không bị gián đoạn.

---

## Phần 2: Thuần hóa MCUboot trong Zephyr - Một câu chuyện thực tế

Để MCUboot hoạt động, chúng ta không chỉ nói mồm. Chúng ta phải "động tay động chân" vào cấu hình của project. Dưới đây là tóm tắt những gì chúng ta đã làm trong project `new_skeleton`:

### Bước 1: Lời thề Kconfig

Chúng ta đã khai báo với Zephyr rằng: "Từ nay, hãy dùng MCUboot". Điều này được thực hiện bằng cách thêm một loạt "bùa chú" vào file `prj.conf`:

```conf
# Bật MCUboot làm bootloader
CONFIG_BOOTLOADER_MCUBOOT=y

# Bật những thứ cần thiết cho ESP32 và quản lý phân vùng
CONFIG_PARTITION_MANAGER_ENABLED=y
CONFIG_SOC_FLASH_ESP32=y

# Bật trình quản lý ảnh và giao tiếp với nó
CONFIG_IMG_MANAGER=y
CONFIG_MCUBOOT_IMG_MANAGER=y
CONFIG_MCUMGR=y
CONFIG_MCUMGR_CMD_IMG_MGMT=y

# Bật shell để ra lệnh cho dễ
CONFIG_SHELL=y
CONFIG_MCUMGR_SHELL=y

# Quan trọng: Cấu hình ký tên và xác thực
CONFIG_MCUBOOT_SIGNATURE_TYPE_RSA_2048=y
CONFIG_MCUBOOT_SIGNATURE_KEY_FILE="bootloader/mcuboot/root-rsa-2048.pem"
```

### Bước 2: Phân lô bán nền (Flash Partitioning)

Tiếp theo, chúng ta phải "chia đất" cho bộ nhớ flash. Việc này được thực hiện trong file Device Tree Overlay (`boards/esp32s3_devkitc.overlay`).

```dts
&flash0 {
	partitions {
		compatible = "fixed-partitions";
		#address-cells = <1>;
		#size-cells = <1>;

		/* Phân vùng cho MCUboot, 64KB */
		boot_partition: partition@0 {
			label = "mcuboot";
			reg = <0x00000000 0x00010000>;
			read-only;
		};

		/* Phân vùng cho app chính (slot-0) */
		slot0_partition: partition@10000 {
			label = "image-0";
			reg = <0x00010000 0x00180000>;
		};

		/* Phân vùng cho app mới (slot-1) */
		slot1_partition: partition@190000 {
			label = "image-1";
			reg = <0x00190000 0x00180000>;
		};

		/* Phân vùng cho MCUboot làm việc */
		scratch_partition: partition@310000 {
			label = "image-scratch";
			reg = <0x00310000 0x00020000>;
		};
	};
};
```

### Bước 3: Giấy thông hành (Image Signing)

Mỗi bản build giờ đây sẽ được "ký tên" bằng một chìa khóa riêng tư. MCUboot sẽ dùng chìa khóa công khai để xác thực chữ ký này. Đây là cơ chế chống hàng giả, đảm bảo chỉ có firmware "chính chủ" từ sếp mới được cài đặt.

Chúng ta đã trỏ đến file `bootloader/mcuboot/root-rsa-2048.pem`. Tin vui là Zephyr sẽ tự tạo file này trong lần build đầu tiên nếu nó chưa tồn tại.

### Bước 4: Cái bắt tay lịch sử (Application Confirmation)

Đây là mảnh ghép cuối cùng. Firmware mới sau khi khởi động phải "báo cáo" cho MCUboot rằng "Tôi ổn, không cần quay lại bản cũ đâu". Chúng ta đã thêm đoạn code này vào `src/app.c`:

```c
#include <zephyr/dfu/mcuboot.h>

static void confirm_image(void)
{
	if (!boot_is_img_confirmed()) {
		int ret = boot_write_img_confirmed();
		if (ret) {
			LOG_ERR("Không thể confirm image: %d", ret);
		} else {
			LOG_INF("Đã confirm image thành công!");
		}
	} else {
		LOG_INF("Image đã được confirm từ trước.");
	}
}

void app_run(void) {
    // ...
	confirm_image();
    // ...
}
```

---

## Phần 3: Toàn cảnh một phi vụ OTA

Bây giờ, hãy xâu chuỗi mọi thứ lại và xem một phi vụ OTA hoàn chỉnh diễn ra như thế nào:

1.  **Chuẩn bị "hàng":** Sếp build một phiên bản firmware mới. Hệ thống build của Zephyr sẽ tự động ký tên lên nó và tạo ra file `app_update.bin`.
2.  **Giao hàng:** Ứng dụng của sếp (đang chạy trên device) dùng một phương thức nào đó (HTTP, MQTT, Bluetooth...) để tải file `app_update.bin` về và ghi nó vào `slot1_partition`.
3.  **Kích hoạt:** Người dùng (hoặc hệ thống) ra lệnh cho MCUmgr `image test <hash_cua_image>`. Lệnh này báo cho MCUboot rằng "Lần reboot tới hãy thử chạy image mới này nhé".
4.  **Reboot:** Thiết bị khởi động lại.
5.  **MCUboot hành động:**
    *   MCUboot thức dậy đầu tiên.
    *   Nó thấy có lệnh "test".
    *   Nó kiểm tra chữ ký của image trong `slot-1`.
    *   Nếu chữ ký hợp lệ, nó thực hiện hoán đổi. Image mới ở `slot-1` sẽ được đánh dấu là image chính để boot.
6.  **Boot image mới:** MCUboot khởi động vào image mới.
7.  **Tự khẳng định:** Image mới chạy, hàm `app_run()` được gọi, và `confirm_image()` được thực thi. Nó gọi `boot_write_img_confirmed()` để báo cho MCUboot: "Tôi đã chạy thành công, hãy biến tôi thành vĩnh viễn".
8.  **Hoàn tất:** Phi vụ update thành công. Từ giờ, thiết bị sẽ luôn khởi động vào image mới này, cho đến khi có một cuộc update khác.

---

## Phần 4: Luật chơi của giới giang hồ OTA

Làm OTA không phải là chuyện "thích gì làm nấy". Để đảm bảo an toàn và có khả năng mở rộng, chúng ta nên tuân theo các tiêu chuẩn đã được cộng đồng công nhận.

### 1. The Update Framework (TUF)

Hãy coi TUF như là một bộ "luật an ninh" tối cao cho hệ thống update. Nó không phải là một giao thức, mà là một **framework** thiết kế để chống lại gần như mọi loại tấn công vào chuỗi cung ứng phần mềm.

*   **Tâm điểm:** Phân chia trách nhiệm và có cơ chế hết hạn. TUF định nghĩa các "vai trò" (roles) khác nhau, mỗi vai trò có một chìa khóa riêng và quyền hạn riêng.
    *   **Root role:** Chìa khóa tối cao, thường được giữ offline. Dùng để ký và chứng thực các vai trò khác.
    *   **Targets role:** Ký vào các file firmware thực tế, chứng nhận chúng là hàng "auth".
    *   **Snapshot role:** Cung cấp một cái nhìn nhất quán về tất cả các file có trên server tại một thời điểm, chống lại tấn công replay attack (kẻ gian bắt bạn update lại phiên bản cũ có lỗ hổng).
    *   **Timestamp role:** Một vai trò được ký rất thường xuyên, để chứng minh rằng server vẫn "sống" và không bị kẻ gian chiếm quyền.
*   **Tại sao nó liên quan?** Mặc dù bạn không cần cài đặt đầy đủ TUF, nhưng tư duy của nó là vàng. Khi thiết kế server OTA của mình, hãy tự hỏi: "Làm sao để kẻ tấn công chiếm được server mà không thể lừa client của mình cài đặt malware?". Câu trả lời nằm ở việc phân chia vai trò và chữ ký như TUF.

### 2. OMA LwM2M (Lightweight M2M)

Nếu TUF là bộ luật an ninh, thì LwM2M là một **hệ thống quản lý thiết bị** hoàn chỉnh. Nó là một giao thức được thiết kế cho các thiết bị IoT bị hạn chế về tài nguyên.

*   **Tâm điểm:** Mô hình "Object". LwM2M định nghĩa một tập hợp các đối tượng tiêu chuẩn để quản lý thiết bị. Trong đó, có 3 đối tượng quan trọng cho OTA:
    *   **Object 3 (Device):** Cung cấp thông tin về thiết bị (model, serial number...).
    *   **Object 5 (Firmware Update):** Đây là trung tâm của FOTA. Nó định nghĩa các "resource" như:
        *   `Package`: Nơi để server ghi URL của firmware mới.
        *   `State`: Trạng thái của quá trình update (Idle, Downloading, Downloaded, Updating...).
        *   `Update Result`: Kết quả của lần update cuối cùng.
        *   `Update`: Một resource mà server có thể "execute" để ra lệnh cho client bắt đầu quá trình update.
*   **Khi nào dùng?** Khi sếp cần quản lý một đội quân hàng ngàn, hàng vạn thiết bị. LwM2M không chỉ giúp update firmware, mà còn giám sát tình trạng, thay đổi cấu hình, đọc dữ liệu cảm biến... Nó là một giải pháp toàn diện. Zephyr cũng có sẵn một LwM2M client.

### Lời khuyên của "chuyên gia"

*   **Cho dự án nhỏ, tự phát triển:** Hãy xây dựng một server HTTP/MQTT đơn giản. **Nhưng**, hãy học hỏi tư duy của **TUF**. Ít nhất, hãy đảm bảo firmware của bạn được ký, và có một cơ chế nào đó (ví dụ: một file manifest được ký riêng) để client xác thực phiên bản mới nhất, tránh replay attack.
*   **Cho sản phẩm thương mại, quy mô lớn:** Nghiêm túc cân nhắc sử dụng **LwM2M**. Nó giải quyết rất nhiều vấn đề đau đầu về quản lý vòng đời thiết bị mà sếp sẽ gặp phải.

---

## Lời kết

Vậy là chúng ta đã cùng nhau đi qua một hành trình khá dài, từ việc hiểu MCUboot là gì, cấu hình nó trong Zephyr, cho đến việc nhìn ra bức tranh lớn hơn với các tiêu chuẩn công nghiệp.

Hy vọng sau bài này, sếp và các huynh đệ không còn "run tay" mỗi khi nghĩ đến việc cập nhật firmware cho sản phẩm của mình nữa. Với MCUboot làm vệ sĩ và Zephyr làm nền tảng, chúng ta có thể tự tin "ship" sản phẩm và update chúng một cách an toàn, chuyên nghiệp.

Giờ thì, hãy đi và cập nhật thiết bị mà không còn nỗi sợ hãi nào!
