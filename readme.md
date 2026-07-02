

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
    B -- Không --> END
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

## Hình 4. Giải Thuật Đọc Nút Xác Nhận — checkButton

```mermaid
flowchart TD
    S([BẮT ĐẦU — mỗi loop])
    A["Đọc GPIO33 — INPUT_PULLUP"]
    B{"Trạng thái\nthay đổi?"}
    B1["Cập nhật debounce timer"]
    C{"Ổn định trên 50ms\nvà vừa xuống LOW?"}
    D{"Đang có\nbáo thức active?"}
    E["confirmDose\nGhi đã uống · Tắt LED\nV16+day lên ERa · V26 = 2"]
    END([KẾT THÚC])

    S --> A --> B
    B -- Có --> B1 --> END
    B -- Không --> C
    C -- Không --> END
    C -- Có --> D
    D -- Không --> END
    D -- Có --> E --> END
```

### Mô tả nguyên lý hoạt động

Mỗi vòng `loop()`, `checkButton` đọc GPIO33 (nút active LOW). Nếu trạng thái **vừa thay đổi** thì cập nhật debounce timer và thoát — chưa xử lý vì chưa ổn định. Nếu **không đổi**, kiểm tra đồng thời hai điều kiện: đã ổn định trên 50ms **và** vừa chuyển xuống LOW (tức là cạnh nhấn); nếu không thỏa thì thoát. Khi thỏa, kiểm tra tiếp có đang có **báo thức active** không; nếu có thì gọi `confirmDose()` để ghi đã uống, tắt LED và đồng bộ lên ERa.

---

## Hình 5. Giải Thuật Giao Tiếp ERa IoT

```mermaid
flowchart TD
    S([BẮT ĐẦU — ERa.begin])
    A["Kết nối WiFi\nKết nối MQTT broker mqtt1.eoh.io"]
    B{"Kết nối\nthành công?"}
    B1["ERA_CONNECTED\nĐồng bộ thời gian NTP UTC+7\nGhi vào RTC DS3231"]
    C["ERa.run — xử lý sự kiện mỗi loop"]
    D{"Nhận lệnh\ntừ App?"}
    DV{"Là pin\nV2\u2013V15?"}
    DV2{"Là pin\nV23 hoặc V24?"}
    E1["Cập nhật lịch thuốc 14 liều"]
    E2["Lưu ngưỡng nhiệt độ và độ ẩm"]
    E3["Điều khiển quạt thủ công"]
    F["Gửi dữ liệu lên App\nV0 V1 mỗi 5s · V16–V22 khi xác nhận · V26 khi báo thức"]
    G{"Mất kết nối?"}
    G1["ERA_DISCONNECTED\nTự động thử kết nối lại"]
    END([KẾT THÚC — chờ loop tiếp theo])

    S --> A --> B
    B -- Thất bại --> A
    B -- Thành công --> B1 --> C
    C --> D
    D -- Có --> DV
    DV -- Có --> E1 --> F
    DV -- Không --> DV2
    DV2 -- Có --> E2 --> F
    DV2 -- Không --> E3 --> F
    D -- Không --> F
    F --> G
    G -- Có --> G1 --> A
    G -- Không --> END
```

### Mô tả nguyên lý hoạt động

Khi khởi động, ERa kết nối WiFi rồi kết nối MQTT broker. Nếu **thất bại** thì thử lại liên tục cho đến khi thành công. Khi **kết nối thành công**, callback `ERA_CONNECTED` được gọi để đồng bộ thời gian từ máy chủ NTP (UTC+7) vào RTC DS3231 — đảm bảo lịch thuốc luôn chạy đúng giờ thực tế.

Trong mỗi vòng `loop()`, `ERa.run()` kiểm tra hàng đợi sự kiện. Nếu **App gửi lệnh xuống**, hệ thống phân loại theo virtual pin: V2–V15 cập nhật lịch uống thuốc 14 liều; V23/V24 lưu ngưỡng nhiệt độ/độ ẩm; V25 điều khiển quạt tức thì. Dù có hay không có lệnh từ App, hệ thống đều **gửi dữ liệu lên App**: nhiệt độ/độ ẩm mỗi 5 giây (V0, V1), trạng thái xác nhận uống thuốc (V16–V22) và thông báo báo thức (V26) khi có sự kiện. Nếu phát hiện **mất kết nối** thì `ERA_DISCONNECTED` được gọi và vòng lặp quay về bước kết nối lại từ đầu.

---
