---
layout: post
title: "Biên Niên Sử Tái Cấu Trúc HTTP Client: Từ Blocking đến Bất Đồng Bộ"
---

# Biên Niên Sử Tái Cấu Trúc HTTP Client: Từ Blocking đến Bất Đồng Bộ

Tài liệu này ghi lại chi tiết cuộc hội thoại và quá trình hướng dẫn tái cấu trúc một HTTP client đơn giản trong Zephyr RTOS, chuyển đổi từ một kiến trúc blocking tuần tự sang một kiến trúc non-blocking, bất đồng bộ dựa trên máy trạng thái.

---

# Biên Niên Sử Tái Cấu Trúc HTTP Client: Từ Blocking đến Bất Đồng Bộ

Tài liệu này ghi lại chi tiết cuộc hội thoại và quá trình hướng dẫn tái cấu trúc một HTTP client đơn giản trong Zephyr RTOS, chuyển đổi từ một kiến trúc blocking tuần tự sang một kiến trúc non-blocking, bất đồng bộ dựa trên máy trạng thái.

## Chương 1: Phán Xét "Bản Án" cho `http_get.c`

Mọi chuyện bắt đầu khi sếp yêu cầu tôi xem xét file `http_get.c` và đánh giá hiệu quả của nó.

> **Sếp:** ok, let look at the http_get.c; explain this and show me if this is efficient

#### `src/http_get.c`
```c
/* 
 * Copyright (c) 2023 Craig Peacock.
 * 
 * API Information:
 * https://docs.zephyrproject.org/3.2.0/connectivity/networking/api/http.html
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <stdio.h>
#include <stdlib.h>
#include <zephyr/net/socket.h>
#include <zephyr/kernel.h>
#include <zephyr/net/http/client.h>

void nslookup(const char * hostname, struct zsock_addrinfo **results)
{
	int err;
	
	struct zsock_addrinfo hints = {
		//.ai_family = AF_INET,     // Allow IPv4 Address
		.ai_family = AF_UNSPEC,		// Allow IPv4 or IPv6	
		.ai_socktype = SOCK_STREAM,
	};

	err = zsock_getaddrinfo(hostname, NULL, &hints, (struct zsock_addrinfo **) results);
	if (err) {
		printf("getaddrinfo() failed, err %d\n", errno);
		return;
	}
}

void print_addrinfo_results(struct zsock_addrinfo **results)
{
	char ipv4[INET_ADDRSTRLEN];
	char ipv6[INET6_ADDRSTRLEN];
	struct sockaddr_in *sa;
	struct sockaddr_in6 *sa6;
	struct zsock_addrinfo *rp;
	
	for (rp = *results; rp != NULL; rp = rp->ai_next) {
		if (rp->ai_addr->sa_family == AF_INET) {
			// IPv4 Address
			sa = (struct sockaddr_in *) rp->ai_addr;
			zsock_inet_ntop(AF_INET, &sa->sin_addr, ipv4, INET_ADDRSTRLEN);
			printf("IPv4: %s\n", ipv4);
		}
		if (rp->ai_addr->sa_family == AF_INET6) {
			// IPv6 Address
			sa6 = (struct sockaddr_in6 *) rp->ai_addr;
			zsock_inet_ntop(AF_INET6, &sa6->sin6_addr, ipv6, INET6_ADDRSTRLEN);
			printf("IPv6: %s\n", ipv6);
		}
	}
}

int connect_socket(struct zsock_addrinfo **results, uint16_t port)
{
	int sock;
	int ret;
	struct zsock_addrinfo *rp;
	struct sockaddr_in6 *sa6;

	// Create IPv6 Socket
	sock = zsock_socket(AF_INET6, SOCK_STREAM, IPPROTO_TCP);
	if (sock < 0) {
		printk("Error creating IPv6 socket\n");
		return(-1);
	}

	// Iterate through IPv6 addresses until we get a successful connection
	for (rp = *results; rp != NULL; rp = rp->ai_next) {
		if (rp->ai_addr->sa_family == AF_INET6) {
			// IPv6 Address
			sa6 = (struct sockaddr_in6 *) rp->ai_addr;
			sa6->sin6_port = htons(port);
			
			char ipv6[INET6_ADDRSTRLEN];
			zsock_inet_ntop(AF_INET6, &sa6->sin6_addr, ipv6, INET6_ADDRSTRLEN);
			printk("Connecting to %s:%d ", ipv6, port);

			ret = zsock_connect(sock, (struct sockaddr *) sa6, sizeof(struct sockaddr_in));
			if (ret == 0) {
				printk("Success\r\n");
				return(sock);
			} else {
				printk("Failure (%d)\r\n", ret);
			}
		}
	}
	// Close IPv6 Socket
	zsock_close(sock);

	// Now try IPv4 Addresses
	struct sockaddr_in *sa;

	// Create IPv4 Socket
	sock = zsock_socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
	if (sock < 0) {
		printk("Error creating IPv4 socket\n");
		return(-1);
	}
	
	// Iterate through IPv4 addresses until we get a successful connection
	for (rp = *results; rp != NULL; rp = rp->ai_next) {
		if (rp->ai_addr->sa_family == AF_INET) {
			// IPv4 Address
			sa = (struct sockaddr_in *) rp->ai_addr;
			sa->sin_port = htons(port);

			char ipv4[INET_ADDRSTRLEN];
			zsock_inet_ntop(AF_INET, &sa->sin_addr, ipv4, INET_ADDRSTRLEN);
			printk("Connecting to %s:%d ", ipv4, port);

			ret = zsock_connect(sock, (struct sockaddr *) sa, sizeof(struct sockaddr_in));
			if (ret == 0) {
				printk("Success\r\n");
				return(sock);
			} else {
				printk("Failure (%d)\r\n", ret);
			}
		}
	}
	
	return(0);
}

static void http_response_cb(struct http_response *rsp,
			enum http_final_call final_data,
			void *user_data)
{
	if (final_data == HTTP_DATA_MORE) {
		//printk("Partial data received (%zd bytes)\n", rsp->data_len);
	} else if (final_data == HTTP_DATA_FINAL) {
		//printk("All the data received (%zd bytes)\n", rsp->data_len);
	}

	//printk("Bytes Recv %zd\n", rsp->data_len);
	//printk("Response status %s\n", rsp->http_status);
	//printk("Recv Buffer Length %zd\n", rsp->recv_buf_len);
	printk("%.*s", rsp->data_len, rsp->recv_buf);
}

void http_get(int sock, char * hostname, char * url)
{
	struct http_request req = { 0 };
	static uint8_t recv_buf[512];
	int ret;

	req.method = HTTP_GET;
	req.url = url;
	req.host = hostname;
	req.protocol = "HTTP/1.1";
	req.response = (void *)http_response_cb;
	req.recv_buf = recv_buf;
	req.recv_buf_len = sizeof(recv_buf);

	ret = http_client_req(sock, &req, 5000, NULL);
}
```

Sau khi xem xét, tôi đã trình bày một phân tích ban đầu.

### 1.1. Phân Tích Sơ Bộ (Cấp độ "Gạch" và "Tường")

Đây là bản phân tích ban đầu của tôi về cách hoạt động của `http_get.c` và những đánh giá đầu tiên.

#### Cấp độ "Viên gạch" (Brick): Giải thích từng hàm

1.  **`nslookup()`**:
    *   **Nhiệm vụ:** Hàm này y hệt như lệnh `nslookup` trên máy tính của sếp. Sếp đưa nó một cái hostname (ví dụ: `google.com`), nó sẽ hỏi DNS server xem "thằng cha" này có địa chỉ IP là gì.
    *   **Cách làm:** Nó dùng hàm `zsock_getaddrinfo()` của Zephyr. Cái `hints` (gợi ý) được set là `AF_UNSPEC` nghĩa là "Tìm IP nào cũng được, IPv4 hay IPv6 đều chơi tuốt!". Kết quả trả về là một danh sách các địa chỉ IP khả dụng.

2.  **`connect_socket()`**:
    *   **Nhiệm vụ:** Sau khi có danh sách IP, hàm này sẽ cố gắng kết nối đến một trong các địa chỉ đó trên một port cụ thể (ví dụ port 80 cho HTTP).
    *   **Cách làm:** Ông dev này cũng thật là "có não" khi viết hàm này. Ổng ưu tiên IPv6 trước, đúng chuẩn "công dân gương mẫu của Internet thế hệ mới".
        1.  Tạo một cái socket IPv6.
        2.  Lần lượt thử kết nối tới từng địa chỉ IPv6 trong danh sách. Nếu một cái thành công, nó la lên "Success" và trả về cái socket đang kết nối ngon lành.
        3.  Nếu thử hết IPv6 mà vẫn "toang", nó sẽ đóng cái socket IPv6 đó lại và cay đắng chuyển sang IPv4.
        4.  Tạo một socket IPv4 mới.
        5.  Lặp lại quy trình, thử kết nối với từng địa chỉ IPv4.
        6.  Nếu vẫn thất bại, nó sẽ trả về 0, báo hiệu một pha kết nối thất bại toàn tập.

3.  **`http_get()` và `http_response_cb()`**:
    *   **Nhiệm vụ:** Cặp đôi hoàn cảnh này dùng để gửi yêu cầu HTTP GET và xử lý dữ liệu trả về.
    *   **Cách `http_get()` làm:** Nó chuẩn bị một cái "đơn đặt hàng" (`struct http_request`) bao gồm: méthode là GET, URL, hostname, và quan trọng nhất là một cái `response callback` (chính là hàm `http_response_cb`). Nó cũng khai báo một buffer nhận dữ liệu `recv_buf` kích thước 512 bytes. Cuối cùng, nó gọi `http_client_req()` để gửi yêu cầu đi.
    *   **Cách `http_response_cb()` làm:** Đây là hàm sẽ được gọi khi có dữ liệu từ server trả về.
        *   Server trả dữ liệu về từng mẩu nhỏ, mỗi mẩu không quá 512 bytes (kích thước của `recv_buf`).
        *   Với mỗi mẩu dữ liệu nhận được, hàm callback này sẽ được gọi, và nó chỉ đơn giản là `printk` (in) cái mẩu dữ liệu đó ra console. Nó cứ in từng mảnh, từng mảnh cho đến khi nhận được mẩu cuối cùng (`HTTP_DATA_FINAL`).

#### Cấp độ "Bức Tường" (Wall): Đánh giá hiệu quả

*   **Ưu điểm (Mặt được):**
    1.  **Code rõ ràng, dễ hiểu:** Logic đi thẳng một đường: tìm IP -> kết nối -> gửi request -> nhận response. Với một ví dụ đơn giản thì quá tuyệt vời.
    2.  **Ưu tiên IPv6:** Việc thử IPv6 trước là một hành động đáng khen.
    3.  **Tiết kiệm RAM:** Việc dùng một buffer nhận 512 bytes cố định là một lựa chọn thông minh cho các thiết bị nhúng (embedded) tài nguyên hạn hẹp.

*   **Nhược điểm (Mặt chưa được - Chỗ để châm biếm đây):**
    1.  **Logic kết nối "ngây thơ":** Cơ chế fallback từ IPv6 sang IPv4 quá đơn giản. Nó sẽ tuần tự thử HẾT tất cả các địa chỉ IPv6, đợi từng cái timeout một, rồi mới chuyển sang IPv4. Nếu mạng bị cấu hình lỗi, ứng dụng sẽ "đứng hình" rất lâu.
    2.  **Xử lý response "lúa":** Hàm callback chỉ `printk` từng mẩu dữ liệu ra. Giả sử sếp cần xử lý một file JSON, sếp sẽ phải tự mình viết code để ráp các mảnh đó lại.
    3.  **Blocking toàn tập:** Toàn bộ quá trình từ `nslookup` đến khi `http_get` xong xuôi đều là các hàm blocking. Nghĩa là cái thread `wifi_thread` sẽ bị treo cứng trong suốt quá trình này. Đối với một ví dụ thì không sao, nhưng trong một ứng dụng thực tế, đây là một thảm họa.

### 1.2. Nâng Tầm "Vũ Trụ": So Sánh Với Tiêu Chuẩn Công Nghiệp

Khi sếp hỏi về "tiêu chuẩn công nghiệp", cuộc thảo luận của chúng ta đã thực sự được nâng tầm.

> **Sếp:** what about industry standard?

Đây là bản tóm tắt chi tiết về những gì làm nên một thư viện "chuẩn công nghiệp" và tại sao code cũ của chúng ta lại chưa đạt được.

#### Triết lý nền tảng

"Tiêu chuẩn công nghiệp" không chỉ là về code chạy được, mà nó là cả một triết lý về sự **BỀN BỈ (Robustness)**, **AN TOÀN (Security)** và **KHẢ NĂNG MỞ RỘNG (Scalability)**.

**1. Kiến trúc bất đồng bộ (Asynchronous) - Triết lý nền tảng**

Trong thế giới embedded, **"blocking" (chờ đợi)** là một từ cấm kỵ. Một hệ thống công nghiệp phải chạy theo **kiến trúc hướng sự kiện (Event-Driven)**, thường được hiện thực hóa bằng **máy trạng thái (State Machine)**.

*   **Tư duy tuần tự (của `http_get.c`):**
    1.  Làm việc A -> **CHỜ**...
    2.  Làm việc B -> **CHỜ**...
    3.  Xong.
    *Hệ quả: Cả hệ thống "đứng hình" trong lúc chờ đợi, không thể làm việc khác.*

*   **Tư duy bất đồng bộ (của "dân chuyên"):**
    1.  Yêu cầu làm việc A. Hàm trả về ngay lập tức.
    2.  CPU rảnh rỗi, đi làm việc C, D, E...
    3.  Khi A làm xong, một "sự kiện" được bắn ra, báo hiệu A đã hoàn thành.
    4.  Hệ thống xử lý sự kiện này và yêu cầu làm việc B. Hàm lại trả về ngay.
    5.  CPU lại rảnh...
    *Hệ quả: Hệ thống không bao giờ "đứng hình". Nó luôn phản hồi, luôn "sống", dù cho mạng có chậm như rùa.*

**2. Chiến lược kết nối thông minh (Robust "Happy Eyeballs")**

Code ví dụ thử IPv6 rồi fallback sang IPv4 một cách rất ngây thơ. Tiêu chuẩn công nghiệp dùng một thuật toán tên là **"Happy Eyeballs"** (RFC 8305), một cuộc đua song song đúng nghĩa để giảm độ trễ cho người dùng.
1.  Bắn yêu cầu kết nối IPv6.
2.  Bấm một đồng hồ đếm ngược cực ngắn (ví dụ: 250 mili giây).
3.  Nếu trong 250ms đó mà IPv6 chưa kết nối được, **ngay lập tức** bắn thêm yêu cầu kết nối IPv4 song song.
4.  Thằng nào về đích (kết nối thành công) trước thì thắng. Thằng còn lại sẽ bị hủy.

**3. Xử lý dữ liệu thông minh (Intelligent Data Handling)**

Code ví dụ chỉ `printk` từng mẩu dữ liệu. Một thư viện chuyên nghiệp sẽ cung cấp nhiều cơ chế xử lý response body:
*   **Ghép nối vào buffer động (Dynamic Buffer):** Tự động cấp phát bộ nhớ để ráp các mảnh JSON lại. Trong Zephyr, người ta thường dùng **Memory Pools (`k_mem_slab`)** để tránh rủi ro của `malloc`.
*   **Xử lý Streaming (Streaming Parser):** Đối với các file siêu lớn (như file firmware OTA), dữ liệu sẽ được đẩy thẳng vào một "cỗ máy xay" khác khi nó vừa đến mà không cần lưu vào RAM.

**4. Bảo mật là mặc định (Security by Default - HTTPS)**

Đây là điểm chí mạng. `http_get.c` dùng HTTP. **Tiêu chuẩn công nghiệp hét lên:** "Dùng HTTP không khác gì đứng ngoài đường la to bí mật của công ty cho cả làng cùng nghe."

**HTTPS (HTTP qua TLS)** là yêu cầu bắt buộc. Điều này có nghĩa là phải tích hợp một thư viện TLS như **Mbed TLS** (có sẵn trong Zephyr), quản lý chứng chỉ, và chấp nhận việc tiêu tốn thêm rất nhiều RAM và CPU.

Chính từ những phân tích này, sếp đã quyết định:

> **Sếp:** so, let fix it, step by step; but teach me, not do it for me

Và cuộc hành trình tái cấu trúc của chúng ta bắt đầu.

## Chương 2: Xây Dựng "Ngôi Nhà" Mới - `http_client`

### Bước 1: Đặt Nền Móng - Định nghĩa Máy Trạng Thái

**Bài học:** Bước đầu tiên của bất kỳ công trình nào là bản vẽ thiết kế. Trong trường hợp của chúng ta, bản vẽ chính là việc định nghĩa các **trạng thái** và một cấu trúc dữ liệu để lưu trữ **ngữ cảnh (context)** của client.

**Lý do:**
*   **`enum http_client_state`**: Việc định nghĩa các trạng thái một cách rõ ràng giúp chúng ta hình dung được toàn bộ luồng hoạt động của client. Mỗi một bước trong quá trình (tìm DNS, kết nối, gửi, nhận) sẽ là một trạng thái.
*   **`struct http_client_context`**: Đây là "bộ não" của client. Thay vì truyền hàng tá tham số giữa các hàm, chúng ta gói gọn mọi thứ vào một `struct`. Trạng thái hiện tại, socket, hostname, URL... tất cả đều nằm ở đây. Điều này giúp code sạch sẽ và dễ quản lý. Việc truyền con trỏ tới `struct` này vào các hàm callback là kỹ thuật cơ bản để duy trì trạng thái trong lập trình bất đồng bộ.

**Hành động:** Chúng ta đã thống nhất tạo ra hai file mới: `src/http_client.h` và `src/http_client.c`.

**Code đầy đủ tại Bước 1:**

#### `src/http_client.h`
```c
#ifndef HTTP_CLIENT_H_
#define HTTP_CLIENT_H_

#include <zephyr/net/socket.h>

/* 
 * Giải thích chi tiết từng dòng:
 * #ifndef HTTP_CLIENT_H_ và #define HTTP_CLIENT_H_: Đây được gọi là "Include Guard".
 * Nó ngăn chặn việc file header này bị include nhiều lần trong một file .c, 
 * một lỗi có thể gây ra việc định nghĩa lại các kiểu dữ liệu và báo lỗi biên dịch.
 * Quy ước đặt tên thường là <TÊN_FILE>_H_.
 */

/* 
 * Enum định nghĩa các trạng thái (states) của HTTP client.
 * Đây là trái tim của State Machine. Mỗi hằng số đại diện cho một bước
 * trong quy trình làm việc của client.
 */
enum http_client_state {
    HTTP_CLIENT_STATE_IDLE,          // Trạng thái nghỉ, chưa làm gì.
    HTTP_CLIENT_STATE_DNS_START,     // Bắt đầu quá trình phân giải DNS.
    HTTP_CLIENT_STATE_DNS_PENDING,   // Đang chờ kết quả DNS từ callback.
    HTTP_CLIENT_STATE_CONNECT_START, // Bắt đầu quá trình kết nối socket.
    HTTP_CLIENT_STATE_CONNECTING,    // Đang chờ kết nối socket thành công (dùng poll).
    HTTP_CLIENT_STATE_REQUEST_SEND,  // Gửi gói tin HTTP request.
    HTTP_CLIENT_STATE_RESPONSE_RECV, // Đang trong quá trình nhận response.
    HTTP_CLIENT_STATE_DONE,          // Hoàn thành, thành công.
    HTTP_CLIENT_STATE_ERROR,         // Trạng thái lỗi, dừng hoạt động.
};

/* 
 * Struct chứa toàn bộ "bộ não" và tài nguyên của một HTTP client instance.
 * Nó gói gọn tất cả dữ liệu cần thiết để duy trì trạng thái qua các lần
 * gọi hàm `http_client_run()` và trong các hàm callback.
 */
struct http_client_context {
    // Trạng thái hiện tại của state machine.
    enum http_client_state state;

    // File descriptor cho socket. -1 nghĩa là chưa được tạo.
    int sock;

    // Con trỏ tới danh sách kết quả địa chỉ IP từ DNS.
    // Chúng ta sẽ sớm thấy việc dùng con trỏ ở đây có rủi ro.
    struct zsock_addrinfo *addr_res;

    // Con trỏ tới địa chỉ IP hiện tại đang được thử kết nối.
    struct zsock_addrinfo *current_addr;

    // Các thông tin do người dùng cung cấp.
    char *hostname;
    char *url;
    uint16_t port;
};

/*
 * Khai báo các hàm công khai (public functions) mà các file .c khác
 * (như main.c) có thể gọi.
 */

/**
 * @brief Khởi tạo context cho HTTP client.
 */
void http_client_init(struct http_client_context *ctx, char *hostname, char *url, uint16_t port);

/**
 * @brief "Nhịp tim" của HTTP client, phải được gọi liên tục.
 */
int http_client_run(struct http_client_context *ctx);


#endif /* HTTP_CLIENT_H_ */
```

#### `src/http_client.c`
```c
#include "http_client.h"
#include <zephyr/logging/log.h>

// Đăng ký một module log riêng cho file này, giúp việc debug dễ dàng hơn.
// Chúng ta có thể bật/tắt log của riêng module này trong prj.conf.
LOG_MODULE_REGISTER(http_client, LOG_LEVEL_INF);

void http_client_init(struct http_client_context *ctx, char *hostname, char *url, uint16_t port)
{
    // TODO: Sẽ được hiện thực ở bước sau.
}

int http_client_run(struct http_client_context *ctx)
{
    // TODO: Sẽ được hiện thực ở bước sau.
    return 0;
}
```

### Bước 2: Tạo "Nhịp Tim" cho Máy Trạng Thái

**Bài học:** Bây giờ chúng ta xây dựng hàm `http_client_run()`. Nó sẽ là một cấu trúc `switch-case` đơn giản, hoạt động như "trái tim" của client. Mỗi khi được gọi, nó sẽ kiểm tra trạng thái hiện tại và thực hiện một hành động nhỏ tương ứng.

**Lý do:**
*   **Cấu trúc `switch-case`**: Đây là cách triển khai một máy trạng thái kinh điển và dễ đọc nhất. Mỗi `case` là một state.
*   **Hàm `set_state()`**: Tạo một hàm phụ trợ (helper function) để thay đổi trạng thái. Lý do là để tập trung logic chuyển đổi trạng thái vào một chỗ. Việc thêm một câu lệnh `LOG_INF` vào đây giúp chúng ta theo dõi luồng chạy của máy trạng thái một cách cực kỳ trực quan.
*   **Hàm `http_client_init()`**: Hàm này rất quan trọng. Nó đảm bảo rằng mỗi khi chúng ta bắt đầu một "phiên làm việc" mới với client, mọi thứ đều được reset về trạng thái ban đầu sạch sẽ. Dùng `memset` để xóa sạch `struct` là một thói quen tốt để tránh các giá trị rác.

**Hành động:** Cập nhật file `src/http_client.c` với khung sườn của máy trạng thái.

**Code đầy đủ tại Bước 2:**

`src/http_client.h` không thay đổi so với Bước 1.

#### `src/http_client.c`
```c
#include "http_client.h"
#include <zephyr/logging/log.h>
#include <string.h> // Thêm thư viện này để dùng hàm memset

LOG_MODULE_REGISTER(http_client, LOG_LEVEL_INF);

void http_client_init(struct http_client_context *ctx, char *hostname, char *url, uint16_t port)
{
    // Dùng memset để ghi toàn bộ byte trong struct ctx về 0.
    // Đây là cách nhanh và an toàn để reset một struct.
    memset(ctx, 0, sizeof(struct http_client_context));

    // Thiết lập các giá trị ban đầu cho phiên làm việc.
    ctx->state = HTTP_CLIENT_STATE_IDLE;
    ctx->hostname = hostname;
    ctx->url = url;
    ctx->port = port;
    ctx->sock = -1; // -1 là một giá trị quy ước cho "socket không hợp lệ".
}

// Hàm helper để đổi state, giúp debug dễ hơn
static void set_state(struct http_client_context *ctx, enum http_client_state new_state)
{
    // In ra sự thay đổi trạng thái giúp chúng ta biết chính xác chương trình đang làm gì.
    LOG_INF("State transition: %d -> %d", ctx->state, new_state);
    ctx->state = new_state;
}

int http_client_run(struct http_client_context *ctx)
{
    // Cấu trúc switch-case, trái tim của state machine.
    switch (ctx->state) {
    case HTTP_CLIENT_STATE_IDLE:
        // Khi ở IDLE, chúng ta tự động bắt đầu quá trình.
        LOG_INF("Client is IDLE, starting process.");
        set_state(ctx, HTTP_CLIENT_STATE_DNS_START);
        break;

    case HTTP_CLIENT_STATE_DNS_START:
        LOG_INF("Handling state: DNS_START");
        // TODO: Gọi hàm DNS lookup bất đồng bộ ở đây
        break;

    case HTTP_CLIENT_STATE_DNS_PENDING:
        LOG_INF("Handling state: DNS_PENDING - Doing nothing, waiting for callback.");
        // State này không làm gì trong hàm run(), nó chỉ chờ đợi một sự kiện từ bên ngoài (callback).
        break;

    case HTTP_CLIENT_STATE_CONNECT_START:
        LOG_INF("Handling state: CONNECT_START");
        // TODO: Bắt đầu quá trình kết nối non-blocking
        break;

    case HTTP_CLIENT_STATE_CONNECTING:
        LOG_INF("Handling state: CONNECTING");
        // TODO: Dùng poll() để kiểm tra kết nối
        break;

    case HTTP_CLIENT_STATE_REQUEST_SEND:
        LOG_INF("Handling state: REQUEST_SEND");
        // TODO: Gửi HTTP request
        break;

    case HTTP_CLIENT_STATE_RESPONSE_RECV:
        LOG_INF("Handling state: RESPONSE_RECV");
        // TODO: Nhận và xử lý response
        break;

    case HTTP_CLIENT_STATE_DONE:
        LOG_INF("Process finished successfully.");
        return 1; // Trả về > 0 để báo hiệu cho nơi gọi (main) là đã xong.

    case HTTP_CLIENT_STATE_ERROR:
        LOG_ERR("Process stopped due to an error.");
        return -1; // Trả về < 0 để báo hiệu lỗi.
    
default:
        LOG_ERR("Unknown state: %d", ctx->state);
        set_state(ctx, HTTP_CLIENT_STATE_ERROR);
        break;
    }

    return 0; // Trả về 0 để báo hiệu "vẫn đang chạy, chưa xong".
}
```

### Bước 2.5: Tích Hợp vào `main.c`

**Bài học:** Máy trạng thái sẽ không tự chạy. Chúng ta cần một vòng lặp ở đâu đó để liên tục gọi hàm `http_client_run()`, tức là liên tục "bơm máu" cho "trái tim".

**Lý do:**
Vòng lặp `while(1)` trong `wifi_thread` chính là nơi thực hiện việc này. Nó thay thế hoàn toàn các lời gọi blocking cũ. Việc thêm một `k_sleep()` nhỏ là rất quan trọng để thread của chúng ta không chiếm 100% CPU. Nó "nhường" CPU cho các thread khác và cho hệ điều hành làm việc.

**Hành động:** Thay thế logic trong `main.c` để gọi "nhịp tim" của chúng ta.

**Code đầy đủ tại Bước 2.5:**

#### `src/main.c`
```c
#include <zephyr/kernel.h>
#include <zephyr/logging/log.h>
#include "wifi.h"
#include "http_client.h" // <-- Include file mới của chúng ta

LOG_MODULE_REGISTER(main_module, LOG_LEVEL_INF);

#define WIFI_STACK_SIZE 4096

void wifi_thread_entry(void *a, void *b, void *c) {
	LOG_INF("Wifi thread is created");
	// wifi_init() là một hàm blocking, nó sẽ chờ đến khi có IP.
	// Sau khi nó chạy xong, chúng ta mới có mạng để bắt đầu state machine.
	wifi_init();

	// --- LOGIC MỚI VỚI STATE MACHINE ---
	struct http_client_context client_ctx;
	
	// Khởi tạo client của chúng ta với các thông tin cần thiết.
	http_client_init(&client_ctx, "iot.beyondlogic.org", "/LoremIpsum.txt", 80);

	// Vòng lặp "nhịp tim".
	while (1) {
		// Gọi hàm run() để thực thi một bước của state machine.
		int ret = http_client_run(&client_ctx);
		if (ret < 0) {
			LOG_ERR("HTTP client run failed");
			break; // Thoát vòng lặp nếu có lỗi.
		} else if (ret > 0) {
			LOG_INF("HTTP client run finished");
			break; // Thoát vòng lặp nếu hoàn thành.
		}
		// Cho thread nghỉ 100ms. Con số này có thể điều chỉnh.
		// Nó cân bằng giữa độ phản hồi của client và việc tiết kiệm CPU.
		k_sleep(K_MSEC(100));
	}
}

K_THREAD_DEFINE(wifi_thread, 
		WIFI_STACK_SIZE,
		wifi_thread_entry,
		NULL, NULL, NULL,
		7,0,0
		);

int main(void) {
	LOG_INF("Main is created");
	return 0;
}
```

### Bước 3: DNS Bất Đồng Bộ - Mồi Câu Đầu Tiên

**Bài học:** Đây là bước nhảy vọt thực sự đầu tiên vào thế giới bất đồng bộ. Thay vì gọi `zsock_getaddrinfo()` (blocking), chúng ta dùng `dns_get_addr_info()` (non-blocking).

**Lý do:**
*   **Hàm `dns_get_addr_info()`:** Nó nhận các tham số tương tự, nhưng quan trọng nhất là 2 tham số cuối: một con trỏ hàm `dns_resolve_cb` và một con trỏ `user_data`. Nó sẽ đăng ký yêu cầu này với DNS client của hệ thống rồi **trả về ngay lập tức**.
*   **Hàm `dns_resolve_cb` (Callback):** Đây là hàm mà chúng ta "hứa" với hệ thống rằng "khi nào có kết quả thì gọi vào đây nhé". Khi DNS client nhận được phản hồi từ server, nó sẽ thực thi hàm callback này.
*   **`user_data`**: Đây là "sợi dây" liên kết. Chúng ta truyền con trỏ tới `client_ctx` của chúng ta vào đây. Khi callback được gọi, hệ thống sẽ trả lại con trỏ này cho chúng ta. Bằng cách đó, bên trong callback, chúng ta biết chính xác client nào vừa nhận được kết quả DNS, và có thể truy cập, thay đổi "bộ não" của nó (ví dụ: `set_state(...)`).
*   **State `DNS_PENDING`**: Sau khi gọi `dns_get_addr_info()`, chúng ta phải ngay lập tức chuyển sang trạng thái "chờ". Tại sao? Vì nếu không, ở lần lặp `http_client_run()` tiếp theo, nó sẽ lại rơi vào case `DNS_START` và gửi thêm một yêu cầu DNS nữa, tạo ra một vòng lặp vô tận. State `PENDING` đảm bảo rằng "tôi đã gửi yêu cầu rồi, giờ tôi sẽ không làm gì ở hàm `run()` nữa mà chỉ ngồi chờ callback thôi".

**Hành động:** Hiện thực hóa logic DNS bất đồng bộ trong `http_client.c`.

**Code đầy đủ tại Bước 3:**

#### `src/http_client.c`
```c
#include "http_client.h"
#include <zephyr/logging/log.h>
#include <string.h>
#include <zephyr/net/dns_resolve.h> // <-- Thêm thư viện DNS

LOG_MODULE_REGISTER(http_client, LOG_LEVEL_INF);

// Hàm http_client_init và set_state giữ nguyên như Bước 2

// === HÀM CALLBACK MỚI CHO DNS ===
/*
 * @brief Hàm callback được gọi bởi DNS resolver khi có kết quả.
 *
 * @param status Trạng thái của việc phân giải (thành công, lỗi, timeout...).
 * @param info Con trỏ tới struct chứa thông tin địa chỉ. NULL nếu lỗi.
 * @param user_data Con trỏ mà chúng ta đã truyền vào lúc gọi dns_get_addr_info.
 */
static void dns_resolve_cb(enum dns_resolve_status status,
                         struct dns_addrinfo *info,
                         void *user_data)
{
    // Ép kiểu user_data trở lại thành con trỏ tới context của chúng ta.
    struct http_client_context *ctx = user_data;

    if (status != DNS_E_SUCCESS) {
        LOG_ERR("DNS resolution failed with status: %d", status);
        set_state(ctx, HTTP_CLIENT_STATE_ERROR);
        return; // Dừng lại ngay lập tức.
    }

    if (info == NULL || info->ai_addrlen == 0) {
        LOG_ERR("DNS resolution returned no addresses");
        set_state(ctx, HTTP_CLIENT_STATE_ERROR);
        return;
    }
    
    // !!! CẢNH BÁO "CHEAT" !!!
    // Lưu con trỏ kết quả vào context.
    // Đây là một cách làm TẠM THỜI để đơn giản hóa.
    // Lý do nó nguy hiểm: `info` là con trỏ tới một vùng nhớ được cấp phát bởi
    // DNS resolver. Chúng ta không biết vòng đời của nó. Nó có thể bị giải phóng
    // bất cứ lúc nào sau khi callback này kết thúc.
    // Ở bước sau, chúng ta sẽ sửa lại bằng cách sao chép dữ liệu thay vì lưu con trỏ.
    ctx->addr_res = (struct zsock_addrinfo *)info;
    
    LOG_INF("DNS resolution successful.");
    // Quan trọng nhất: Chuyển state machine sang bước tiếp theo!
    set_state(ctx, HTTP_CLIENT_STATE_CONNECT_START);
}

int http_client_run(struct http_client_context *ctx)
{
    switch (ctx->state) {
    case HTTP_CLIENT_STATE_IDLE:
        LOG_INF("Client is IDLE, starting process.");
        set_state(ctx, HTTP_CLIENT_STATE_DNS_START);
        break;

    case HTTP_CLIENT_STATE_DNS_START:
        LOG_INF("Handling state: DNS_START - Beginning DNS resolution");
        
        int ret = dns_get_addr_info(ctx->hostname,
                                    DNS_QUERY_TYPE_A, // Chỉ tìm địa chỉ IPv4 cho đơn giản
                                    NULL,
                                    dns_resolve_cb, // Con trỏ tới hàm callback
                                    (void *)ctx,    // Truyền context của chúng ta làm user_data
                                    1000);          // Timeout 1 giây
        if (ret < 0) {
            LOG_ERR("Failed to start DNS resolution: %d", ret);
            set_state(ctx, HTTP_CLIENT_STATE_ERROR);
        } else {
            // Chuyển sang trạng thái chờ callback
            set_state(ctx, HTTP_CLIENT_STATE_DNS_PENDING);
        }
        break;

    case HTTP_CLIENT_STATE_DNS_PENDING:
        // Trong trạng thái này, chúng ta không làm gì cả, chỉ chờ.
        // Có thể thêm logic timeout ở đây nếu muốn.
        break;

    case HTTP_CLIENT_STATE_CONNECT_START:
        LOG_INF("Handling state: CONNECT_START");
        break;
        
    case HTTP_CLIENT_STATE_CONNECTING:
        LOG_INF("Handling state: CONNECTING");
        break;
        
    case HTTP_CLIENT_STATE_REQUEST_SEND:
        LOG_INF("Handling state: REQUEST_SEND");
        break;
        
    case HTTP_CLIENT_STATE_RESPONSE_RECV:
        LOG_INF("Handling state: RESPONSE_RECV");
        break;
        
    case HTTP_CLIENT_STATE_DONE:
        LOG_INF("Process finished successfully.");
        return 1;
        
    case HTTP_CLIENT_STATE_ERROR:
        LOG_ERR("Process stopped due to an error.");
        return -1;
        
default:
        LOG_ERR("Unknown state: %d", ctx->state);
        set_state(ctx, HTTP_CLIENT_STATE_ERROR);
        break;
    }

    return 0;
}
```

### Bước 4: Kết Nối Non-Blocking & Dọn Dẹp Bộ Nhớ

**Bài học 1 - Dọn dẹp bộ nhớ DNS:** Như đã cảnh báo, lưu con trỏ `info` là rất nguy hiểm. Cách làm đúng là sao chép (copy) dữ liệu mà chúng ta cần vào chính `context` của mình, sau đó giải phóng ngay vùng nhớ mà DNS resolver đã cấp phát.

**Lý do:**
*   **`struct sockaddr_storage`**: Đây là một kiểu dữ liệu đặc biệt trong C networking. Nó đủ lớn để chứa được cả `sockaddr_in` (cho IPv4) và `sockaddr_in6` (cho IPv6). Dùng nó giúp code của chúng ta an toàn và tương thích với cả 2 loại địa chỉ mà không cần lo về kích thước.
*   **`memcpy()`**: Chúng ta dùng `memcpy` để sao chép toàn bộ nội dung của địa chỉ IP từ `info->ai_addr` vào `ctx->current_addr`. Bây giờ dữ liệu đã nằm an toàn trong "nhà" của chúng ta.
*   **`dns_free_addrinfo(info)`**: Sau khi đã lấy thứ mình cần, chúng ta phải "dọn dẹp". Hàm này báo cho hệ thống rằng "tôi đã dùng xong vùng nhớ này, ông có thể thu hồi nó". Không gọi hàm này sẽ gây ra **rò rỉ bộ nhớ (memory leak)**.

**Bài học 2 - Kết nối Non-Blocking:** Tương tự DNS, chúng ta có thể bắt đầu kết nối socket mà không cần chờ.

**Lý do:**
*   **`zsock_fcntl(ctx->sock, F_SETFL, O_NONBLOCK)`**: Đây là câu lệnh "thần chú". `fcntl` là hàm để điều khiển các thuộc tính của file descriptor. `F_SETFL` nghĩa là "tôi muốn set cờ (flag)". `O_NONBLOCK` là cờ "không blocking". Sau câu lệnh này, mọi thao tác trên socket `ctx->sock` (như `connect`, `send`, `recv`) sẽ trả về ngay lập tức.
*   **`errno == EINPROGRESS`**: Khi gọi `zsock_connect()` trên một non-blocking socket, nó sẽ trả về -1 và set biến `errno` thành `EINPROGRESS`. Đây **không phải là lỗi**, mà là một thông báo rằng "quá trình kết nối đang được tiến hành trong nền". Chúng ta phải kiểm tra điều kiện này để phân biệt nó với các lỗi kết nối thật sự khác.
*   **`zsock_poll()`**: Đây là công cụ để theo dõi trạng thái của nhiều socket cùng lúc mà không cần blocking.
    *   `struct zsock_pollfd`: Chúng ta khai báo một mảng các struct này. Mỗi struct đại diện cho một socket cần theo dõi (`fd`) và các sự kiện chúng ta quan tâm trên socket đó (`events`).
    *   `ZSOCK_POLLOUT`: Sự kiện "có thể ghi". Đối với `connect`, khi một kết nối TCP được thiết lập thành công, socket sẽ chuyển sang trạng thái "writable". Đây chính là dấu hiệu cho thấy chúng ta đã kết nối thành công.
    *   `timeout`: Chúng ta `poll` với một timeout rất ngắn (10ms). Nếu sau 10ms mà chưa có sự kiện gì, `poll` sẽ trả về 0. Hàm `http_client_run` của chúng ta sẽ kết thúc, và ở lần lặp sau, nó sẽ lại vào đây để `poll` tiếp. Đây chính là bản chất của việc "chờ đợi mà không blocking".

**Hành động:** Cập nhật `http_client.h` và `http_client.c` với logic mới.

**Code đầy đủ tại Bước 4:**

#### `src/http_client.h`
```c
#ifndef HTTP_CLIENT_H_
#define HTTP_CLIENT_H_

#include <zephyr/net/socket.h>

// Các enum state giữ nguyên như Bước 1

struct http_client_context {
    enum http_client_state state;
    int sock;

    // Loại bỏ các con trỏ không an toàn
    // struct zsock_addrinfo *addr_res;
    // struct zsock_addrinfo *current_addr;

    // Thay thế bằng một vùng nhớ an toàn trong chính context.
    struct sockaddr_storage current_addr;
    socklen_t current_addr_len;

    char *hostname;
    char *url;
    uint16_t port;
};

// Các khai báo hàm init và run giữ nguyên

#endif /* HTTP_CLIENT_H_ */
```

#### `src/http_client.c`
```c
#include "http_client.h"
#include <zephyr/logging/log.h>
#include <string.h>
#include <zephyr/net/dns_resolve.h>
#include <zephyr/fcntl.h> // <-- Thêm thư viện cho fcntl

LOG_MODULE_REGISTER(http_client, LOG_LEVEL_INF);

// Hàm http_client_init và set_state giữ nguyên như Bước 2

static void dns_resolve_cb(enum dns_resolve_status status,
                         struct dns_addrinfo *info,
                         void *user_data)
{
    struct http_client_context *ctx = user_data;
    
    if (status != DNS_E_SUCCESS || info == NULL) {
        LOG_ERR("DNS resolution failed with status: %d", status);
        set_state(ctx, HTTP_CLIENT_STATE_ERROR);
        // Nếu info không phải NULL, vẫn phải giải phóng nó.
        if (info) {
            dns_free_addrinfo(info);
        }
        return;
    }

    // SAO CHÉP địa chỉ một cách an toàn.
    ctx->current_addr_len = info->ai_addrlen;
    memcpy(&ctx->current_addr, info->ai_addr, info->ai_addrlen);

    // GIẢI PHÓNG bộ nhớ ngay sau khi sao chép.
    dns_free_addrinfo(info);
    
    LOG_INF("DNS resolution successful, address copied.");
    set_state(ctx, HTTP_CLIENT_STATE_CONNECT_START);
}

int http_client_run(struct http_client_context *ctx)
{
    int ret;

    switch (ctx->state) {
    
    // Các case IDLE, DNS_START, DNS_PENDING giữ nguyên như Bước 3

    case HTTP_CLIENT_STATE_CONNECT_START:
        LOG_INF("Handling state: CONNECT_START");

        // Set port cho địa chỉ đã lưu trong context
        if (ctx->current_addr.ss_family == AF_INET) {
            // Ép kiểu sockaddr_storage thành sockaddr_in để set port
            ((struct sockaddr_in *)&ctx->current_addr)->sin_port = htons(ctx->port);
        } else if (ctx->current_addr.ss_family == AF_INET6) {
            // Tương tự cho IPv6
            ((struct sockaddr_in6 *)&ctx->current_addr)->sin6_port = htons(ctx->port);
        } else {
            LOG_ERR("Unknown address family: %d", ctx->current_addr.ss_family);
            set_state(ctx, HTTP_CLIENT_STATE_ERROR);
            break;
        }

        // Tạo socket dựa trên họ địa chỉ đã nhận được từ DNS
        ctx->sock = zsock_socket(ctx->current_addr.ss_family, SOCK_STREAM, IPPROTO_TCP);
        if (ctx->sock < 0) {
            LOG_ERR("Failed to create socket: %d", errno);
            set_state(ctx, HTTP_CLIENT_STATE_ERROR);
            break;
        }

        // Đặt socket ở chế độ non-blocking
        zsock_fcntl(ctx->sock, F_SETFL, O_NONBLOCK);

        // Bắt đầu kết nối
        ret = zsock_connect(ctx->sock, (struct sockaddr *)&ctx->current_addr, ctx->current_addr_len);
        // Kiểm tra lỗi, nhưng bỏ qua lỗi EINPROGRESS
        if (ret < 0 && errno != EINPROGRESS) {
            LOG_ERR("connect() failed: %d", errno);
            set_state(ctx, HTTP_CLIENT_STATE_ERROR);
        } else {
            LOG_INF("Connection initiated...");
            set_state(ctx, HTTP_CLIENT_STATE_CONNECTING);
        }
        break;

    case HTTP_CLIENT_STATE_CONNECTING:
        {
            struct zsock_pollfd fds[1];
            fds[0].fd = ctx->sock;
            fds[0].events = ZSOCK_POLLOUT; // Chúng ta chỉ quan tâm đến sự kiện "có thể ghi"

            // Poll với timeout 10ms
            ret = zsock_poll(fds, 1, 10);

            if (ret < 0) {
                LOG_ERR("poll() failed: %d", errno);
                set_state(ctx, HTTP_CLIENT_STATE_ERROR);
                break;
            }

            if (ret > 0) {
                // Kiểm tra xem có đúng là sự kiện POLLOUT không
                if (fds[0].revents & ZSOCK_POLLOUT) {
                    LOG_INF("Connection successful!");
                    set_state(ctx, HTTP_CLIENT_STATE_REQUEST_SEND);
                }
            }
            // Nếu ret == 0 (timeout), không làm gì cả. Vòng lặp `http_client_run`
            // ở main sẽ gọi lại hàm này để poll tiếp.
        }
        break;
    
    // ... các case còn lại giữ nguyên
    default:
        break;
    }
    return 0;
}
```

### Bước 5: Hái Quả - Gửi Request và Nhận Response

**Bài học:** Chúng ta đã đến bước cuối cùng. Tại đây, chúng ta sẽ tái sử dụng lại hàm `http_client_req` có sẵn của Zephyr.

**Lý do:**
*   **Sự cân bằng:** Mặc dù `http_client_req` là một hàm blocking, nhưng nó đã gói gọn rất nhiều logic phức tạp (tạo HTTP header, gửi request, đọc response header, gọi callback cho body). Việc tự tay làm lại tất cả những thứ này bằng `zsock_send` và `zsock_recv` non-blocking là rất phức tạp và không cần thiết cho mục đích học tập.
*   **Tích hợp khéo léo:** Chúng ta sẽ gọi hàm blocking này trong một state của máy trạng thái non-blocking. Hàm này sẽ chạy cho đến khi nó hoàn thành (nhận đủ body) hoặc đến khi timeout mà chúng ta đưa ra (5000ms). Trong thời gian đó, đúng là `wifi_thread` sẽ bị block, nhưng nó chỉ block ở giai đoạn cuối cùng này. Toàn bộ quá trình DNS và `connect` trước đó đều đã là non-blocking. Đây là một sự đánh đổi chấp nhận được.
*   **Callback lồng callback:** Hàm `http_client_req` cũng sử dụng cơ chế callback. Chúng ta sẽ cung cấp cho nó một hàm `http_response_cb`. Khi `http_client_req` nhận được các mảnh body, nó sẽ gọi hàm callback của chúng ta. Và trong hàm callback đó, khi nhận được mảnh cuối cùng, chúng ta sẽ là người quyết định chuyển state machine sang `DONE`.

**Hành động:** Hoàn thiện nốt các state còn lại.

**Code đầy đủ tại Bước 5:**

#### `src/http_client.h`
```c
#ifndef HTTP_CLIENT_H_
#define HTTP_CLIENT_H_

#include <zephyr/net/socket.h>
#include <zephyr/net/http/client.h> // <-- Thêm thư viện HTTP client

#define RECV_BUF_SIZE 2048

struct http_client_context {
    enum http_client_state state;
    int sock;
    struct sockaddr_storage current_addr;
    socklen_t current_addr_len;
    char *hostname;
    char *url;
    uint16_t port;

    // Struct để chuẩn bị request cho thư viện http-client
    struct http_request req;
    // Buffer để thư viện http-client lưu response body vào
    uint8_t recv_buf[RECV_BUF_SIZE];
};

// Khai báo hàm giữ nguyên

void http_client_init(struct http_client_context *ctx, char *hostname, char *url, uint16_t port);
int http_client_run(struct http_client_context *ctx);


#endif /* HTTP_CLIENT_H_ */
```

#### `src/http_client.c`
```c
#include "http_client.h"
#include <zephyr/logging/log.h>
#include <string.h>
#include <zephyr/net/dns_resolve.h> 
#include <zephyr/fcntl.h> 
#include <zephyr/net/http/client.h> // <-- Thêm thư viện HTTP client

LOG_MODULE_REGISTER(http_client, LOG_LEVEL_INF);

// Hàm init, set_state giữ nguyên như Bước 2

// === HÀM CALLBACK MỚI CHO HTTP RESPONSE ===
static void http_response_cb(struct http_response *rsp,
                           enum http_final_call final_data,
                           void *user_data)
{
    struct http_client_context *ctx = user_data;

    // body_frag_start là con trỏ tới đầu của mảnh body vừa nhận được.
    // body_frag_len là độ dài của nó.
    // Chúng ta chỉ in ra nội dung body, bỏ qua header.
    if (rsp->body_frag_len > 0) {
        printk("%.*s", rsp->body_frag_len, rsp->body_frag_start);
    }
    
    // final_data cho biết đây có phải là lần gọi callback cuối cùng không.
    if (final_data == HTTP_DATA_FINAL) {
        printk("\n--- HTTP transfer complete ---");
        // Đóng socket sau khi xong việc để giải phóng tài nguyên.
        (void)zsock_close(ctx->sock);
        ctx->sock = -1;
        // Chuyển state machine sang trạng thái kết thúc.
        set_state(ctx, HTTP_CLIENT_STATE_DONE);
    }
}


// Hàm dns_resolve_cb giữ nguyên như Bước 4

static void dns_resolve_cb(enum dns_resolve_status status,
                         struct dns_addrinfo *info,
                         void *user_data)
{
    struct http_client_context *ctx = user_data;
    
    if (status != DNS_E_SUCCESS || info == NULL) {
        LOG_ERR("DNS resolution failed with status: %d", status);
        set_state(ctx, HTTP_CLIENT_STATE_ERROR);
        if (info) {
            dns_free_addrinfo(info);
        }
        return;
    }

    ctx->current_addr_len = info->ai_addrlen;
    memcpy(&ctx->current_addr, info->ai_addr, info->ai_addrlen);

    dns_free_addrinfo(info);
    
    LOG_INF("DNS resolution successful, address copied.");
    set_state(ctx, HTTP_CLIENT_STATE_CONNECT_START);
}


int http_client_run(struct http_client_context *ctx)
{
    int ret;

    switch (ctx->state) {
    
    // Các case đến CONNECTING giữ nguyên như Bước 4

    case HTTP_CLIENT_STATE_REQUEST_SEND:
        LOG_INF("Handling state: REQUEST_SEND");

        // Chuẩn bị struct request.
        memset(&ctx->req, 0, sizeof(ctx->req));
        ctx->req.method = HTTP_GET;
        ctx->req.url = ctx->url;
        ctx->req.host = ctx->hostname;
        ctx->req.protocol = "HTTP/1.1";
        ctx->req.response = http_response_cb; // Đăng ký callback xử lý response.
        ctx->req.recv_buf = ctx->recv_buf;     // Cung cấp buffer nhận.
        ctx->req.recv_buf_len = sizeof(ctx->recv_buf);
        ctx->req.user_data = ctx; // Truyền context vào callback.

        // Gọi http_client_req. Hàm này sẽ gửi request và nhận response.
        // Nó sẽ block cho đến khi hoàn thành hoặc timeout (5000ms).
        ret = http_client_req(ctx->sock, &ctx->req, 5000, ctx);
        if (ret < 0) {
            LOG_ERR("http_client_req failed: %d", ret);
            set_state(ctx, HTTP_CLIENT_STATE_ERROR);
        } else {
            // Nếu không lỗi, chúng ta chỉ cần chuyển sang state chờ, 
            // vì callback `http_response_cb` sẽ là người chuyển sang DONE.
            set_state(ctx, HTTP_CLIENT_STATE_RESPONSE_RECV);
        }
        break;

    case HTTP_CLIENT_STATE_RESPONSE_RECV:
        // Trong state này, chúng ta không làm gì cả.
        // Toàn bộ logic nhận và xử lý đã được `http_client_req` và callback của nó đảm nhiệm.
        // Chúng ta chỉ ở đây chờ cho đến khi callback chuyển state sang DONE hoặc ERROR.
        break;

    case HTTP_CLIENT_STATE_DONE:
        LOG_INF("Process finished successfully.");
        return 1; // Báo hiệu hoàn thành

    case HTTP_CLIENT_STATE_ERROR:
        LOG_ERR("Process stopped due to an error.");
        // Đảm bảo socket được đóng nếu có lỗi xảy ra.
        if (ctx->sock >= 0) {
            (void)zsock_close(ctx->sock);
            ctx->sock = -1;
        }
        return -1; // Báo hiệu lỗi
    
default:
        // Các case khác sẽ rơi vào đây.
        break;
    }

    return 0; // Báo hiệu vẫn đang chạy
}
```

## Chương 3: Dọn Dẹp "Tàn Tích"

Cuối cùng, sau khi đã xây xong "biệt thự" `http_client` mới, "căn nhà cấp 4" `http_get.c` và `http_get.h` đã trở nên vô dụng và đã bị xóa đi để giữ cho dự án gọn gàng.

## Tổng kết

Chúng ta đã cùng nhau đi một chặng đường dài, từ việc phân tích một đoạn code blocking đơn giản, hiểu rõ các nhược điểm chí mạng của nó trong môi trường nhúng, cho đến việc tự tay thiết kế và hiện thực một HTTP client mới dựa trên kiến trúc máy trạng thái bất đồng bộ.

Sản phẩm cuối cùng, `http_client`, tuy vẫn còn nhiều không gian để cải tiến (ví dụ: hỗ trợ "Happy Eyeballs" thực sự, thêm HTTPS), nhưng nó đã là một nền tảng vững chắc, tuân thủ các nguyên tắc thiết kế hiện đại và bền bỉ.

Chúc mừng sếp đã hoàn thành khóa huấn luyện cấp tốc này!
