# Chuyến Phiêu Lưu vào Lãnh địa Thời gian: Tự Tay Chế Driver DHT11 và (Hy vọng) không làm nổ gì cả

Vậy là bạn đã có trong tay một con ESP32, một môi trường Zephyr RTOS mới coóng và một túi cảm biến DHT11 "ngon, bổ, rẻ" mua từ cửa hàng điện tử gần nhà. Bạn đã làm chủ được `printk`, đã nháy được vài con LED. Giờ là lúc cho một thử thách thực sự.

"Viết driver cho cái của nợ này thì có gì khó?" - bạn tự nhủ. Sau tất cả, nó chỉ là một con cảm biến thôi mà.

![Hold my beer](https://i.kym-cdn.com/photos/images/newsfeed/001/088/637/e11.png)

Và đó là lúc cuộc phiêu lưu của chúng ta bắt đầu.

## Giai đoạn 1: Lướt Datasheet - "It's some form of Elvish... I can't read it."

Việc đầu tiên của mọi kỹ sư là tìm datasheet. Và bạn nhận ra ngay vấn đề đầu tiên: con chip này không nói thứ tiếng I2C hay SPI thân thuộc. Nó nói một thứ ngôn ngữ riêng, một giao thức 1-dây tùy chỉnh, và "từ vựng" của nó được đo bằng micro-giây.

![It's some form of elvish](https://i.imgflip.com/1l6o4x.jpg)

Sau một hồi "thiền" với tài liệu, chúng ta rút ra được "vũ đạo" mà chúng ta phải thực hiện. Datasheet nói:

> Single-bus data format is used for communication and synchronization between MCU and DHT11 sensor. One communication process is about 4ms. Data consists of decimal and integral parts. A complete data transmission is 40bit...
>
> Data format: 8bit integral RH data + 8bit decimal RH data + 8bit integral T data + 8bit decimal T data + 8bit check sum.

Và đây là điệu nhảy timing cụ thể:
1.  **MCU chào hỏi:** Kéo đường dây xuống THẤP trong ít nhất 18ms, rồi kéo lên CAO trong 20-40µs.
2.  **Cảm biến trả lời:** Nó sẽ kéo xuống THẤP 80µs, rồi kéo lên CAO 80µs để nói "OK, tôi nghe rồi".
3.  **Cảm biến nói chuyện:** Nó sẽ gửi một chuỗi 40 bit. Mỗi bit bắt đầu bằng một xung THẤP 50µs. Sau đó, độ dài của xung CAO quyết định giá trị:
    *   Một xung CAO **ngắn** (26-28µs) là bit **0**.
    *   Một xung CAO **dài** (70µs) là bit **1**.

Nghe có vẻ... không quá tệ? Phải không?

## Giai đoạn 2: Viết Code - "This is where the fun begins."

Đây là lúc chúng ta biến "vũ đạo" trên giấy thành code C.

#### Màn Chào hỏi kiểu DHT11
Chúng ta cần một hàm để gửi tín hiệu "start". Dùng các API GPIO của Zephyr, chúng ta cấu hình pin thành OUTPUT, kéo xuống, `k_msleep(18)`, kéo lên, `k_busy_wait(30)`, rồi chuyển lại thành INPUT.

```c
static int dht11_send_start_signal(const struct device *dev)
{
    const struct dht11_config *config = dev->config;
    int ret;

    // Cấu hình chân GPIO thành chế độ OUTPUT để có thể "nói"
    ret = gpio_pin_configure_dt(&config->gpio, GPIO_OUTPUT);
    if (ret != 0) { return ret; }

    // Kéo chân xuống THẤP (LOW)
    ret = gpio_pin_set_dt(&config->gpio, 0);
    if (ret != 0) { return ret; }

    // Giữ trạng thái THẤP trong 18 mili-giây
    k_msleep(18);

    // Kéo chân lên CAO (HIGH) và đợi một khoảng thời gian rất ngắn
    ret = gpio_pin_set_dt(&config->gpio, 1);
    if (ret != 0) { return ret; }
    k_busy_wait(30);

    // Chuyển chân GPIO thành chế độ INPUT để "lắng nghe"
    ret = gpio_pin_configure_dt(&config->gpio, GPIO_INPUT);
    if (ret != 0) { return ret; }

    return 0;
}
```

#### Lắng nghe trong im lặng
Bây giờ là lúc chờ cảm biến trả lời. Chúng ta phải viết các vòng lặp `while` với `k_busy_wait(1)` và một biến timeout để liên tục "hỏi": "Cậu đã kéo đường dây xuống chưa? ... Giờ cậu kéo lên chưa? ... Ok, cậu kéo xuống lại chưa?". Cảm giác thật hồi hộp.

```c
static int dht11_check_response(const struct device *dev)
{
    const struct dht11_config *config = dev->config;
    uint32_t timeout_us = 100;

    // Chờ cảm biến kéo chân xuống THẤP
    while (gpio_pin_get_dt(&config->gpio) == 1) {
        if (--timeout_us == 0) { return -EIO; }
        k_busy_wait(1);
    }

    // Chờ cảm biến kéo chân lên CAO
    timeout_us = 100;
    while (gpio_pin_get_dt(&config->gpio) == 0) {
        if (--timeout_us == 0) { return -EIO; }
        k_busy_wait(1);
    }

    // Chờ cảm biến kết thúc tín hiệu phản hồi
    timeout_us = 100;
    while (gpio_pin_get_dt(&config->gpio) == 1) {
        if (--timeout_us == 0) { return -EIO; }
        k_busy_wait(1);
    }
    return 0;
}
```

#### Giải mã Ma trận
Đây là phần cao trào, nơi chúng ta đo độ dài xung và dùng các phép dịch bit (`<<=`) và OR bit (`|=`) để "đóng gói" 40 bit rời rạc vào 5 byte dữ liệu.

```c
static int dht11_read_data_bits(const struct device *dev, uint8_t data[5])
{
    const struct dht11_config *config = dev->config;
    memset(data, 0, 5);

    for (int i = 0; i < 40; i++) {
        uint32_t timeout_us = 100;
        // Chờ bắt đầu của xung CAO chứa dữ liệu
        while (gpio_pin_get_dt(&config->gpio) == 0) {
            if (--timeout_us == 0) { return -EIO; }
            k_busy_wait(1);
        }

        // Đo độ dài của xung CAO
        uint32_t pulse_len_us = 0;
        timeout_us = 100;
        while (gpio_pin_get_dt(&config->gpio) == 1) {
            if (--timeout_us == 0) { return -EIO; }
            k_busy_wait(1);
            pulse_len_us++;
        }

        // Dịch trái để tạo chỗ trống cho bit mới
        data[i / 8] <<= 1;
        // Nếu xung dài > 40µs, đó là bit 1.
        if (pulse_len_us > 40) {
            data[i / 8] |= 1; // Set bit cuối cùng thành 1.
        }
    }
    return 0;
}
```

#### Kiểm tra bài
Cuối cùng, chúng ta cộng 4 byte dữ liệu đầu tiên lại. Nếu nó bằng byte thứ 5, chúng ta đã thành công. Nếu không...

![This is fine](https://i.imgur.com/c4jt321.png)

...thì chúng ta bắt đầu lại từ đầu, và tự hỏi liệu `k_busy_wait` có thực sự chính xác không, hay dây cắm của mình bị lỏng.
```c
uint8_t checksum = raw_buffer[0] + raw_buffer[1] + raw_buffer[2] + raw_buffer[3];
if (checksum != raw_buffer[4]) {
    LOG_ERR("Invalid checksum!");
    return -EIO;
}
```

## Kết quả: "It's alive! ALIVE!"

Sau vài lần loay hoay với timeout và ngưỡng thời gian, cuối cùng chúng ta cũng đọc được một giá trị hợp lệ! Checksum khớp! Độ ẩm 60%, Nhiệt độ 25°C! Cảm giác thật tuyệt vời.

![It's alive](https://media.tenor.com/T0p8n5v5j9EAAAAC/alive-itsalive.gif)

Chúng ta đã tạo ra một driver hoạt động được từ con số không, chỉ bằng datasheet và các hàm API cơ bản của Zephyr. Nó có thể không hoàn hảo, không tối ưu, nhưng nó là *của chúng ta*.

**Nhưng... liệu nó có 'tốt' không?**

Hãy cùng xem các 'pháp sư' ở Zephyr đã làm điều này như thế nào trong bài viết tiếp theo, nơi chúng ta so sánh 'chiếc xe độ' của mình với một chiếc 'xe đua F1' của hãng.
