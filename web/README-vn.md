# Dashboard web MQTT Deye

> English: [README.md](README.md)

Trang tĩnh subscribe **`iriv/ivt/#`**. Trình duyệt dùng **MQTT qua WebSockets** — không dùng cổng TCP 1883.

## Tương thích

UI chỉ hiển thị dữ liệu khi có thiết bị **publish MQTT** dưới `iriv/ivt/...` với payload `{"value": <number>}`:

| Publisher | Hỗ trợ |
|-----------|--------|
| Cytron **IRIV IOC MQTT Gateway** | Có |
| **ESP32** (hoặc master khác) publish cùng topic `iriv/ivt/#` | Có |
| ESPHome **chỉ API** (không publish MQTT) | **Không** — dùng entity Home Assistant |

Mosquitto Docker với WebSocket **9001** nằm trong [`../iriv-ioc-mqtt-gateway/`](../iriv-ioc-mqtt-gateway/) (`docker compose up -d`).

## Chạy local

Trong thư mục `web/`:

```powershell
python -m http.server 8080
```

Mở `http://127.0.0.1:8080`. Biểu tượng bánh răng → URL WebSocket broker (mặc định `ws://<this-host>:9001`) → Connect.

![Giao diện Simple MQTT viewer](../docs/images/simple-mqtt-viewer-web.jpg)

Layout (góc trên bên phải, được nhớ trong trình duyệt):

- **Simplify** — tổng quan theo tab. PV1 / PV2 kéo full chiều ngang.
- **Full** — hàng KPI, sơ đồ dòng Deye, năng lượng hôm nay, bảng live kèm tuổi dữ liệu. Pin **âm = sạc**, **dương = xả**. Grid CT giống HA/Deye Cloud: **dương = Import** (mua lưới), **âm = Export**. Dòng tải chỉ lấy từ MQTT (không ước lượng từ U×I).
- **Minimal** — sơ đồ SCADA một dòng: nền đen, thanh feeder cyan/đỏ, nhãn monospace, dải MW live.

Host thư mục bằng nginx, Caddy hoặc GitHub Pages tương tự. Không cần backend.

## Bật WebSockets trên Mosquitto

Ưu tiên stack Docker trong `iriv-ioc-mqtt-gateway/` (cổng 1883 + 9001). Trên Mosquitto cài tay, thêm listener thứ hai:

```conf
# /etc/mosquitto/conf.d/websockets.conf
listener 9001
protocol websockets
allow_anonymous true
```

Nếu broker đã dùng file mật khẩu, giữ cấu hình đó và bỏ `allow_anonymous`. Sau đó:

```bash
sudo systemctl restart mosquitto
```

Chỉ mở `9001/tcp` trên LAN (hoặc Tailscale), không mở ra internet công cộng.

Nginx reverse proxy tùy chọn (TLS + same origin):

```nginx
location / {
    root /var/www/deye-dashboard;
    index index.html;
}

location /mqtt {
    proxy_pass http://127.0.0.1:9001/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
}
```

Khi đó URL trang là `wss://your-host/mqtt` (đặt trong hộp thoại bánh răng).

## Truy cập từ xa

**Không** port-forward 1883/9001 ra internet nếu chưa có TLS và auth.

An toàn hơn:

1. **Tailscale / WireGuard** — mở dashboard và `ws://100.x.x.x:9001` trên tailnet.
2. **HTTPS reverse proxy** như trên, kèm basic auth hoặc VPN phía trước.

GitHub Pages có thể serve thư mục này, nhưng **broker** vẫn phải reachable từ điện thoại/PC (Pages không tunnel MQTT).

## Client MQTT generic

Nếu chỉ cần xem raw topic, bỏ qua UI này:

| Client | Ghi chú |
|--------|---------|
| [HiveMQ Web Client](https://www.hivemq.com/demos/websocket-client/) | Trình duyệt; cần listener WebSocket LAN hoặc public |
| [MQTTX](https://mqttx.app/) | Desktop + web; TCP hoặc WebSocket |
| [MQTT Explorer](https://mqtt-explorer.com/) | Desktop; tiện debug `iriv/ivt/#` |

Subscribe `iriv/ivt/#`. Payload là `{"value": <number>}`.
