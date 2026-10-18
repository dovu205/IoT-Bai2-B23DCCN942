# Bài thực hành số 2 — Thu thập, lưu trữ và tiền xử lý dữ liệu IoT

**Học phần:** IoT và Ứng dụng (INT14149)
**Sinh viên:** Đỗ Minh Vũ — MSSV B23DCCN942 — Lớp IOT-N05-01

## Kiến trúc pipeline

```
+-----------+   MQTT (JSON)   +----------------+   subscribe   +------------------+
| ESP32     |  topic:         | broker.hivemq  |-------------->| collector.py     |
| (Wokwi)   |  ptit/B23DCCN   |.com :1883      |  validate,    | - validate JSON  |
| DHT22 +   |  942/iot2/      +----------------+  dedup/mất    | - hàng đợi bền   |
| HC-SR04   |  esp32-01/                    gói (sequence)    |   (SQLite)       |
+-----------+  telemetry                                     | - đo độ trễ      |
                                                             v
                                                    +--------+---------+
                                                    | InfluxDB 2.x     |
                                                    | (chạy native     |
                                                    |  trên Windows,   |
                                                    |  không Docker)   |
                                                    | bucket: iot_raw  |
                                                    |         (30 ngày)|
                                                    +--------+---------+
                                                              |
                                                              | Flux query
                                 +-----------------------------+-----------------------------+
                                 |                                                           |
                          +------v-------+                                          +--------v--------+
                          | preprocessing/|                                          | dashboard/      |
                          | preprocess.py|                                          | dashboard.py    |
                          | - resample   |                                          | (Streamlit)     |
                          |   cửa sổ 30s |                                          | - realtime      |
                          | - nội suy    |                                          | - so sánh thô/  |
                          | - IQR outlier|                                          |   đã xử lý      |
                          | - rolling    |                                          | - thống kê      |
                          |   mean, delta|                                          | - độ trễ        |
                          | - chuẩn hóa  |                                          +-----------------+
                          +------+-------+
                                 | ghi
                                 v
                          bucket: iot_processed
```

**Phần cứng (mô phỏng Wokwi):** ESP32 + DHT22 (nhiệt độ, độ ẩm — chân D15) +
HC-SR04 (khoảng cách — TRIG D5, ECHO D18) + LED báo publish (D2).
Firmware đọc cảm biến, kiểm tra `isfinite`, đóng gói JSON 6 khóa
(`temperature`, `humidity`, `distance_cm`, `rssi`, `sequence`, `uptime_s`),
gửi mỗi 5 giây.

## Yêu cầu

- Python 3.10+ (khuyến nghị dùng venv riêng)
- InfluxDB 2.x chạy trên máy (không cần Docker)

## Hướng dẫn chạy

### Bước 0 — Cài InfluxDB 2.x native trên Windows

1. Tải `influxdb2-2.8.0-windows.zip` tại
   https://github.com/influxdata/influxdb/releases (tag v2.8.0), giải nén
   vào `C:\influxdb\`.
2. Chạy `C:\influxdb\influxd.exe` (giữ cửa sổ này mở suốt quá trình chạy).
3. Mở http://localhost:8086 → **Get Started**: username tùy ý,
   organization `ptit`, bucket `iot_raw` → copy **token** (chỉ hiện 1 lần).
4. Tạo thêm bucket `iot_processed`: **Data → Buckets → Create Bucket**.
5. Đặt retention cho cả 2 bucket: **Buckets → tên bucket → Settings →
   Retention 30 days**.

### Bước 1 — Cài thư viện Python

Mở PowerShell tại thư mục gốc của repo:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

### Bước 2 — Cấu hình

```powershell
copy .env.example .env
```

Mở file `.env`, dán token InfluxDB vào `INFLUXDB_TOKEN`
(giữ nguyên các giá trị còn lại nếu làm đúng Bước 0).

> ⚠️ Không commit file `.env` (đã có trong `.gitignore`) vì chứa token.

### Bước 3 — Chạy mô phỏng ESP32

- Mở project Wokwi: xem link trong `firmware/wokwi-project-2.txt`,
  hoặc import `firmware/esp32-blink-2.ino` + `firmware/diagram.json`
  + `firmware/libraries.txt` vào Wokwi mới.
- Nhấn **Start** — Serial Monitor phải hiện `Publish: OK` mỗi 5 giây.

### Bước 4 — Chạy collector (thu thập + ghi dữ liệu)

```powershell
python collector/collector.py
```

Collector subscribe topic MQTT, validate JSON, phát hiện trùng lặp/mất gói
qua `sequence`, ghi vào bucket `iot_raw` (measurement `sensor_raw`), đồng thời
ghi log độ trễ từng bản tin (`[HH:MM:SS] DA LUU | seq=... | tre UOC TINH=...`).
Nhấn `Ctrl+C` để dừng (hàng đợi SQLite đảm bảo không mất dữ liệu khi dừng).

### Bước 5 — Tiền xử lý

Để collector chạy 5–10 phút cho có dữ liệu, mở terminal mới:

```powershell
python preprocessing/preprocess.py
```

Các bước: đọc dữ liệu thô (Flux) → resample cửa sổ 30s → nội suy khoảng
trống ngắn (≤ 2 ô 5s) → đánh dấu outlier bằng IQR → đặc trưng rolling mean
(3 cửa sổ), delta → chuẩn hóa Min-Max. Kết quả ghi vào bucket
`iot_processed` (measurement `sensor_processed`).

### Bước 6 — Dashboard

```powershell
streamlit run dashboard/dashboard.py
```

Mở địa chỉ hiện ra (thường http://localhost:8501): KPI mới nhất, biểu đồ
so sánh dữ liệu thô/đã xử lý, thống kê tiền xử lý, panel độ trễ chu trình.
Tự làm mới mỗi 5 giây.

## Đo độ trễ

Collector tự đo 2 thành phần cho mỗi chu trình lấy mẫu (dùng cùng một đồng
hồ trên máy tính):
- `tre UOC TINH` (request → nhận dữ liệu, gồm cả 2 chiều MQTT qua broker public)
- `ghi DB` (thời gian ghi InfluxDB local)

Số liệu tổng hợp xem trên dashboard (trung vị, P95) hoặc tải file CSV
`iot_timing.csv` từ dashboard.

## Cấu trúc thư mục

```
firmware/            Code ESP32 + sơ đồ Wokwi
collector/           Script thu thập MQTT -> InfluxDB
preprocessing/       Script tiền xử lý
dashboard/           Dashboard Streamlit
.env.example         Mẫu cấu hình (không chứa secret)
requirements.txt     Thư viện Python
```

## Lưu ý khi nộp/chấm

- Repo này không chứa secret nào (token chỉ nằm trong `.env` local).
- Để tái hiện kết quả: chạy đủ Bước 0–6, thay đổi thanh trượt DHT22 trên
  Wokwi để tạo biến thiên nhiệt độ, chụp ảnh dashboard + Data Explorer.
