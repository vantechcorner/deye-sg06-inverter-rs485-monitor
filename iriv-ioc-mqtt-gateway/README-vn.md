# IRIV IOC MQTT Gateway — Deye SG06

> English: [README.md](README.md)

Cytron **IRIV IOC MQTT Gateway** đọc biến tần Deye qua **Modbus RTU (RS485)** và publish giá trị đã scale lên MQTT dưới `iriv/ivt/...`.

**Trang sản phẩm (tải firmware):** [IRIV IO Controller MQTT — Cytron](https://www.cytron.io/p-iriv-io-controller-mqtt-ir4.0-industrial-i-o-controller-with-mqtt-ready)

**Một Modbus master trên mỗi bus RS485.** Không chạy gateway này chung cặp A/B với ESPHome / ESP32.

| Mục | Giá trị |
|-----|---------|
| Baud | **9600** 8N1 |
| Slave ID | **1** |
| MQTT base | `iriv/ivt` |
| Payload | JSON `{"value": <number>}` |
| Firmware | Nên dùng **≥ V1.2.6** cho bộ 27 job |

## File trong thư mục

| File | Vai trò |
|------|---------|
| [`iriv-ioc-config.json`](iriv-ioc-config.json) | Import trên firmware **≥ V1.2.6** (27 job, đủ PV2) |
| [`iriv-ioc-config-26.json`](iriv-ioc-config-26.json) | Chỉ firmware cũ (bỏ PV2 Current) |
| [`_gen_iriv_jobs.py`](_gen_iriv_jobs.py) | Sinh lại cả hai file JSON |
| [`iriv_deye_mqtt.yaml`](iriv_deye_mqtt.yaml) | Gói sensor MQTT cho Home Assistant |
| [`docker-compose.yml`](docker-compose.yml) + [`mosquitto.conf`](mosquitto.conf) | Mosquitto local (1883 + WebSocket 9001) |

Đăng nhập web thiết bị trong template JSON: user **`admin`**, mật khẩu **`12345678`** (lưu dạng `adminPassHash` / `adminPassSalt`). Đổi mật khẩu sau khi setup nếu thiết bị lộ ra ngoài LAN.

Sinh lại job (tránh sửa tay hàng chục poll slot nếu không cần):

```bash
python iriv-ioc-mqtt-gateway/_gen_iriv_jobs.py
```

---

## 1. Chạy Mosquitto bằng Docker

Trong thư mục này:

```bash
cd iriv-ioc-mqtt-gateway
docker compose up -d
```

| Cổng | Dùng cho |
|------|----------|
| **1883** | MQTT TCP — gateway IRIV, tích hợp MQTT của HA |
| **9001** | MQTT qua WebSockets — dashboard [`web/`](../web/) |

Xem log broker:

```bash
docker compose logs -f mosquitto
```

Trỏ MQTT host trên IRIV tới máy đang chạy Docker (hostname hoặc IP LAN). Template mặc định dùng `iriv-pi-control`; sửa trên thiết bị hoặc đổi trong JSON rồi import lại nếu cần.

Dừng:

```bash
docker compose down
```

**Không** mở 1883/9001 ra internet công cộng nếu chưa có TLS và xác thực.

---

## 2. Restore lần đầu trên IRIV IOC MQTT Gateway

### Cập nhật firmware trước

Trước khi import JSON của repo này, hãy flash firmware **mới nhất** của IRIV IOC MQTT Gateway từ Cytron:

1. Mở trang sản phẩm: [IRIV IO Controller MQTT](https://www.cytron.io/p-iriv-io-controller-mqtt-ir4.0-industrial-i-o-controller-with-mqtt-ready).
2. Tải gói firmware / release notes mới nhất từ trang đó (hoặc tutorial cập nhật firmware của Cytron).
3. Làm theo hướng dẫn **Factory Reset and Firmware Update** của Cytron cho IRIV IOC MQTT Gateway cho đến khi thiết bị báo đúng bản build mới.
4. Nên dùng firmware **≥ V1.2.6** để import đủ 27 job trong [`iriv-ioc-config.json`](iriv-ioc-config.json). Bản cũ hơn vẫn xóa hết job khi bật job thứ 27 — chỉ dùng [`iriv-ioc-config-26.json`](iriv-ioc-config-26.json) nếu tạm thời chưa nâng cấp được.

### Restore / import config qua USB

Với gateway mới, vừa factory-reset, hoặc vừa cập nhật firmware, cấu hình lần đầu qua cổng Ethernet gadget USB:

1. Nối **USB-C** từ IRIV IOC MQTT Gateway vào máy tính.
2. Đợi máy nhận link USB Ethernet / RNDIS (Windows có thể cài driver).
3. Mở trình duyệt tới **`http://10.0.0.1`**.
4. Đăng nhập bằng thông tin trong template JSON (**`admin` / `12345678`**) hoặc mật khẩu mặc định nhà máy nếu chưa import (tài liệu Cytron thường dùng `admin` / `admin` trước khi đổi mật khẩu).
5. Restore / import config poll:
   - Firmware **≥ V1.2.6** → [`iriv-ioc-config.json`](iriv-ioc-config.json)
   - Firmware cũ hơn → [`iriv-ioc-config-26.json`](iriv-ioc-config-26.json)
6. Kiểm tra MQTT: host = IP/hostname Mosquitto, cổng **1883**, base topic **`iriv/ivt`**, tắt auth trừ khi broker bắt buộc.
7. Nối **RS485 A/B** từ gateway tới cổng **datalogger / meter RS485** của Deye (không phải RJ45 CAN của BMS). Baud **9600**, slave **1**.
8. Sau khi bật Ethernet LAN, dùng IP LAN của thiết bị cho các lần cấu hình sau; USB `10.0.0.1` chủ yếu dùng lúc mang lên lần đầu.

Kiểm tra traffic (ví dụ bằng mosquitto client):

```bash
mosquitto_sub -h <broker-host> -t "iriv/ivt/#" -v
```

Bạn sẽ thấy các topic như `iriv/ivt/battery/soc`, `iriv/ivt/pv1/power`, `iriv/ivt/load/current`.

---

## 3. Thêm sensor vào Home Assistant

### Điều kiện trước khi bắt đầu

1. Home Assistant phải truy cập được cùng broker Mosquitto với gateway IRIV.
2. Phải **cài sẵn tích hợp MQTT** (MQTT integration / extension):
   - **Cài đặt → Thiết bị & dịch vụ → Thêm tích hợp → MQTT** (hoặc xác nhận MQTT đã có và đang kết nối).
   - Trỏ tới host/port broker (**1883**). Nếu chưa có tích hợp này, gói YAML bên dưới sẽ không tạo entity hoạt động.

### Cách A — thư mục packages (khuyên dùng)

1. Copy [`iriv_deye_mqtt.yaml`](iriv_deye_mqtt.yaml) vào config HA, ví dụ `config/packages/iriv_deye_mqtt.yaml`.
2. Bật packages trong `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

3. Khởi động lại Home Assistant (hoặc reload YAML / MQTT nếu phiên bản của bạn hỗ trợ).
4. Entity xuất hiện dưới thiết bị **Deye SG06 (IRIV)**; mỗi sensor đọc `value_json.value` từ `iriv/ivt/...`.

### Cách B — File Editor + `configuration.yaml`

Nếu không muốn upload file package riêng:

1. Cài add-on **File editor** (Cài đặt → Add-ons → File editor) nếu chưa có.
2. Mở File editor và sửa `/config/configuration.yaml` (trên một số layout supervised / HA OS hiển thị là `/homeassistant/configuration.yaml`).
3. Thực hiện một trong hai:
   - bật `packages` như Cách A và đặt `iriv_deye_mqtt.yaml` trong thư mục `packages/`, hoặc
   - dán / gộp khối `mqtt:` từ [`iriv_deye_mqtt.yaml`](iriv_deye_mqtt.yaml) thẳng vào `configuration.yaml`.
4. Kiểm tra cấu hình, rồi khởi động lại Home Assistant.

---

## Chu kỳ poll (template)

| Nhóm | Chu kỳ |
|------|--------|
| Điện áp / Dòng / Công suất | **1 s** |
| Nhiệt độ / SOC / tần số / trạng thái | **10 s** |
| Năng lượng `*_today` | **30 s** |

Scale đã kiểm chứng trên SG06: charge/discharge today **70 / 71**; dòng PV **0.1** A; dòng lưới **160** ×**0.01** A; dòng tải **179** ×0.01 A; tần số inverter **193** ×0.01 Hz. Dùng `dataType` **s16** (`3`) cho holding 16-bit.
