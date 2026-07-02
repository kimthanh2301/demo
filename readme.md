

## Hình 1. Giải Thuật Hệ Thống Tổng Quát

```mermaid
flowchart TD
    S([BẮT ĐẦU])
    A["Khởi tạo phần cứng\nLCD · RTC · MP3 · LED · Nút · Quạt"]
    B["Kết nối WiFi và MQTT — ERa.begin"]
    C["Đọc DHT11 lần đầu\nĐăng ký interval 1s và 5s\nPhát âm thanh khởi động"]
    D["ERa.run — duy trì WiFi, MQTT, kích hoạt interval"]
    E["runAlarm — điều phối báo thức đang active"]
    F["checkButton — đọc nút xác nhận"]
    G["handleSerial — lệnh LICH · HIS · TIME"]
    H{"Có lệnh dừng?"}
    END([KẾT THÚC])

    S --> A --> B --> C --> D --> E --> F --> G --> H
    H -- Không --> D
    H -- Có --> END
```

### Mô tả nguyên lý hoạt động

Hệ thống bắt đầu bằng `setup()`: khởi tạo phần cứng, kết nối WiFi/MQTT, đọc DHT11 lần đầu, đăng ký hai interval (1 giây và 5 giây), phát âm thanh khởi động. Sau đó chuyển sang `loop()` chạy tuần tự 4 tác vụ: `ERa.run()` duy trì kết nối và kích hoạt interval; `runAlarm()` xử lý báo thức; `checkButton()` đọc nút xác nhận; `handleSerial()` xử lý lệnh debug. Khối điều kiện "Có lệnh dừng?" trên thực tế luôn trả về **Không**, tạo thành vòng quét vô hạn đặc trưng của hệ thống nhúng.

---

## Hình 2. Giải Thuật Kiểm Tra Giờ và Báo Uống Thuốc

```mermaid
flowchart TD
    S([BẮT ĐẦU — mỗi 1 giây])
    A{"Sang ngày mới?"}
    A1["Reset trạng thái 14 liều\nXóa xác nhận ngày hôm nay lên ERa"]
    B{"Giờ hiện tại\nkhớp lịch thuốc?"}
    B1([KẾT THÚC])
    C["Bật báo thức\nLED ngày sáng · Gửi V26 = 1 lên ERa"]
    D{"Nút xác nhận\nđã nhấn?"}
    E["Ghi đã uống · Tắt LED\nCập nhật ERa · V26 = 2"]
    F{"Quá 2 phút\nchưa nhấn?"}
    G["Ghi bỏ lỡ · Tắt LED\nCập nhật ERa · V26 = 3"]
    H["Phát MP3 nhắc uống thuốc\nmỗi 3 giây"]
    END([KẾT THÚC])

    S --> A
    A -- Có --> A1 --> B
    A -- Không --> B
    B -- Không --> B1
    B -- Có --> C --> D
    D -- Có --> E --> END
    D -- Không --> F
    F -- Có --> G --> END
    F -- Không --> H --> D
```

### Mô tả nguyên lý hoạt động

Mỗi giây, hệ thống kiểm tra nếu sang ngày mới thì reset toàn bộ 14 trạng thái liều và đồng bộ lên ERa. Tiếp theo so sánh phút hiện tại với từng slot sáng/chiều — nếu **không khớp** thì kết thúc ngay. Khi **khớp**, bật báo thức: LED ngày sáng, gửi V26=1. Hệ thống vào vòng chờ: cứ 3 giây phát MP3 nhắc một lần. Nếu người dùng **nhấn nút** trước 2 phút → ghi đã uống, V26=2, kết thúc. Nếu **quá 2 phút** không nhấn → ghi bỏ lỡ, V26=3, kết thúc.

---

## Hình 3. Giải Thuật Cảm Biến DHT11 và Điều Khiển Quạt

```mermaid
flowchart TD
    S([BẮT ĐẦU — mỗi 5 giây])
    A["Đọc nhiệt độ và độ ẩm từ DHT11"]
    B{"Đọc thất bại?"}
    C["Lưu giá trị · Gửi V0 V1 lên ERa · Cập nhật LCD"]
    D{"Đang chế độ tay?"}
    E{"Đã đủ 5 phút?"}
    E2["Hủy chế độ tay · Đồng bộ V25 = 2 lên App"]
    F{"Nhiệt độ trên 32°C\nhoặc Độ ẩm trên 70%?"}
    F1["Bật quạt — FAN HIGH"]
    F2["Tắt quạt — FAN LOW"]
    END([KẾT THÚC])

    S --> A --> B
    B -- Có --> END
    B -- Không --> C --> D
    D -- Có --> E
    E -- Chưa --> END
    E -- Đủ --> E2 --> F
    D -- Không --> F
    F -- Có --> F1 --> END
    F -- Không --> F2 --> END
```

### Mô tả nguyên lý hoạt động

Mỗi 5 giây, `dhtEvent` đọc DHT11. Nếu **đọc thất bại** (NaN) thì bỏ qua chu kỳ, không ghi đè giá trị hợp lệ cuối. Khi thành công, lưu nhiệt độ/độ ẩm, gửi lên ERa (V0, V1) và cập nhật LCD. Tiếp theo `controlFan` kiểm tra **chế độ tay**: nếu đang chế độ tay mà chưa đủ 5 phút thì giữ nguyên và thoát; nếu đã đủ 5 phút thì hủy chế độ tay và đồng bộ V25=2 lên app. Ở **chế độ tự động**: nhiệt độ vượt 32°C hoặc độ ẩm vượt 70% thì **bật quạt**; ngược lại thì **tắt quạt**.

---

