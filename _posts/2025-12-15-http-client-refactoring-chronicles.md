---
layout: post
title: "Biên Niên Sử HTTP Client: Cuộc 'Đại Phẫu' Từ Gã 'Đần' Blocking Sang 'Cao Thủ' Bất Đồng Bộ"
---

Tài liệu này ghi lại chi tiết cuộc hội thoại và quá trình hướng dẫn tái cấu trúc một HTTP client đơn giản trong Zephyr RTOS, chuyển đổi từ một kiến trúc blocking tuần tự sang một kiến trúc non-blocking, bất đồng bộ dựa trên máy trạng thái.

## Hồi 1: Phán Xét "Bản Án Tử Hình" cho Code Cũ

Mọi chuyện bắt đầu khi sếp yêu cầu tôi xem xét file `http_get.c` và đánh giá hiệu quả của nó. Sau khi xem xét, tôi đã trình lên một bản "luận tội" chi tiết.

`[MEME: Quan toà đang đập búa phán xét]`

### 1.1 Phân Tích "Tội Trạng"

#### Tội danh #1: "Blocking Toàn Tập" - Lười biếng và Vô trách nhiệm
> **Lời luận tội:** "Thưa sếp, thằng `http_get.c` này là một kẻ lười biếng điển hình. Toàn bộ quá trình từ `nslookup` đến khi `http_get` xong xuôi đều là các hàm blocking. Nghĩa là cái thread `wifi_thread` sẽ bị treo cứng, không làm được gì khác trong suốt quá trình này. Nó giống như một nhân viên làm một việc rồi ngồi chơi xơi nước, mặc kệ các việc khác đang cháy nhà. Trong một ứng dụng thực tế, đây là một thảm họa!"

`[MEME: Con lười trong Zootopia đang làm việc]`

**Chi tiết kỹ thuật:** Mỗi lời gọi hàm như `zsock_getaddrinfo()`, `zsock_connect()`, `http_client_req()` đều khiến thread bị dừng lại hoàn toàn, chờ đợi phản hồi từ mạng. Trong thời gian chờ đó (có thể là vài giây), thread không thể thực hiện bất kỳ công việc nào khác, như đọc một cảm biến khác hay phản hồi một nút nhấn. Điều này làm giảm nghiêm trọng khả năng phản hồi của hệ thống.

#### Tội danh #2: Logic "Ngây Thơ Như Nai Tơ"
> **Lời luận tội:** "Cơ chế fallback từ IPv6 sang IPv4 quá đơn giản. Nó sẽ tuần tự thử HẾT tất cả các địa chỉ IPv6, đợi từng cái timeout một, rồi mới chuyển sang IPv4. Nếu mạng bị cấu hình lỗi, ứng dụng sẽ 'đứng hình' rất lâu. Thật là một logic ngây thơ không thể chấp nhận được."

#### Tội danh #3: Xử lý Dữ liệu "Lúa"
> **Lời luận tội:** "Hàm callback chỉ `printk` từng mẩu dữ liệu ra. Giả sử sếp cần xử lý một file JSON, sếp sẽ phải tự mình viết code để ráp các mảnh đó lại từ những dòng log rời rạc. Há, cái ông dev viết ra cái này cũng thật là 'đê tiện' quá chứ hả!"

### 1.2 Nâng Tầm "Vũ Trụ": So Sánh Với Tiêu Chuẩn Công Nghiệp
Một thư viện "chuẩn công nghiệp" phải được xây dựng trên triết lý về sự **BỀN BỈ (Robustness)**, **AN TOÀN (Security)** và **KHẢ NĂNG MỞ RỘNG (Scalability)**.
*   **Kiến trúc Bất đồng bộ (Asynchronous):** Phải là Event-Driven, dùng State Machine, không bao giờ "đứng hình".
*   **Chiến lược kết nối "Happy Eyeballs":** Phải chạy đua song song giữa IPv4 và IPv6, thằng nào nhanh hơn thì thắng.
*   **Bảo mật HTTPS:** HTTP không khác gì đứng ngoài đường la to bí mật. HTTPS là bắt buộc.

Chính từ những phân tích này, sếp đã quyết định: "Ok, let's fix it, step by step; but teach me, not do it for me". Và cuộc hành trình tái cấu trúc của chúng ta bắt đầu.

---
## Hồi 2: Dựng Lại "Cơ Đồ" Với Máy Trạng Thái

Cuộc "đại phẫu" bắt đầu. Chúng ta sẽ vứt bỏ "căn nhà cấp 4" `http_get.c` và xây một "biệt thự" `http_client` mới.

### Bước 1: Đặt Nền Móng - Định nghĩa State và Context
> **Đại ca:** "Chú em, việc đầu tiên khi xây nhà là phải có bản vẽ. Trong lập trình bất đồng bộ, bản vẽ chính là **máy trạng thái (State Machine)**. Chúng ta phải định nghĩa rõ các 'phòng' (trạng thái) và một cái 'hộp đen' (context) để chứa toàn bộ thông tin."

**Chi tiết kỹ thuật:**
*   **`enum http_client_state`**: Đây là trái tim của State Machine. Mỗi `enum` là một bước trong quy trình: `IDLE`, `DNS_START`, `CONNECTING`, `DONE`, `ERROR`... Việc định nghĩa rõ ràng giúp ta hình dung được toàn bộ luồng đi của chương trình.
*   **`struct http_client_context`**: Đây là "bộ não". Thay vì truyền cả chục tham số qua lại, ta gói mọi thứ vào một `struct`: trạng thái hiện tại, socket, hostname, URL... Mọi hàm sẽ chỉ cần truyền con trỏ tới cái `struct` này.

**Code đầy đủ tại Bước 1:**
*File `src/http_client.h`:*
```c
#ifndef HTTP_CLIENT_H_
#define HTTP_CLIENT_H_

#include <zephyr/net/socket.h>

/* Include Guard: Ngăn chặn việc file header này bị include nhiều lần */
enum http_client_state {
    HTTP_CLIENT_STATE_IDLE,
    HTTP_CLIENT_STATE_DNS_START,
    HTTP_CLIENT_STATE_DNS_PENDING,
    HTTP_CLIENT_STATE_CONNECT_START,
    HTTP_CLIENT_STATE_CONNECTING,
    HTTP_CLIENT_STATE_REQUEST_SEND,
    HTTP_CLIENT_STATE_RESPONSE_RECV,
    HTTP_CLIENT_STATE_DONE,
    HTTP_CLIENT_STATE_ERROR,
};

struct http_client_context {
    enum http_client_state state;
    int sock;
    struct zsock_addrinfo *addr_res; // Sẽ sớm thấy đây là một rủi ro
    struct zsock_addrinfo *current_addr;
    char *hostname;
    char *url;
    uint16_t port;
};

void http_client_init(struct http_client_context *ctx, char *hostname, char *url, uint16_t port);
int http_client_run(struct http_client_context *ctx);

#endif /* HTTP_CLIENT_H_ */
```
*File `src/http_client.c` (ban đầu):*
```c
#include "http_client.h"
#include <zephyr/logging/log.h>
LOG_MODULE_REGISTER(http_client, LOG_LEVEL_INF);
void http_client_init(struct http_client_context *ctx, char *hostname, char *url, uint16_t port) { /*TODO*/ }
int http_client_run(struct http_client_context *ctx) { return 0; }
```

### Bước 2 & 2.5: Tạo "Nhịp Tim" và Tích Hợp
> **Đại ca:** "Ok, có bản vẽ rồi. Giờ tạo 'nhịp tim' cho nó. Đó chính là hàm `http_client_run()`, một cái `switch-case` xấu xí nhưng hiệu quả. Và cái máy này không tự chạy được, phải có một thằng 'ngu' nào đó liên tục gọi nó trong `while(1)`. Thằng đó chính là `wifi_thread`."

**Chi tiết kỹ thuật:**
Hàm `http_client_init()` sẽ dùng `memset` để reset context. Hàm `http_client_run()` sẽ chứa `switch(ctx->state)`. Vòng lặp `while(1)` trong `main.c` sẽ liên tục gọi `http_client_run()` và `k_sleep()` để nhường CPU.

`[MEME: 'It's not much, but it's honest work' cho vòng lặp while(1)]`

**Code `main.c` đã tích hợp:**
```c
#include <zephyr/kernel.h>
#include <zephyr/logging/log.h>
#include "wifi.h"
#include "http_client.h"

LOG_MODULE_REGISTER(main_module, LOG_LEVEL_INF);

#define WIFI_STACK_SIZE 4096

void wifi_thread_entry(void *a, void *b, void *c) {
	LOG_INF("Wifi thread is created");
	wifi_init();

	struct http_client_context client_ctx;
	http_client_init(&client_ctx, "iot.beyondlogic.org", "/LoremIpsum.txt", 80);

	while (1) {
		int ret = http_client_run(&client_ctx);
		if (ret != 0) {
			LOG_INF("HTTP client process finished with status: %d", ret);
			break;
		}
		k_sleep(K_MSEC(100));
	}
}

K_THREAD_DEFINE(wifi_thread, WIFI_STACK_SIZE, wifi_thread_entry, NULL, NULL, NULL, 7,0,0);
int main(void) { LOG_INF("Main is created"); return 0; }
```

---
## Hồi 3: Những Cú Nhảy Bất Đồng Bộ "Thót Tim"

### Bước 3: DNS Bất Đồng Bộ - "Quăng Câu Chờ Cá"
> **Đại ca:** "Giờ mới là phần hay. Thay vì gọi `zsock_getaddrinfo()` và ngồi chờ, ta sẽ dùng 'cao chiêu' hơn: `dns_get_addr_info()`. Thằng này nó như kiểu chú quăng một cái mồi câu (yêu cầu DNS) xuống hồ, rồi chú đi làm việc khác. Khi nào cá cắn câu (có kết quả), nó sẽ tự động giật cái phao (gọi hàm callback `dns_resolve_cb`)."

**Chi tiết kỹ thuật:** `dns_get_addr_info()` nhận một con trỏ hàm callback và một `user_data`. Chúng ta truyền con trỏ tới `client_ctx` vào `user_data` để khi callback được gọi, ta biết client nào đã có kết quả. Sau khi gọi, phải ngay lập tức chuyển sang state `DNS_PENDING` để chờ.

### Bước 4: Kết Nối Non-Blocking & Cái Bẫy Con Trỏ
> **Đại ca:** "Chú em cẩn thận! Sau khi 'cá cắn câu', chú em không được nuốt chửng cả con cá và cái dây câu! Cái con trỏ `info` mà callback trả về là 'hàng đi mượn', nó có thể bị hệ thống thu hồi bất cứ lúc nào. Phải 'lóc thịt' con cá (dùng `memcpy` sao chép địa chỉ IP vào `sockaddr_storage` trong context của mình), rồi 'trả lại cần câu' (gọi `dns_free_addrinfo(info)`) ngay lập tức. Không làm thế là memory leak, chết chương trình như chơi!"

`[MEME: It's a trap!]`

> **Đại ca (tiếp tục):** "Có IP rồi thì kết nối. Lại một lần nữa, không có chờ đợi! Dùng `zsock_fcntl` để biến cái socket thành 'non-blocking'. Khi gọi `zsock_connect()`, nó sẽ trả về lỗi `EINPROGRESS`. Há, đây không phải lỗi, đây là một 'tính năng'! Nó báo là 'Tao đang kết nối trong nền đây, đi làm việc khác đi'. Để biết khi nào kết nối xong, ta dùng `zsock_poll()` với sự kiện `ZSOCK_POLLOUT`."

**Chi tiết kỹ thuật về `zsock_poll()`:**
`poll()` là một hàm cho phép theo dõi nhiều socket cùng lúc. Ta bảo nó: "Này, mày canh chừng cái socket này cho tao. Khi nào nó sẵn sàng để 'ghi' (tức là kết nối thành công), thì báo tao một tiếng". Ta cho nó một timeout rất ngắn, ví dụ 10ms. Nếu chưa xong thì thôi, lần `run()` sau ta hỏi tiếp.

`[MEME: 'It's not a bug, it's a feature']`

**Code đầy đủ tại Bước 4:**
*File `src/http_client.h` đã được sửa:*
```c
#ifndef HTTP_CLIENT_H_
#define HTTP_CLIENT_H_
#include <zephyr/net/socket.h>

// enum giữ nguyên
struct http_client_context {
    // ...
    // Loại bỏ con trỏ không an toàn
    struct sockaddr_storage current_addr; // Thay thế bằng vùng nhớ an toàn
    socklen_t current_addr_len;
    // ...
};
// ...
#endif
```
*File `src/http_client.c` với logic `connect` non-blocking:*
```c
// ...
static void dns_resolve_cb(enum dns_resolve_status status, struct dns_addrinfo *info, void *user_data)
{
    // ... (kiểm tra lỗi) ...
    // SAO CHÉP địa chỉ một cách an toàn.
    ctx->current_addr_len = info->ai_addrlen;
    memcpy(&ctx->current_addr, info->ai_addr, info->ai_addrlen);
    // GIẢI PHÓNG bộ nhớ ngay sau khi sao chép.
    dns_free_addrinfo(info);
    set_state(ctx, HTTP_CLIENT_STATE_CONNECT_START);
}

int http_client_run(struct http_client_context *ctx)
{
    // ...
    case HTTP_CLIENT_STATE_CONNECT_START:
        // Set port, tạo socket, đặt non-blocking
        zsock_fcntl(ctx->sock, F_SETFL, O_NONBLOCK);
        ret = zsock_connect(ctx->sock, (struct sockaddr *)&ctx->current_addr, ctx->current_addr_len);
        if (ret < 0 && errno != EINPROGRESS) {
            set_state(ctx, HTTP_CLIENT_STATE_ERROR);
        } else {
            set_state(ctx, HTTP_CLIENT_STATE_CONNECTING);
        }
        break;

    case HTTP_CLIENT_STATE_CONNECTING:
        struct zsock_pollfd fds[1];
        fds[0].fd = ctx->sock;
        fds[0].events = ZSOCK_POLLOUT;
        ret = zsock_poll(fds, 1, 10);
        if (ret > 0 && (fds[0].revents & ZSOCK_POLLOUT)) {
            set_state(ctx, HTTP_CLIENT_STATE_REQUEST_SEND);
        }
        break;
    // ...
}
```

### Bước 5: Hái Quả - Chơi "Ăn Gian"
> **Đại ca:** "Đến đây thì gần xong rồi. Ta có thể 'chơi ăn gian' một chút. Ta sẽ dùng lại hàm `http_client_req` có sẵn của Zephyr. Nó là một hàm blocking, nhưng nó đã làm quá nhiều việc phức tạp. Tự làm lại từ đầu rất mệt. Ta chỉ cần gọi nó trong một state của cái máy trạng thái của ta là được. Đây là một sự đánh đổi chấp nhận được."

**Code đầy đủ tại Bước 5:**
*File `src/http_client.h` cuối cùng:*
```c
#ifndef HTTP_CLIENT_H_
#define HTTP_CLIENT_H_
// ... (thêm include http/client.h) ...
#define RECV_BUF_SIZE 2048
struct http_client_context {
    // ... (thêm http_request và recv_buf) ...
};
// ...
#endif
```
*File `src/http_client.c` với logic gửi request:*
```c
// ...
static void http_response_cb(struct http_response *rsp, enum http_final_call final_data, void *user_data)
{
    if (rsp->body_frag_len > 0) {
        printk("%.*s", rsp->body_frag_len, rsp->body_frag_start);
    }
    if (final_data == HTTP_DATA_FINAL) {
        (void)zsock_close(ctx->sock);
        ctx->sock = -1;
        set_state(ctx, HTTP_CLIENT_STATE_DONE);
    }
}
// ...
int http_client_run(...)
{
    //...
    case HTTP_CLIENT_STATE_REQUEST_SEND:
        // Chuẩn bị request
        ctx->req.method = HTTP_GET;
        ctx->req.url = ctx->url;
        ctx->req.host = ctx->hostname;
        // ... (các trường khác) ...
        ctx->req.response = http_response_cb;
        ctx->req.user_data = ctx;
        // Gọi hàm blocking của Zephyr
        ret = http_client_req(ctx->sock, &ctx->req, 5000, ctx);
        if (ret < 0) {
            set_state(ctx, HTTP_CLIENT_STATE_ERROR);
        } else {
            set_state(ctx, HTTP_CLIENT_STATE_RESPONSE_RECV);
        }
        break;
    // ...
}
```

---
## Hồi 4: Dọn Dẹp "Tàn Tích" và Tổng Kết
Sau khi "biệt thự" `http_client` mới đã xây xong, "căn nhà cấp 4" `http_get.c` và `http_get.h` đã bị "san phẳng".

Chúng ta đã đi từ một đoạn code blocking "ngu ngơ", đến việc hiểu rõ nhược điểm, và tự tay thiết kế một client mới dựa trên kiến trúc máy trạng thái bất đồng bộ. Sản phẩm cuối cùng là một nền tảng vững chắc, tuân thủ các nguyên tắc thiết kế hiện đại. Chúc mừng sếp đã hoàn thành khóa huấn luyện "đày ải" này!

`[MEME: Một nhân vật giơ cúp chiến thắng hoặc gật đầu mãn nguyện]`
