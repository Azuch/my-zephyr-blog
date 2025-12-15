---
layout: post
title: "Biên Niên Sử HTTP Client: Cuộc 'Đại Phẫu' Từ Gã 'Đần' Blocking Sang 'Cao Thủ' Bất Đồng Bộ"
---

Tài liệu này ghi lại chi tiết cuộc hội thoại và quá trình hướng dẫn tái cấu trúc một HTTP client đơn giản trong Zephyr RTOS, chuyển đổi từ một kiến trúc blocking tuần tự sang một kiến trúc non-blocking, bất đồng bộ dựa trên máy trạng thái.

## Hồi 1: Phán Xét "Bản Án Tử Hình" cho Code Cũ

Mọi chuyện bắt đầu khi sếp yêu cầu tôi xem xét file `http_get.c` và đánh giá hiệu quả của nó. Sau khi xem xét, tôi đã trình bày một bản "luận tội" chi tiết.

`[MEME: Quan toà đang đập búa phán xét]`

### Phân Tích Tội Trạng

Đây là bản phân tích của tôi về cách hoạt động của `http_get.c` và những "tội danh" của nó.

#### Tội danh #1: "Blocking Toàn Tập" - Lười biếng và Vô trách nhiệm
> **Lời luận tội:** "Thưa sếp, thằng `http_get.c` này là một kẻ lười biếng điển hình. Toàn bộ quá trình từ `nslookup` đến khi `http_get` xong xuôi đều là các hàm blocking. Nghĩa là cái thread `wifi_thread` sẽ bị treo cứng, không làm được gì khác trong suốt quá trình này. Nó giống như một nhân viên làm một việc rồi ngồi chơi xơi nước, mặc kệ các việc khác đang cháy nhà. Trong một ứng dụng thực tế, đây là một thảm họa."

#### Tội danh #2: Logic "Ngây Thơ"
> **Lời luận tội:** "Cơ chế fallback từ IPv6 sang IPv4 quá đơn giản. Nó sẽ tuần tự thử HẾT tất cả các địa chỉ IPv6, đợi từng cái timeout một, rồi mới chuyển sang IPv4. Nếu mạng bị cấu hình lỗi, ứng dụng sẽ 'đứng hình' rất lâu. Thật là một logic ngây thơ không thể chấp nhận được."

#### Tội danh #3: Xử lý Dữ liệu "Lúa"
> **Lời luận tội:** "Hàm callback chỉ `printk` từng mẩu dữ liệu ra. Giả sử sếp cần xử lý một file JSON, sếp sẽ phải tự mình viết code để ráp các mảnh đó lại từ những dòng log rời rạc. Há, cái ông dev viết ra cái này cũng thật là 'đê tiện' quá chứ hả!"

### Nâng Tầm "Vũ Trụ": So Sánh Với Tiêu Chuẩn Công Nghiệp
Khi sếp hỏi về "tiêu chuẩn công nghiệp", cuộc thảo luận đã thực sự được nâng tầm. Một thư viện "chuẩn công nghiệp" phải được xây dựng trên triết lý về sự **BỀN BỈ (Robustness)**, **AN TOÀN (Security)** và **KHẢ NĂNG MỞ RỘNG (Scalability)**.

*   **Kiến trúc Bất đồng bộ (Asynchronous):** "Blocking" là từ cấm kỵ. Một hệ thống công nghiệp phải chạy theo kiến trúc hướng sự kiện (Event-Driven), thường được hiện thực hóa bằng **máy trạng thái (State Machine)**. Nó không bao giờ "đứng hình" chờ đợi.
*   **Chiến lược kết nối "Happy Eyeballs":** Thay vì chờ IPv6 timeout, các thư viện chuyên nghiệp sẽ bắn yêu cầu IPv4 song song chỉ sau một khoảnh khắc rất ngắn (ví dụ 250ms). Thằng nào kết nối xong trước thì thắng.
*   **Bảo mật HTTPS:** HTTP không khác gì đứng ngoài đường la to bí mật. HTTPS qua TLS là yêu cầu bắt buộc.

Chính từ những phân tích này, sếp đã quyết định: "Ok, let's fix it, step by step; but teach me, not do it for me". Và cuộc hành trình tái cấu trúc của chúng ta bắt đầu.

## Hồi 2: Dựng Lại "Cơ Đồ" Với Máy Trạng Thái

Cuộc "đại phẫu" bắt đầu. Chúng ta sẽ vứt bỏ "căn nhà cấp 4" `http_get.c` và xây một "biệt thự" `http_client` mới.

### Bước 1: Đặt Nền Móng - Định nghĩa State và Context
> **Đại ca:** "Chú em, việc đầu tiên khi xây nhà là phải có bản vẽ. Trong lập trình bất đồng bộ, bản vẽ chính là **máy trạng thái (State Machine)**. Chúng ta phải định nghĩa rõ các 'phòng' (trạng thái) và một cái 'hộp đen' (context) để chứa toàn bộ thông tin."

**Chi tiết kỹ thuật:**
*   **`enum http_client_state`**: Đây là trái tim của State Machine. Mỗi `enum` là một bước trong quy trình: `IDLE`, `DNS_START`, `CONNECTING`, `DONE`, `ERROR`... Việc định nghĩa rõ ràng giúp ta hình dung được toàn bộ luồng đi của chương trình.
*   **`struct http_client_context`**: Đây là "bộ não". Thay vì truyền cả chục tham số qua lại, ta gói mọi thứ vào một `struct`: trạng thái hiện tại, socket, hostname, URL... Mọi hàm sẽ chỉ cần truyền con trỏ tới cái `struct` này.

`[MEME: Kiến trúc sư đang vẽ trên một bản thiết kế lớn]`

### Bước 2: Tạo "Nhịp Tim" cho Máy Trạng Thái
> **Đại ca:** "Ok, có bản vẽ rồi. Giờ tạo 'nhịp tim' cho nó. Đó chính là hàm `http_client_run()`. Nó sẽ là một cái vòng lặp `switch-case` xấu xí nhưng hiệu quả. Mỗi nhịp tim, nó sẽ kiểm tra xem client đang ở 'phòng' nào và phải làm gì tiếp theo."

**Chi tiết kỹ thuật:**
*   **Cấu trúc `switch-case`**: Cách triển khai state machine kinh điển. Mỗi `case` là một `state`.
*   **Hàm `set_state()`**: Tạo một hàm phụ trợ chỉ để chuyển state và `LOG_INF` ra sự thay đổi. `LOG_INF("State transition: %d -> %d", ...)` câu lệnh trông đơn giản nhưng nó là 'đèn pin' soi sáng con đường của state machine, cực kỳ hữu ích khi debug.
*   **`http_client_init()`**: Hàm này là nút "reset". Phải đảm bảo mọi thứ sạch sẽ trước mỗi lần chạy. Dùng `memset(ctx, 0, ...)` là một thói quen tốt để tránh các giá trị rác từ lần chạy trước.

### Bước 2.5: Tích Hợp vào `main.c`
> **Đại ca:** "Cái máy trạng thái này không tự chạy được. Phải có một thằng 'ngu' nào đó liên tục gọi nó. Thằng đó chính là cái vòng lặp `while(1)` trong `wifi_thread`. Nó sẽ liên tục gọi `http_client_run()` và `k_sleep()` một chút để nhường CPU cho đứa khác."

`[MEME: 'It's not much, but it's honest work' cho vòng lặp while(1)]`

## Hồi 3: Những Cú Nhảy Bất Đồng Bộ Đầu Tiên

### Bước 3: DNS Bất Đồng Bộ - Mồi Câu và Con Cá
> **Đại ca:** "Giờ mới là phần hay. Thay vì gọi `zsock_getaddrinfo()` và ngồi chờ nó trả IP về, chúng ta sẽ dùng 'cao chiêu' hơn: `dns_get_addr_info()`. Thằng này nó như kiểu chú quăng một cái mồi câu (yêu cầu DNS) xuống hồ, rồi chú đi làm việc khác. Khi nào cá cắn câu (có kết quả DNS), nó sẽ tự động giật cái phao (gọi hàm callback `dns_resolve_cb`)."

**Chi tiết kỹ thuật:**
*   **`dns_get_addr_info()`**: Hàm này trả về ngay lập tức. Nó nhận vào một con trỏ hàm callback và một `user_data`.
*   **`user_data`**: Đây là "sợi dây" liên kết. Chúng ta truyền con trỏ tới `client_ctx` vào đây. Khi callback được gọi, hệ thống trả lại con trỏ này, giúp ta biết chính xác client nào vừa có kết quả DNS.
*   **State `DNS_PENDING`**: Sau khi "quăng câu", phải ngay lập tức chuyển sang trạng thái "chờ cá". Nếu không, ở lần lặp `run()` tiếp theo, nó sẽ lại "quăng câu" nữa, tạo ra một vòng lặp vô tận.

### Bước 4: Kết Nối Non-Blocking & Cái Bẫy Con Trỏ
> **Đại ca:** "Sau khi 'cá cắn câu', chú em không được nuốt chửng cả con cá và cái dây câu! Cái con trỏ `info` mà callback trả về là 'hàng đi mượn', nó có thể bị hệ thống thu hồi bất cứ lúc nào. Phải 'lóc thịt' con cá (dùng `memcpy` sao chép địa chỉ IP vào `sockaddr_storage` trong context của mình), rồi 'trả lại cần câu' (gọi `dns_free_addrinfo(info)`) ngay lập tức. Không làm thế là memory leak, chết chương trình như chơi!"

`[MEME: It's a trap!]`

> **Đại ca (tiếp tục):** "Có IP rồi thì kết nối. Lại một lần nữa, không có chờ đợi. Dùng `zsock_fcntl` để biến cái socket thành 'non-blocking'. Khi gọi `zsock_connect()`, nó sẽ trả về lỗi `EINPROGRESS`. Há, đây không phải lỗi, đây là một 'tính năng'! Nó báo là 'Tao đang kết nối trong nền đây, đi làm việc khác đi'. Để biết khi nào kết nối xong, ta dùng `zsock_poll()` với sự kiện `ZSOCK_POLLOUT`."

**Chi tiết kỹ thuật về `zsock_poll()`:**
`poll()` là một hàm cho phép theo dõi nhiều socket cùng lúc. Ta bảo nó: "Này, mày canh chừng cái socket này cho tao. Khi nào nó sẵn sàng để 'ghi' (tức là kết nối thành công), thì báo tao một tiếng". Ta cho nó một timeout rất ngắn, ví dụ 10ms. Nếu chưa xong thì thôi, lần `run()` sau ta hỏi tiếp. Đây là bản chất của "chờ đợi mà không blocking".

### Bước 5: Hái Quả - Gửi Request và Nhận Response
> **Đại ca:** "Đến đây thì gần xong rồi. Ta có thể 'chơi ăn gian' một chút. Ta sẽ dùng lại hàm `http_client_req` có sẵn của Zephyr. Nó là một hàm blocking, nhưng nó đã làm quá nhiều việc phức tạp (tạo header, gọi callback xử lý body...). Tự làm lại từ đầu rất mệt. Ta chỉ cần gọi nó trong một state của cái máy trạng thái non-blocking của ta là được. Đây là một sự đánh đổi chấp nhận được."

## Hồi 4: Dọn Dẹp "Tàn Tích" và Tổng Kết

Sau khi "biệt thự" `http_client` mới đã xây xong, "căn nhà cấp 4" `http_get.c` và `http_get.h` đã bị "san phẳng" để giữ cho dự án gọn gàng.

Chúng ta đã đi từ một đoạn code blocking "ngu ngơ", đến việc hiểu rõ nhược điểm của nó, và tự tay thiết kế một client mới dựa trên kiến trúc máy trạng thái bất đồng bộ.

Sản phẩm cuối cùng, `http_client`, tuy vẫn có thể cải tiến nữa, nhưng nó đã là một nền tảng vững chắc, tuân thủ các nguyên tắc thiết kế hiện đại. Chúc mừng sếp đã hoàn thành khóa huấn luyện "đày ải" này!

`[MEME: Một nhân vật giơ cúp chiến thắng hoặc gật đầu mãn nguyện]`
